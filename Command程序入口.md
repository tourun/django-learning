工作中主要基于django框架，进行项目的开发，我是主要做后台相关比较多一些，用的时间长了，总想了解其内部的运行原理，工作之余，对django的源码进行了学习，并在这里进行整理记录。

---
熟悉django的同学知道，django的后台进程通常通过下面这种方式运行：
> python manage.py app [options]

我们来看下它是如何实现的，在django结构层级中，有一个core目录，与命令相关的操作实现，都在这个目录中。我们假设当前的项目名为myproject，这里app表示要运行的app名称，具体为django项目中module/management/commands中自定义的进程文件名，文件中通常会定义Command类，并继承自django.core.management.BaseCommand，options表示一些可选的参数。以python manage app为例，看下它的运行原理。manage.py是在项目创建之后，自动生成的一个py文件，它的定义如下:
```python
if __name__ == "__main__":
    os.environ.setdefault("DJANGO_SETTINGS_MODULE", "myproject.settings")

    from django.core.management import execute_from_command_line

    execute_from_command_line(sys.argv)
```
execute_from_command_line 方法用于读取命令行参数，并执行相应的app程序代码:
```python
def execute_from_command_line(argv=None):
    """
    A simple method that runs a ManagementUtility.
    """
    utility = ManagementUtility(argv)
    utility.execute()
```
从这里可以看出，实际上app程序是通过 ManagementUtility.execute() 方法来执行的。execute方法定义在django.core.manage.__init__.py中：
```python
def execute(self):
    try:
        subcommand = self.argv[1]
    except IndexError:
        subcommand = 'help'  # Display help if no arguments were given.

    parser = CommandParser(None, usage="%(prog)s subcommand [options] [args]", add_help=False)
    parser.add_argument('--settings')
    parser.add_argument('--pythonpath')
    parser.add_argument('args', nargs='*')  # catch-all
    try:
        options, args = parser.parse_known_args(self.argv[2:])
        handle_default_options(options)
    except CommandError:
        pass  # Ignore any option errors at this point.

    no_settings_commands = [
        'help', 'version', '--help', '--version', '-h',
        'compilemessages', 'makemessages',
        'startapp', 'startproject',
    ]

    try:
        settings.INSTALLED_APPS
    except ImproperlyConfigured as exc:
        self.settings_exception = exc
        if subcommand in no_settings_commands:
            settings.configure()

    if settings.configured:
        if subcommand == 'runserver' and '--noreload' not in self.argv:
            try:
                autoreload.check_errors(django.setup)()
            except Exception:
                pass
        else:
            django.setup()

    self.autocomplete()

    if subcommand == 'help':
        if '--commands' in args:
            sys.stdout.write(self.main_help_text(commands_only=True) + '\n')
        elif len(options.args) < 1:
            sys.stdout.write(self.main_help_text() + '\n')
        else:
            self.fetch_command(options.args[0]).print_help(self.prog_name, options.args[0])
    elif subcommand == 'version' or self.argv[1:] == ['--version']:
        sys.stdout.write(django.get_version() + '\n')
    elif self.argv[1:] in (['--help'], ['-h']):
        sys.stdout.write(self.main_help_text() + '\n')
    else:
        self.fetch_command(subcommand).run_from_argv(self.argv)
```
我们来分解一下这段程序，subcommnad是python manage.py后的参数，即子程序名，argv[0]表示manage.py。这里如果没有指定，那么子程序默认为help。接着通过CommandParser来解析随后的参数，app子程序名之后的参数，这里我们默认没有其他参数。接着在try语句中执行 settings.INSTALLED_APPS，这句乍看上去很是不解，没有赋值，没有输出，注意settings是django.conf.\__init\__.py中定义的一个LazySettings对象，**LazySettings继承自LazyObject类，它重写了__getattr__和__setattr__方法，那么在调用settings.INSTALLED_APPS时，会通过其自定义的__getattr__方法实现**：
```python
settings = LazySettings()

# django.conf.__init__.py
class LazySettings(LazyObject):
    # other functions ...
    
    def _setup(self, name=None):
        settings_module = os.environ.get(ENVIRONMENT_VARIABLE)
        if not settings_module:
            desc = ("setting %s" % name) if name else "settings"
            raise ImproperlyConfigured("Requested %s, but settings are not configured. You must either define the environment variable %s or call settings.configure() before accessing settings."
                % (desc, ENVIRONMENT_VARIABLE))

        self._wrapped = Settings(settings_module)
        
    def __getattr__(self, name):
        if self._wrapped is empty:
            self._setup(name)
        return getattr(self._wrapped, name)
        
    # other functions ...
```
_setup方法从当前环境变量中获取ENVIRONMENT_VARIABLE（"DJANGO_SETTINGS_MODULE"），这个值在manage.py文件中已经定义，**manage.y文件是在我们在创建django项目时，通过startproject命令创建的**:
> os.environ.setdefault("DJANGO_SETTINGS_MODULE", "my_project.settings")

通过getter/setter方法，对settings对象的操作转到其私有成员self._wrapped对象的调用上，这里在第一次使用settings对象时，将其私有成员self._wrapped初始化为Settings类实例，其构造函数如下：
```python
# django.conf.__init__.py

class Settings(BaseSettings):
    def __init__(self, settings_module):
        # update this dict from global settings (but only for ALL_CAPS settings)
        for setting in dir(global_settings):
            if setting.isupper():
                setattr(self, setting, getattr(global_settings, setting))

        # store the settings module in case someone later cares
        self.SETTINGS_MODULE = settings_module

        mod = importlib.import_module(self.SETTINGS_MODULE)

        tuple_settings = (
            "ALLOWED_INCLUDE_ROOTS",
            "INSTALLED_APPS",
            "TEMPLATE_DIRS",
            "LOCALE_PATHS",
        )
        self._explicit_settings = set()
        for setting in dir(mod):
            if setting.isupper():
                setting_value = getattr(mod, setting)

                if (setting in tuple_settings and
                        isinstance(setting_value, six.string_types)):
                    raise ImproperlyConfigured("The %s setting must be a tuple. Please fix your settings." % setting)
                setattr(self, setting, setting_value)
                self._explicit_settings.add(setting)

        if not self.SECRET_KEY:
            raise ImproperlyConfigured("The SECRET_KEY setting must not be empty.")

        if ('django.contrib.auth.middleware.AuthenticationMiddleware' in self.MIDDLEWARE_CLASSES and
                'django.contrib.auth.middleware.SessionAuthenticationMiddleware' not in self.MIDDLEWARE_CLASSES):
            warnings.warn("Session verification will become mandatory in Django 1.10. Please add 'django.contrib.auth.middleware.SessionAuthenticationMiddleware' "
                "to your MIDDLEWARE_CLASSES setting when you are ready to opt-in after reading the upgrade considerations in the 1.8 release notes.",
                RemovedInDjango110Warning
            )

        if hasattr(time, 'tzset') and self.TIME_ZONE:
            zoneinfo_root = '/usr/share/zoneinfo'
            if (os.path.exists(zoneinfo_root) and not
                    os.path.exists(os.path.join(zoneinfo_root, *(self.TIME_ZONE.split('/'))))):
                raise ValueError("Incorrect timezone setting: %s" % self.TIME_ZONE)
            os.environ['TZ'] = self.TIME_ZONE
            time.tzset()
    # other functions ...
```
这里传递给settings_module的参数值为my_project.settings，构造函数会先通过global_settings来设置其属性，接着读取my_project.settings，设置其特定的属性，主要有ALLOWED_INCLUDE_ROOTS、INSTALLED_APPS、TEMPLATE_DIRS、LOCALE_PATHS这几个key，这几个key的解释如下：

- ALLOWED_INCLUDE_ROOTS， 默认值为 () (即空元组，在global_settings中)，它表示嵌入文件根路径的字符串——只有在某字符串存在于该元组的情况下，Django的 {% ssi %} 模板标签才会嵌入以其为前缀的文件。 这样做是出于安全考虑，从而使模板作者不能访问到他们不该访问的文件
- INSTALLED_APPS，默认同样为空元组，它表示项目中哪些 app 处于激活状态。元组中的字符串，除了django默认自带的命令之外，就是我们自己定义的app，也就是用python manage.py所启动的app了
- TEMPLATE_DIRS，默认同样为空元组，它表示模板文件的处处路径
- LOCALE_PATHS，默认同样为空元组，它表示Django将在这些路径中查找包含实际翻译文件的<locale_code>/LC_MESSAGES目录

**代码中使用了importlib.import_module这个方法，它支持程序动态引入以'.'分割的目录层次，比如importlib.import_module('django.core.management.commands.migrate')**，这里该方法引入了myproject.settings模块，加载settings配置文件中上述4个key的值。接着校验中间件和时区的配置信息，完成全局实例settings中self._wrapped属性的初始化，最终通过__getattr__方法，将加载到的INSTALLED_APPS信息返回。回到execute函数，这里的全局settings实例以及初始化完毕，我们的subcommand不是runserver（runserver的情况下来之后再分析），接着运行django.setup()方法：
```python
# django.__init__.py

def setup():
    from django.apps import apps
    from django.conf import settings
    from django.utils.log import configure_logging

    configure_logging(settings.LOGGING_CONFIG, settings.LOGGING)
    apps.populate(settings.INSTALLED_APPS)
```
这里setup函数配置日志信息，并且加载settings.INSTALLED_APPS中的自定义模块以及models模块，保存在django.apps中，这是一个全局的Apps类实例，用以注册或者说存储项目中的INSTALLED_APPS模块信息。我们来看下apps.populate方法：
```python
class Apps(object):
    # other functions ...
    
    def populate(self, installed_apps=None):
        if self.ready:
            return
        with self._lock:
            if self.ready:
                return

            if self.app_configs:
                raise RuntimeError("populate() isn't reentrant")

            for entry in installed_apps:
                if isinstance(entry, AppConfig):
                    app_config = entry
                else:
                    app_config = AppConfig.create(entry)
                if app_config.label in self.app_configs:
                    raise ImproperlyConfigured(
                        "Application labels aren't unique, "
                        "duplicates: %s" % app_config.label)

                self.app_configs[app_config.label] = app_config

            counts = Counter(
                app_config.name for app_config in self.app_configs.values())
            duplicates = [name for name, count in counts.most_common() if count > 1]
            if duplicates:
                raise ImproperlyConfigured("Application names aren't unique, duplicates: %s" % ", ".join(duplicates))

            self.apps_ready = True

            for app_config in self.app_configs.values():
                all_models = self.all_models[app_config.label]
                app_config.import_models(all_models)

            self.clear_cache()
            self.models_ready = True

            for app_config in self.get_app_configs():
                app_config.ready()

            self.ready = True
            
    # other functions ...
```
在for循环中，使用AppConfig.create(entry) 加载installed_apps里面的各模块，并保存在app_cofigs中，**注意create方法是AppConfig类的classmethod**，用以实现工厂模式，它根据installed_apps中的模块构造出 AppConfig(app_name, app_module) 这样的实例，其中app_name表示INSTALLED_APPS中指定的应用字符串，app_module表示根据app_name加载到的module。当加载的模块中有定义default_app_config时，那么会构造其表示的类对象，例如我们在django项目中会用到的用户认证鉴权模块，在INSTALLED_APPS中配置为'django.contrib.auth'，当在import_module此模块时，实际django.contrib.auth是一个python的package，在\__init\__.py文件中有定义了default_app_config = 'django.contrib.auth.apps.AuthConfig'，那么最终会构造apps.py中定义的AuthConfig类实例，这些default_app_config对应的类同样继承自AppConfig。在AppConfig实例的初始化方法中，会记录这些应用的标签、文件路径等信息，最终将这些实例会保存在其属性app_configs中。接着每个AppConfig实例会加载其指定模块的models，all_models定义为all_models = defaultdict(OrderedDict)，**defaultdict会创建表示一个类似dict的实例，在构造时可以指定字典中元素值的默认类型，这里用OrderedDict来指定其默认的类型，OrderedDict是dict的子类，它可以记录元素添加到字典中的顺序，保证元素有序，因此在获取all_models中的元素时，当key不存在时，会创建一个OrderedDict对象**，我们来看下models是如何加载的：
```python
for app_config in self.app_configs.values():
    all_models = self.all_models[app_config.label]
    app_config.import_models(all_models)

MODELS_MODULE_NAME = 'models'

def import_models(self, all_models):
    self.models = all_models

    if module_has_submodule(self.module, MODELS_MODULE_NAME):
        models_module_name = '%s.%s' % (self.name, MODELS_MODULE_NAME)
        self.models_module = import_module(models_module_name)
```
在module指定的目录或者package中，查找是否有定义models模块，并将其import进来。再回到execute方法中，如果python manage.py之后传递的是非help或者version这种帮助信息，那么会执行到语句：
> self.fetch_command(subcommand).run_from_argv(self.argv)

fetch_command方法内部先通过get_commands方法，从全局的apps对象中获取之前加载到的INSTALLED_APPS模块对应的management/commands包：
```python
# django.core.management.__init__.py

@lru_cache.lru_cache(maxsize=None)
def get_commands():
    commands = {name: 'django.core' for name in find_commands(upath(__path__[0]))}

    if not settings.configured:
        return commands

    for app_config in reversed(list(apps.get_app_configs())):
        path = os.path.join(app_config.path, 'management')
        commands.update({name: app_config.name for name in find_commands(path)})

    return commands

class ManagementUtility(object):
    # other functions ...
    def fetch_command(self, subcommand):
        commands = get_commands()
        try:
            app_name = commands[subcommand]
        except KeyError:
            # This might trigger ImproperlyConfigured (masked in get_commands)
            settings.INSTALLED_APPS
            sys.stderr.write("Unknown command: %r\nType '%s help' for usage.\n" %
                (subcommand, self.prog_name))
            sys.exit(1)
        if isinstance(app_name, BaseCommand):
            # If the command is already loaded, use it directly.
            klass = app_name
        else:
            klass = load_command_class(app_name, subcommand)
        return klass
        
    # other functions ...
```
**注意方法定义在django.core.management.\__init\__.py文件中，get_commands方法中的__path__[0]是其\__init\__.py的绝对路径**，这里通过find_commands首先将django.core.management.commands目录下的模块引入进来，像我们常用的一些基础模块（通过python manage.py进行调用）比如startpp、migrate、compilemessages、runserver、shell等都在此目录下。加载完这些基础模块之后，接着加载apps中的自定义的commands模块，即INSTALLED_APPS对应的各个模块。再根据subcommand从中这些包中获取到对应的Command，返回Command类对象。django后台服务中的Command继承自BaseCommand，并且实现了各自业务的handle方法。  
接着，通过返回的对象调用其run_from_argv方法，从名称可以看出，这个方法是通过命令行参数，进行函数调用的：
```python
def run_from_argv(self, argv):
    self._called_from_command_line = True
    parser = self.create_parser(argv[0], argv[1])

    if self.use_argparse:
        options = parser.parse_args(argv[2:])
        cmd_options = vars(options)
        # Move positional args out of options to mimic legacy optparse
        args = cmd_options.pop('args', ())
    else:
        options, args = parser.parse_args(argv[2:])
        cmd_options = vars(options)
    handle_default_options(options)
    try:
        self.execute(*args, **cmd_options)
    except Exception as e:
        if options.traceback or not isinstance(e, CommandError):
            raise

        if isinstance(e, SystemCheckError):
            self.stderr.write(str(e), lambda x: x)
        else:
            self.stderr.write('%s: %s' % (e.__class__.__name__, e))
        sys.exit(1)
    finally:
        connections.close_all()
```
我们知道 fetch_command 返回的Command对象继承自BaseCommand，那么不同的后台任务可能需要不同的参数信息，在run_from_argv方法中，通过调用create_parser方法，Command子类将不同的参数信息进行设置，再通过执行execute方法，最终调用子类Command对象中定义的handle方法，完成自定义项目中业务逻辑的实现。
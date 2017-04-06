在django配置文件settings.py中，使用DATABASES中指定项目中使用到的数据库连接信息，而通常我们只定义一个key为'default'的数据库连接信息，比如：
```python
DATABASES = {
    'default': {
        'NAME': conf.MYSQL_DBNAME,
        'ENGINE': 'django.db.backends.mysql',
        'USER': conf.MYSQL_USERNAME,
        'PASSWORD': conf.MYSQL_PASSWORD,
        'HOST': conf.MYSQL_HOST,
        'PORT': conf.MYSQL_PORT
    }
}
```
这节我们来看下在django项目中我们如何指定多个数据库连接信息，并且如何实现读写分离。

回顾下之前对model模块代码的分析，我们已经知道model.objects的增删改查操作都是委托其QuerySet类中的query类实例来实现，也就是说这些操作指向哪个数据库连接，也是由query类实例完成的。在分析model模块的源码时，同样提到在使用objects进行CURD时，都会调用它来获取具体的Query类及其子类实例(django源码中删除操作未调用get_compiler，子类分别为InserQuery、UpdateQuery、DeleteQuery)，并最终通过SQLCompiler类及其子类(分别为SQLInsertCompiler、SQLUpdateCompiler、SQLDeleteCompiler)完成增删改查操作。这里在调用get_compiler方法时，需要指定using参数，表示使用settings.DATABASES中的哪个数据库配置，get_compiler方法定义如下：
```python
class Query(object):
   
    def get_compiler(self, using=None, connection=None):
        if using is None and connection is None:
            raise ValueError("Need either using or connection")
        if using:
            connection = connections[using]
        return connection.ops.compiler(self.compiler)(self, connection, using)

    # other function ...
```
在调用get_compiler时，需要指定using或connection参数，表示这里需要对哪一个数据库进行操作。这个参数在QuerySet中表示为db属性，也就是说，在调用models.objects.all/count/filter等方法时，最终将db参数传递给get_compiler方法，db的定义如下：
```python
class QuerySet(object):
    # other functions ...
    
    @property
    def db(self):
        "Return the database that will be used if this query is executed now"
        if self._for_write:
            return self._db or router.db_for_write(self.model, **self._hints)
        return self._db or router.db_for_read(self.model, **self._hints)
```
db方法被property修饰为一个类属性，这里的router对象是全局的ConnectionRouter实例，通过它我们可以配置数据库的路由信息，db_for_write和db_for_read是ConnectionRouter的两个方法：
```python
# django.db.utils.py

class ConnectionRouter(object):
    def __init__(self, routers=None):
        """
        If routers is not specified, will default to settings.DATABASE_ROUTERS.
        """
        self._routers = routers

    @cached_property
    def routers(self):
        if self._routers is None:
            self._routers = settings.DATABASE_ROUTERS
        routers = []
        for r in self._routers:
            if isinstance(r, six.string_types):
                router = import_string(r)()
            else:
                router = r
            routers.append(router)
        return routers

    def _router_func(action):
        def _route_db(self, model, **hints):
            chosen_db = None
            for router in self.routers:
                try:
                    method = getattr(router, action)
                except AttributeError:
                    # If the router doesn't have a method, skip to the next one.
                    pass
                else:
                    chosen_db = method(model, **hints)
                    if chosen_db:
                        return chosen_db
            instance = hints.get('instance')
            if instance is not None and instance._state.db:
                return instance._state.db
            return DEFAULT_DB_ALIAS
        return _route_db

    db_for_read = _router_func('db_for_read')
    db_for_write = _router_func('db_for_write')
    
    # other functions ...
```
类定义中，使用cached_property装饰routers(在分析connections时已经介绍此装饰器)，其表示的是配置文件settings.py中的DATABASE_ROUTERS属性，默认为空list，可以在项目配置文件settings.py中通过此属性来指定我们自定义的路由配置。默认情况下，我们通常只会配置一个default的数据库连接信息，如果需要同时配置多个，需要将settings.DATABASES和settings.DATABASE_ROUTERS结合使用，前者配置多个数据库信息，后者需要指明绝对路径，通过import_string将我们自定义的路由信息类加载到内存(routers)，具体实现我们稍后会看到。db_for_read和db_for_write是_router_func通过不同参数返回的两个不同方法，从_router_func的定义中可以看出，它会遍历routers，并且从routers中加载的路由信息中，获取名为db_for_read或db_for_write的方法，通常在这些方法中返回我们指定的某些app的特殊路由，这里同样**通过db_for_read和db_for_write可以让我们从代码层面上实现对数据库操作的读写分离。如果DATABASES_ROUTER为空，或者DATABASES_ROUTER中的类未定义db_for_read/db_for_write方法，则会使用DATABASES中default的数据库配置**。我们通过一个简单的例子来说明下：
```python
# settings.py

DATABASES = {
    'default': {
        'NAME': 'project',
        'ENGINE': 'django.db.backends.mysql',
        'USER': 'tourun',
        'PASSWORD': 'tourun',
        'HOST': host_ip,
        'PORT': 3306
    },
    # app1 的数据库配置
    'app1': {
        'NAME': 'app1',
        'ENGINE': 'django.db.backends.mysql',
        'USER': 'tes1',
        'PASSWORD': 'tes1'
        'HOST': host_ip,
        'PORT': 13306
    }
    # app2 的数据库配置
   'app2': {
        'NAME': 'app2',
        'ENGINE': 'django.db.backends.mysql',
        'USER': 'tes2',
        'PASSWORD': 'tes2'
        'HOST': host_ip,
        'PORT': 23306
    }
}  # 多个数据库路由

DATABASE_ROUTERS = ['project.database_router.Model1Router' , 'project.database_router.Model2Router']  # 自定义的路由类

# project.database_router.py

class App1Router(object):
    def db_for_read(self, model, **hints):
        if model._meta.app_label == 'app1':
            return 'app1'
        return None

    def db_for_write(self, model, **hints):
        if model._meta.app_label == 'app1':
            return 'app1'
        return None


class App2Router(object):
    def db_for_read(self, model, **hints):
        if model._meta.app_label == 'app2':
            return 'app2'
        return None

    def db_for_write(self, model, **hints):
        if model._meta.app_label == 'app2':
            return 'app2'
        return None
```
上面的代码在配置文件settings.py中，通过DATABASES中指定了多个数据库配置，其中app1和app2是配置文件settings.py中INSTALLED_APP中的app信息，在DATABASES_ROUTER中指定了自定义路由信息配置类的绝对路径，那么我们在使用models进行数据的查询时，会循环DATABASES_ROUTER(for in routers)中加载的信息，在db_for_read/db_for_write的中，model表示自定义的表类，继承自Model基类，_meta表示model的属性，其中app_label表示app对应的字符串(相关代码可以查看ModelBase.\__new\__)，运行时如果遇到app名为app1的模块，则使用DATABASES中app1的数据库配置，app2同理，其他情况则使用default中的数据库配置。通过这种方式，我们实现了项目中不同的app操作不同的数据库。

与之类似，我们可以修改两个自定义类的db_for_read/db_for_write方法，使其返回在DATABASES中定义的不同的key，即实现应用层的读写分离，比如：
```python
class App1Router(object):
    def db_for_read(self, model, **hints):
        if model._meta.app_label == 'app1':
            return 'default'
        return None

    def db_for_write(self, model, **hints):
        if model._meta.app_label == 'app1':
            return 'app1'
        return None
```
在运行时遇到app名为app1的模块时，其对数据库的读操作指向default对应的数据库，而写操作则指向app1对应的数据库。但是在django中是如何识别当前的操作是读还是写呢，原来在QuerySet类中有定义了属性_for_write，默认为False表示读操作，那么通过objects.all/count等操作时，db属性会返回router.db_for_read方法，而当调用objects.create/update/delete等操作时，将_for_write赋值为True，表示当前操作为写操作，db属性会返回router.db_for_write方法，从而实现读写指向不同的数据库。
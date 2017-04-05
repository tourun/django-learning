在使用django操作数据库时，需要与数据库建立连接，通常要用到connections，相关的实现在django.db模块中，这节我们就俩看下django是如何通过connections操作数据库的。

先看下connections的定义：
```python
# django.db.__init__.py

connections = ConnectionHandler()
```
它是一个全局的ConnectionHandler对象，用来存放database的配置信息并且管理其连接。**ConnectionHandler重写了__getitem__、__setitem__方法，实现对其进行以索引形式的访问和设置，类似字典**：
```python
# django.db.utils.py

class ConnectionHandler(object):
    def __init__(self, databases=None):
        # 存放settings.DATABASES中的信息
        self._databases = databases
        # 存放连接信息
        self._connections = local()
        
    @cached_property
    def databases(self):
        if self._databases is None:
            self._databases = settings.DATABASES
        if self._databases == {}:
            self._databases = {
                DEFAULT_DB_ALIAS: {
                    'ENGINE': 'django.db.backends.dummy',
                },
            }
        if DEFAULT_DB_ALIAS not in self._databases:
            raise ImproperlyConfigured("You must define a '%s' database" % DEFAULT_DB_ALIAS)
        return self._databases

    def __getitem__(self, alias):
        if hasattr(self._connections, alias):
            return getattr(self._connections, alias)

        self.ensure_defaults(alias)
        self.prepare_test_settings(alias)
        db = self.databases[alias]
        backend = load_backend(db['ENGINE'])
        conn = backend.DatabaseWrapper(db, alias)
        setattr(self._connections, alias, conn)
        return conn

    def __setitem__(self, key, value):
        setattr(self._connections, key, value)
        
    def ensure_defaults(self, alias):
        try:
            conn = self.databases[alias]
        except KeyError:
            raise ConnectionDoesNotExist("The connection %s doesn't exist" % alias)

        conn.setdefault('ATOMIC_REQUESTS', False)
        conn.setdefault('AUTOCOMMIT', True)
        conn.setdefault('ENGINE', 'django.db.backends.dummy')
        if conn['ENGINE'] == 'django.db.backends.' or not conn['ENGINE']:
            conn['ENGINE'] = 'django.db.backends.dummy'
        conn.setdefault('CONN_MAX_AGE', 0)
        conn.setdefault('OPTIONS', {})
        conn.setdefault('TIME_ZONE', 'UTC' if settings.USE_TZ else settings.TIME_ZONE)
        for setting in ['NAME', 'USER', 'PASSWORD', 'HOST', 'PORT']:
            conn.setdefault(setting, '')    
        
    # other functions ...
```
根据不同的db（db表示settings.DATABASES中的不同数据库配置）使用connections[db]连接时，调用__getitem__方法，ensure_defaults设置databases的默认选项，databases存放配置信息，确保连接属性的正确性。**注意在定义databases时用到了cached_property这个装饰器**，cached_property是一个描述符类：
```python
# django.utils.functional.py

class cached_property(object):
    def __init__(self, func, name=None):
        self.func = func
        self.__doc__ = getattr(func, '__doc__')
        self.name = name or func.__name__

    def __get__(self, instance, type=None):
        if instance is None:
            return self
        res = instance.__dict__[self.name] = self.func(instance)
        return res
```
从它的实现上可以看出，被cached_property修饰的方法，实际上会变成一个对象实例，比如这里：
```python
# 装饰器的原理如下，可以看做databases_instance是经过cached_property修饰后的一个实例
databases_instance = cached_property(databases)
```
在cached_property的初始化方法中，self.func指向了原来的databases，返回一个新的cached_property对象实例。当connections有访问databases到时，会间接调用databases_instance.\__get\__，它将计算过得res直接放置在instance的\__dict\__中，这样避免了重复的运算，在下次访问connections.databases时，直接从其__dict__中获取，不在通过描述符调用。
> 属性访问的原理与描述器  
访问对象属性时，实际上会调用object.__getattribute__()，先访问对象的__dict__，如果没有再访问类（或父类，元类除外）的__dict__，最后如果此属性在这个__dict__中是一个描述器，则会调用描述器的__get__方法。而如何对__get__的参数进行传递，这取决于调用描述符的是对象还是类，如果是对象obj.x，则会调用type(obj).\__dict\__['x'].\__get\__(obj, type(obj))。如果是类，class.x， 则会调用type(class).\__dict\__['x'].\__get\__(None, type(class)。

回到__getitem__方法，接下来，prepare_test_settings方法设置DATABASES的Test相关参数。 接着通过load_backend加载后台的使用的具体数据库模块，我们知道django后台支持多种数据库类型，比如mysql、oracle、sqlite3等，这些通常在settings.DATABASES.engine中指定，那么load_backend就是通过engine指定的数据库类型，从相应的目录中（django.db.backend.mysql/oracle/sqlite3等）加载其base模块，并返回DatabaseWrapper类对象，每种数据库类型定义的DatabaseWrapper均继承自BaseDatabaseWrapper，用来实现对数据库的管理操作，例如新建连接、提交事务、回滚等。

来看下在基类BaseDatabaseWrapper中是如何建立连接的，**在BaseDatabaseWrapper基类中定义了连接对象connection，初始化时为None，使用connect方法用来建立连接，而它也是django内部获取连接的唯一入口**。通过调用connect方法对connection赋值：
```python
# django.db.backends.base.base.py

def connect(self):
    """Connects to the database. Assumes that the connection is closed."""
    # In case the previous connection was closed while in an atomic block
    self.in_atomic_block = False
    self.savepoint_ids = []
    self.needs_rollback = False
    # Reset parameters defining when to close the connection
    max_age = self.settings_dict['CONN_MAX_AGE']
    self.close_at = None if max_age is None else time.time() + max_age
    self.closed_in_transaction = False
    self.errors_occurred = False
    # Establish the connection
    conn_params = self.get_connection_params()
    self.connection = self.get_new_connection(conn_params)
    self.set_autocommit(self.settings_dict['AUTOCOMMIT'])
    self.init_connection_state()
    connection_created.send(sender=self.__class__, connection=self)
```
创建连接时，首先通过get_connection_params方法获取建立连接需要的参数信息，接着通过这些参数调用get_new_connections方法，获取真正的数据库连接，最后调用init_connection_state方法，对数据库连接做一些初始化设置。每种具体的数据库类型的连接的建立方式和连接所需的参数各有不同，所以不难看出这三个方法都由具体的DatabaseWrapper子类重写，这些子类定义在django.db.backend路径下不同的目录中，每个子类获取连接参数、建立连接、初始化连接的方法不同，但是都需要通过安装对应相应的数据库python包来完成对连接的操作，这些依赖的python包需要提前安装，比如mysql需要MySQLdb的支持：
```python
# django.db.backends.mysql.base.py

try:
    import MySQLdb as Database
except ImportError as e:
    from django.core.exceptions import ImproperlyConfigured
    raise ImproperlyConfigured("Error loading MySQLdb module: %s" % e)


class DatabaseWrapper(BaseDatabaseWrapper):
    vendor = 'mysql'
    # other functions ...
    
    def get_new_connection(self, conn_params):
        conn = Database.connect(**conn_params)
        conn.encoders[SafeText] = conn.encoders[six.text_type]
        conn.encoders[SafeBytes] = conn.encoders[bytes]
        return conn 
        
    # other functions ...
```
而oracle需要cx_Oracle的支持：
```python
# django.db.backends.oracle.base.py

try:
    import cx_Oracle as Database
except ImportError as e:
    from django.core.exceptions import ImproperlyConfigured
    raise ImproperlyConfigured("Error loading cx_Oracle module: %s" % e)
    
    
class DatabaseWrapper(BaseDatabaseWrapper):
    vendor = 'oracle'
    # other functions ...
    
    def get_new_connection(self, conn_params):
        conn_string = convert_unicode(self._connect_string())
        return Database.connect(conn_string, **conn_params)
        
    # other functions ...
```
知道connect方法是django中获取连接的唯一途径后，接着我们来反向查找下在何时会通过connect方法建立连接。在BaseDatabaseWrapper基类中，摘出有调用到connect的相关方法（注意这里并不是全部），展示如下：
```python
# django.db.backends.base.base.py

class BaseDatabaseWrapper(object):
    # other functions ...
    
    def ensure_connection(self):
        """
        Guarantees that a connection to the database is established.
        """
        if self.connection is None:
            with self.wrap_database_errors:
                self.connect()

    def _cursor(self):
        self.ensure_connection()
        with self.wrap_database_errors:
            return self.create_cursor()
            
    def cursor(self):
        """
        Creates a cursor, opening a connection if necessary.
        """
        self.validate_thread_sharing()
        if self.queries_logged:
            cursor = self.make_debug_cursor(self._cursor())
        else:
            cursor = self.make_cursor(self._cursor())
        return cursor
        
    def make_cursor(self, cursor):
        """
        Creates a cursor without debug logging.
        """
        return utils.CursorWrapper(cursor, self)
        
    def _savepoint(self, sid):
        with self.cursor() as cursor:
            cursor.execute(self.ops.savepoint_create_sql(sid))
            
    def get_autocommit(self):
        """
        Check the autocommit state.
        """
        self.ensure_connection()
        return self.autocommit
        
    def set_autocommit(self, autocommit):
        """
        Enable or disable autocommit.
        """
        self.validate_no_atomic_block()
        self.ensure_connection()
        self._set_autocommit(autocommit)
        self.autocommit = autocommit
        
    # other functions ...
```
ensure_connection方法会调用connect获取连接，在BaseDatabasesWrapper类中，_cursor()/get_autocommit()/set_autocommit()方法中都有调用ensure_connection，这三个方法分别用于创建游标，获取和设置自动提交的状态。

_cursor方法通常由基类或子类对象进行调用，实际上它是由BaseDatabaseWrapper的cursor方法进行调用，注意到_cursor方法返回了self.create_cursor()，不难猜测，这个方法同样由具体的子类实现，返回各个数据库的游标对象。在BaseDatabaseWrapper类中_cursor方法由cursor方法调用。make_cursor返回的CursorWrapper类对象实现了\__enter\__和\__exit\__方法，使cursor方法支持with关键字的使用，在BaseDatabaseWrapper中定义了savepoint方法用来实现数据库的回滚操作。

除了BaseDatabaseWrapper基类本身之外，还有哪里会通过cursor方法来连接数据库呢？我们知道，在django项目中，models模块用来定义项目中用到的数据库表，通过django的后台命令python manage.py migrate来创建项目中自定义的表，并且可以通过models.objects更便捷的实现增删改查等操作。

我们先来看下migrate命令，它是django提供的一个数据表同步的命令，对应的文件是 django.core.management.commands 目录下的 migrate.py。之前我们分析过，在此目录下的命令，都是django自身提供的一些核心指令，用以完成一些项目中的基础操作。文件中定义了Commands，继承自基类BaseCommands，并重写了handle方法，用来实现数据库表的同步操作。在获取连接之前需要获取数据库的配置信息，handle方法中默认使用的是 settings.py中DATABASES.default的连接参数，也就是说default参数中配置了什么类型的数据库（mysql、orable、sqlite），就会创建相应类型的connection，见文章开头对connections的解释。

handle方法中引入全局对象apps加载到的INSTALLED_APPS模块实例，然后调用一个名为sync_apps的方法，将models中定义的表同步到数据库中。在sync_apps中，我们看到它首先会获取游标：
```python
cursor = connection.cursor()
```
connection对象是通过connections全局对象和settings.DATABASES中配置的数据库信息创建的：
```python
db = options.get('database')
connection = connections[db]
```
我们知道它实际是一个DatabaseWrapper实例，调用cursor方法，cursor方法中调用ensure_connection方法，建立真正的连接。我们来看下sync_apps方法的主要逻辑：
```python
with transaction.atomic(using=connection.alias, savepoint=connection.features.can_rollback_ddl):
    deferred_sql = []
    for app_name, model_list in manifest.items():
        for model in model_list:
            if model._meta.proxy or not model._meta.managed:
                continue
            if self.verbosity >= 3:
                self.stdout.write(
                    " Processing %s.%s model\n" % (app_name, model._meta.object_name)
                )
            with connection.schema_editor() as editor:
                if self.verbosity >= 1:
                    self.stdout.write("    Creating table %s\n" % model._meta.db_table)
                editor.create_model(model)
                deferred_sql.extend(editor.deferred_sql)
                editor.deferred_sql = []
            created_models.add(model)

    if self.verbosity >= 1:
        self.stdout.write("Running deferred SQL...\n")
    for statement in deferred_sql:
        cursor.execute(statement)
```
transation.atomic保证sql执行的原子性，manifest中保存所有要创建的models模块信息。创建表时使用了:
```python
editer.create_model(model)
```
editor是一个connection.schema_editor方法返回的对象，具体为BaseDatabaseSchemaEditor的子类类型，它是BaseDatabaseWrapper的类属性，我们知道对表的操作（增删改查、添加索引外键等）都需要建表语法的支持，那么不同的数据库类型，建表语法上也可能略有不同，在django中，表的创建实际是由这些不同的editor完成的。与BaseDatabaseWrapper类似，不同的数据库会定义不同的SchemaEditor子类，对应的定义在django.db.backend.mysql/oracle/sqlite3目录中的schema.py中。如上所述，schema_editor方法根据connection代表的具体数据库类型，返回相应的建表语法实例，再通过create_model方法，生成对应的语句，最后通过游标cursor执行这些语句。
```python
for statement in deferred_sql:
    cursor.execute(statement)
```
与migrate相似的还有sqlall、createcachetable、loaddata等命令，同样通过connect获取cursor，来执行相应的建表语句，完成对表结构的操作。

这里简单分析了django的connection模块内部设计，主要用于完成数据库的相关操作，而在我们后台最常用的models相关操作，实际也是通过connection模块完成的，下来还有接着对models模块进行分析。
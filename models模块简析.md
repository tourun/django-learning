在django中我们通常使用models.objects的相关方法来完成对数据库的增删改查操作，这里我们就来看下这些简单方法的背后是如何实现的。

要理解当我们使用models.objects时，objects代表什么，我们知道models表示自定义的表，继承自django.db.models.base.Model这个类，那么猜测objects应该是Model的一个属性，注意这个属性并不是Model基类中定义的，来看下Model类的声明，以及相关上下文：
```python
def with_metaclass(meta, *bases):
    class metaclass(meta):
        def __new__(cls, name, this_bases, d):
            return meta(name, bases, d)
    return type.__new__(metaclass, 'temporary_class', (), {})


class ModelBase(type):
    # functions ...
    

class Model(six.with_metaclass(ModelBase)):
    # functions ...
```
Model类，继承自with_metaclass(ModelBase)，with_metaclass方法中使用type返回了一个动态创建的类，这里type是python的一个build-in class，注意并不是一个方法。我们通常用它来获取对象的类型，它的另一个更强大的功能是被用来动态的创建类：
```python
class type(object)
  | type(object) -> the object's type
  | type(name, bases, dict) -> a new type
```
在python中，用来创建类的东西被称作元类(metaclass)，python中用type这个内建类，或者它的子类来实现元类的功能。**如果类中定义了__metaclass__属性，会在这个类定义时，执行元类\__metaclass\__指向的类)的__new__方法(如果有定义)**。注意with_metaclass方法结尾处：
```python
   return type.__new__(metaclass, 'temporary_class', (), {})
```
这里通过type\.\__new\__，新建并返回一个类名(class\.\__name\__)为temporary_class的动态类，这里第一个参数metaclass在这里的含义是指被创建的类的元类类型，也就是说temporary_class这个动态类的元类是metaclass，它定义在with_metaclass方法中，并且继承自meta，根据Model与with_metaclass的定义可以看出meta实际上是一个名为ModelBase的基类，ModelBase类继承自type，而metaclass继承自ModelBase，它同样是一个元类。

不过这里并没有出现我们之前提到的__metaclass__属性，在网上查阅了相关的文档，找到以下说明：
> 
    Create a new class with base class base and metaclass metaclass. This is designed to be used in class declarations like this:

    from six import with_metaclass

    class Meta(type):
        pass

    class Base(object):
        pass

    class MyClass(with_metaclass(Meta, Base)):
        pass

    This is needed because the syntax to attach a metaclass changed between Python 2 and 3:
    
    Python 2:
    
    class MyClass(object):
        __metaclass__ = Meta

    Python 3:
    
    class MyClass(metaclass=Meta):
        pass
    
    The with_metaclass() function makes use of the fact that metaclasses are:
    1) inherited by subclasses
    2) a metaclass can be used to generate new classes
    it effectively creates a new base class by using the metaclass as a factory to generate an empty class
    
    文中提到使用with_metaclass来创建一个拥有基类base以及元类metaclass新类，with_metaclass的必要性在于元类的语法在python2和pyhton3中有改变，并且指出如何使用with_metaclass实现元类功能。
    
现在我们知道了Model实际由metaclass动态生成的，在这个类定义时，执行元类的__new__方法，由元类来创建我们定义的类。我们来看下这里的元类metaclass的__new__方法，它会直接返回meta类实例，也就是ModelBase的实例，而**在python创建类实例时，会先调用__new__方法，接着会在__new__返回的实例上进行__init__方法**，完成实例的初始化，通过meta(name, bases, d)会调用ModelBase的__new__方法。到这里，我们需要再梳理一下：
> 假设我们自定义的一个User类，继承自django.db.models.base.Model，而Model类本身继承自with_metaclass(ModelBase)，这个方法返回由metaclass创建的一个动态类，metaclass是一个元类，继承自ModelBase,在其构造函数中(metaclass\.\__new\__)新建并返回一个ModelBase实例，也就是说，我们自定义的User类最终会通过这个名为ModelBase的元类来创建。

ModelBase.__new__方法定义的较为复杂，这里不单独展示出来了，它主要完成的功能如下:
* 使用元类(ModelBase)新建一个类类型new_class
* 根据new_class类的__module__属性，查找其app_config(根据settings.py中INSTALLED_APP中的APP加载到的相关信息，具体可以查看Command模块相关代码)
* 设置new_class类的_meta属性，new_class.add_to_class('_meta', Opt ions(meta, **kwargs))
* 根据创建new_class时的attrs，设置其属性，比如表中各字段的Field
* 设置Model的object属性，cls.add_to_class('objects', Manager())

其在设置Manager实例时，通过ensure_default_manager方法完成：
```python
# django.db.models.manager.py

def ensure_default_manager(cls):
    if cls._meta.abstract:
        setattr(cls, 'objects', AbstractManagerDescriptor(cls))
        return
    elif cls._meta.swapped:
        setattr(cls, 'objects', SwappedManagerDescriptor(cls))
        return
    if not getattr(cls, '_default_manager', None):
        if any(f.name == 'objects' for f in cls._meta.fields):
            raise ValueError(
                "Model %s must specify a custom Manager, because it has a "
                "field named 'objects'" % cls.__name__
            )
        # Create the default manager, if needed.
        cls.add_to_class('objects', Manager())
        cls._base_manager = cls.objects
    elif not getattr(cls, '_base_manager', None):
        default_mgr = cls._default_manager.__class__
        if (default_mgr is Manager or
                getattr(default_mgr, "use_for_related_fields", False)):
            cls._base_manager = cls._default_manager
        else:
            # Default manager isn't a plain Manager class, or a suitable
            # replacement, so we walk up the base class hierarchy until we hit
            # something appropriate.
            for base_class in default_mgr.mro()[1:]:
                if (base_class is Manager or
                        getattr(base_class, "use_for_related_fields", False)):
                    cls.add_to_class('_base_manager', base_class())
                    return
            raise AssertionError(
                "Should never get here. Please report a bug, including your "
                "model and model manager setup."
            )
```
参数cls表示自定义的Models类（由ModelBase.__new__创建），来看下设置objects属性的语句及其实现：
```python
cls.add_to_class('objects', Manager())

class ModelBase(type):
    # other functions ...
    
    def add_to_class(cls, name, value):
        # We should call the contribute_to_class method only if it's bound
        if not inspect.isclass(value) and hasattr(value, 'contribute_to_class'):
            value.contribute_to_class(cls, name)
        else:
            setattr(cls, name, value)
            
    # other functions ...
```
在add_to_class方法中，value为Manager实例，它继承自BaseManager类，而BaseManager中有定义contribute_to_class方法，接着调用它的contribute_to_class方法对objects进行设置：
```python
class BaseManager(object):
    # other functions ...
    def contribute_to_class(self, model, name):
        # TODO: Use weakref because of possible memory leak / circular reference.
        self.model = model
        if not self.name:
            self.name = name
        # Only contribute the manager if the model is concrete
        if model._meta.abstract:
            setattr(model, name, AbstractManagerDescriptor(model))
        elif model._meta.swapped:
            setattr(model, name, SwappedManagerDescriptor(model))
        else:
            # if not model._meta.abstract and not model._meta.swapped:
            setattr(model, name, ManagerDescriptor(self))
        if (not getattr(model, '_default_manager', None) or
                self.creation_counter < model._default_manager.creation_counter):
            model._default_manager = self

        abstract = False
        if model._meta.abstract or (self._inherited and not self.model._meta.proxy):
            abstract = True
        model._meta.managers.append((self.creation_counter, self, abstract))
		
    # other functions ...
```
实际的设置语句是:
```python
    setattr(model, name, ManagerDescriptor(self))
```
model表示新建的model类，name表示'object'字符串，self表示的是Manager实例本身，ManagerDescriptor从名称来看是一个描述符，用来修饰Manager实例。这里有疑问的是为什么不将object属性直接设置成Manager实例，即：
```python
    setattr(model, name, self)
```
而是要设置为一个描述符，通过描述符来修饰Manager实例，我们先来看下ManagerDescription的定义：
```python
class ManagerDescriptor(object):
    # This class ensures managers aren't accessible via model instances.
    # For example, Poll.objects works, but poll_obj.objects raises AttributeError.
    def __init__(self, manager):
        self.manager = manager

    def __get__(self, instance, type=None):
        if instance is not None:
            raise AttributeError("Manager isn't accessible via %s instances" % type.__name__)
        return self.manager
```
这里需要解释下__get__方法的参数，**instance表示类实例，type表示类类型，当我们通过类实例来访问属性时，instance表示的是实例本身，而当我们通过类名来访问属性时，instance为None**。在__get__定义中，可以看到如果是实例来调用它时，那么会抛出异常，也就是说，**在django中，只有 Model 类可以使用 objects, 而Model 类的实例则不行**。

结合上下文，现在我们知道**objects实际上是一个Manger类实例，并在使用中需要以类属性的方式调用而非实例属性**，来看下Manager类的声明，以及相关上下文:
```python
# django.db.models.manager.py

class Manager(BaseManager.from_queryset(QuerySet)):
    # functions ...
    
    
class BaseManager(object):
    # other function ...
    
    @classmethod
    def from_queryset(cls, queryset_class, class_name=None):
        if class_name is None:
            class_name = '%sFrom%s' % (cls.__name__, queryset_class.__name__)
        class_dict = {
            '_queryset_class': queryset_class,
        }
        class_dict.update(cls._get_queryset_methods(queryset_class))
        return type(class_name, (cls,), class_dict)
	
    @classmethod
    def _get_queryset_methods(cls, queryset_class):
        def create_method(name, method):
            def manager_method(self, *args, **kwargs):
                return getattr(self.get_queryset(), name)(*args, **kwargs)
            manager_method.__name__ = method.__name__
            manager_method.__doc__ = method.__doc__
            return manager_method

        new_methods = {}
        predicate = inspect.isfunction if six.PY3 else inspect.ismethod
        for name, method in inspect.getmembers(queryset_class, predicate=predicate):
            # Only copy missing methods.
            if hasattr(cls, name):
                continue
            # Only copy public methods or methods with the attribute `queryset_only=False`.
            queryset_only = getattr(method, 'queryset_only', None)
            if queryset_only or (queryset_only is None and name.startswith('_')):
                continue
            # Copy the method onto the manager.
            new_methods[name] = create_method(name, method)
        return new_methods
	# other function ...
```
我们已经知道type可以用来动态的生成一个类，在这里Manager继承自这个动态生成的类，结合上下文的参数
```python
    type(class_name, (cls,), class_dict)
```
返回的类继承自BaseManger，类名(class.\__name\__)为BaseManagerFromQuerySet，类的属性通过BaseManager._get_queryset_methods方法进行设置，通过代码可以看出这个方法实际是将QuerySet类的一些方法设置成自身的方法，那么在我们使用model.objects时，实际是通过QuerySet这个类进行操作的，QuerySet类包含了我们通过obejcts查询的方法：all、count、filter、create、exists、update、values、bulk_create、get_or_create等等。篇幅有限，这里只列出QuerySet中一些常见方法的声明：
```python
# django.db.models.query.py

class QuerySet(object):

    def __init__(self, model=None, query=None, using=None, hints=None):
        # 初始化
        
    def __len__(self):
        # 实现len方法
        
    def __iter__(self):
        # 迭代
        
    def __bool__(self):
        # 用于if判空
        
    def __getitem__(self, k):
        # 用于切片
        
    def iterator(self):
        # 迭代方法
        
    def aggregate(self, *args, **kwargs):
        # 聚合方法
        
    def count(self):
        # 统计个数
        
    def get(self, *args, **kwargs):
        # 获取
    
    def create(self, **kwargs):
        # 新建
        
    def bulk_create(self, objs, batch_size=None):
        # 批量新建
        
    def update(self, **kwargs):
        # 更新
        
    def exists(self):
        # 存在与否
        
    def all(self):
        # 所有集合
        
    def filter(self, *args, **kwargs):
        # 根据条件过滤集合
        
    def exclude(self, *args, **kwargs):
        # 根据条件排除集合
        
    # other functions ...
```
先来看看查询相关操作，在对all、count、filter等方法抽丝剥茧之后，会发现它们都是通过QuerySet中一个名为query的属性变量来完成的，query是一个sql.Query的类实例，有如下定义：
```python
# django.db.models.sql.query.py

class Query(object):
    """
    A single SQL query.
    """
    alias_prefix = 'T'
    subq_aliases = frozenset([alias_prefix])
    query_terms = QUERY_TERMS

    compiler = 'SQLCompiler'
    
    def get_compiler(self, using=None, connection=None):
        if using is None and connection is None:
            raise ValueError("Need either using or connection")
        if using:
            connection = connections[using]
        return connection.ops.compiler(self.compiler)(self, connection, using)

    # other function ...
```
Query的类属性compiler表示SQLCompiler类，在使用objects进行查询时，首先通过query.get_compiler获取到SQLCompiler类实例，再由SQLCompiler.execute_sql方法完成表的查询操作，定义如下：
```python
# django.db.models.sql.compiler.py

class SQLCompiler(object):
    # other functions ...
    
    def execute_sql(self, result_type=MULTI):    
        if not result_type:
            result_type = NO_RESULTS
        try:
            sql, params = self.as_sql()
            if not sql:
                raise EmptyResultSet
        except EmptyResultSet:
            if result_type == MULTI:
                return iter([])
            else:
                return

        cursor = self.connection.cursor()
        try:
            cursor.execute(sql, params)
        except Exception:
            cursor.close()
            raise

        if result_type == CURSOR:
            return cursor
        if result_type == SINGLE:
            try:
                val = cursor.fetchone()
                if val:
                    return val[0:self.col_count]
                return val
            finally:
                cursor.close()
        if result_type == NO_RESULTS:
            cursor.close()
            return

        result = cursor_iter(
            cursor, self.connection.features.empty_fetchmany_value,
            self.col_count
        )
        if not self.connection.features.can_use_chunked_reads:
            try:
                return list(result)
            finally:
                cursor.close()
        return result
        
    # other functions ...
```
在execute_sql方法中as_sql会创建查询语句以及执行它所需的参数，由cursor来执行查询语句并将结果返回。这里的cursor是通过connection获取的，它是一个DatabaseWrapper类对象，继承自BaseDatabaseWrapper，我们结合[之前的文章](https://github.com/tourun/django-learning/blob/master/django%20connections%E7%AE%80%E6%9E%90.md)已经介绍过，调用cursor方法会去创建真正的连接，在这里，不同的数据库类型需要安装不同的第三方python包完成数据库的操作，比如mysql需要MySQLdb，oracle需要cx_Oracle，具体的数据库操作还是需要通过这些第三方的包来完成。

通过对源码简单的分析，我们了解了使用model.objects来对model对应的表进行查询时，django后台是如何实现的，接着我们来看下如果通过model.objects完成create/update/delete等增删改等操作是如何实现的。我们已经知道objects返回Manager实例，它实际是通过QuerySet类实例提供的方法来进行增删改查的，那么这些对数据进行写的操作，同样由QuerySet完成，首先来看下create/bulk_create等插入数据的方法，只关注数据库相关的操作，抽丝剥茧后，会发现最终都会通过_insert这个私有方法来进行实际的插入，这里的步骤与查询类似，先通过get_compiler获取到SQLCompiler类实例，再由SQLCompiler.execute_sql方法完成插入操作：
```python
class QuerySet(object):
    # other functions...
    
    def _insert(self, objs, fields, return_id=False, raw=False, using=None):
        """
        Inserts a new record for the given model. This provides an interface to
        the InsertQuery class and is how Model.save() is implemented.
        """
        self._for_write = True
        if using is None:
            using = self.db
        query = sql.InsertQuery(self.model)
        query.insert_values(fields, objs, raw=raw)
        return query.get_compiler(using=using).execute_sql(return_id)
    
    # other functions...
```
这里query是InsertQuery类实例，继承自Query类：
```python
# django.db.models.sql.subqueries.py

class InsertQuery(Query):
    compiler = 'SQLInsertCompiler'
    
    # functions ...
```
调用get_compiler方法时，会返回SQLInsertCompiler类实例，它继承自SQLCompiler类，并重写了父类的execute_sql和as_sql，用以完成插入的操作，而实际的插入操作，仍然由各自数据库对应的python包来实现。

与create方法类似，通过models.objects进行update和delete相关操作时，同样有UpdateQuery和DeleteQuery类实例，继承自Query类，在执行update/delete时，创建相应的SQLUpdateCompiler和SQLDeleteCompiler实例，重写execute_sql和as_sql方法完成相应的更新和删除操作，这里不再赘述。

---

这节我们分析了django model相关的源码，主要了解了在通过model.objects进行CURD操作时，背后的处理逻辑，实际是通过QuerySet这个代理类来完成的，而最终会通过我们之前分析到的connections相关操作，来与数据库进行交互。
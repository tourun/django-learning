runserver是django自带的一个轻量级的web server，它使用django自带的WSGI Server运行，经常在开发和测试中使用，便于调试。 runserver运行方式为：
> python manage.py runserver [options] [optional port number, or ipaddr:port]

runserver是一种django自带的命令，文件位于django.core.management.commands目录下，我们之前分析过django的Commands命令入口，现在来看下runserver的业务逻辑。首先，runserver.py定义了Commands类，继承自BaseCommands类，添加了一些自定义参数，并重写handle方法，来实现web server的功能。来看看它有哪些参数：
```python
def add_arguments(self, parser):
    parser.add_argument('addrport', nargs='?',
        help='Optional port number, or ipaddr:port')
    parser.add_argument('--ipv6', '-6', action='store_true', dest='use_ipv6', default=False,
        help='Tells Django to use an IPv6 address.')
    parser.add_argument('--nothreading', action='store_false', dest='use_threading', default=True,
        help='Tells Django to NOT use threading.')
    parser.add_argument('--noreload', action='store_false', dest='use_reloader', default=True,
        help='Tells Django to NOT use the auto-reloader.')
```
需要指定runserver启动的addr和端口，ipv6选项表示是否使用ipv6格式的地址，默认否，nothreading选项表示是否启用多线程，默认是，noreload选项表示是否python的自动重载功能，默认是，它的具体作用下面会有解释。

接着来看下handle方法的定义：
```python
# django.core.management.commands.runserver.py

def handle(self, *args, **options):
    from django.conf import settings

    if not settings.DEBUG and not settings.ALLOWED_HOSTS:
        raise CommandError('You must set settings.ALLOWED_HOSTS if DEBUG is False.')

    self.use_ipv6 = options.get('use_ipv6')
    if self.use_ipv6 and not socket.has_ipv6:
        raise CommandError('Your Python does not support IPv6.')
    self._raw_ipv6 = False
    if not options.get('addrport'):
        self.addr = ''
        self.port = DEFAULT_PORT
    else:
        m = re.match(naiveip_re, options['addrport'])
        if m is None:
            raise CommandError('"%s" is not a valid port number '
                               'or address:port pair.' % options['addrport'])
        self.addr, _ipv4, _ipv6, _fqdn, self.port = m.groups()
        if not self.port.isdigit():
            raise CommandError("%r is not a valid port number." % self.port)
        if self.addr:
            if _ipv6:
                self.addr = self.addr[1:-1]
                self.use_ipv6 = True
                self._raw_ipv6 = True
            elif self.use_ipv6 and not _fqdn:
                raise CommandError('"%s" is not a valid IPv6 address.' % self.addr)
    if not self.addr:
        self.addr = '::1' if self.use_ipv6 else '127.0.0.1'
        self._raw_ipv6 = bool(self.use_ipv6)
    self.run(**options)
```
从代码中可以看出，在启动runserver命令时，要确保项目的配置文件settings.py中开起了debug模式，或者配置了ALLOW_HOST参数，ALLOWED_HOSTS 是为了限定请求中的host值，以防止黑客构造包来发送请求，只有在列表中的host才能访问。当端口参数为空时，默认使用8000端口。在对参数进行校验之后，最后调用了Commands类的run方法：
```python
def run(self, **options):
    use_reloader = options.get('use_reloader')

    if use_reloader:
        autoreload.main(self.inner_run, None, options)
    else:
        self.inner_run(None, **options)
```
user_reloader参数默认为True，表示使用python的自动重载功能。**具体来说，就是我们在使用python manage.py runserver作为web server时，当对代码做了修改，不需要重启runserver服务，修改后的代码会被重新加载执行**。这里通过python标准库中的autoreload.main方法实现自动重载功能，runserver的web server实现定义在inner_run方法中，我们来看下inner_run中主要逻辑：
```python
try:
    handler = self.get_handler(*args, **options)
    run(self.addr, int(self.port), handler,
        ipv6=self.use_ipv6, threading=threading)
except socket.error as e:
    # Use helpful error messages instead of ugly tracebacks.
    ERRORS = {
        errno.EACCES: "You don't have permission to access that port.",
        errno.EADDRINUSE: "That port is already in use.",
        errno.EADDRNOTAVAIL: "That IP address can't be assigned-to.",
    }
    try:
        error_text = ERRORS[e.errno]
    except KeyError:
        error_text = force_text(e)
    self.stderr.write("Error: %s" % error_text)
    # Need to use an OS exit because sys.exit doesn't work in a thread
    os._exit(1)
except KeyboardInterrupt:
    if shutdown_message:
        self.stdout.write(shutdown_message)
    sys.exit(0)
```
通过get_handler获取handler，实际调用get_internal_wsgi_application，返回application接口:
```python
# django.core.servers.basehttp.py

def get_internal_wsgi_application():
    from django.conf import settings
    app_path = getattr(settings, 'WSGI_APPLICATION')
    if app_path is None:
        return get_wsgi_application()

    try:
        return import_string(app_path)
    except ImportError as e:
        msg = (
            "WSGI application '%(app_path)s' could not be loaded; "
            "Error importing module: '%(exception)s'" % ({
                'app_path': app_path,
                'exception': e,
            })
        )
        six.reraise(ImproperlyConfigured, ImproperlyConfigured(msg),
                    sys.exc_info()[2])
```
先在settings配置文件中找到WSGI_APPLICATION的配置，它表示项目中wsgi的application接口的路径，通常定义在project目录下的wsgi.py文件中：
```python
# wsgi.py

import os

from django.core.wsgi import get_wsgi_application

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "project.settings")

application = get_wsgi_application()
```
实际上application是一个WSGIHandler对象：
```python
def get_wsgi_application():

    django.setup()
    return WSGIHandler()
```
之前的文章中有提到WSGI协议的规范，这里WSGIHandler对象即使实现了WSGI协议的对象，用来连接web server和python应用程序。

获取到此接口之后，调用run方法，runserver服务启动：
```python
# django.core.servers.basehttp.py

def run(addr, port, wsgi_handler, ipv6=False, threading=False):
    server_address = (addr, port)
    if threading:
        httpd_cls = type(str('WSGIServer'), (socketserver.ThreadingMixIn, WSGIServer), {})
    else:
        httpd_cls = WSGIServer
    httpd = httpd_cls(server_address, WSGIRequestHandler, ipv6=ipv6)
    if threading:
        httpd.daemon_threads = True
    httpd.set_app(wsgi_handler)
    httpd.serve_forever()
```
ipv6和threading参数通过启动命名传递过来，如果没有指定，那么ipv6为False，threading为True（表示以多线程方式运行）。函数中通过set_app，将application对象设置为httpd实例的属性，最终以httpd.serve_forever()方式运行，这里以多线程方式运行，所以httpd在这里表示WSGIServer类实例，这个类通过type动态生成，它继承自socketserver.ThreadingMixIn和WSGIServer类。**这里ThreadingMixIn类的定义使用了Mixin技术**：
> Mixin编程是一种开发模式，是一种将多个类中的功能单元的进行组合的利用的方式,通常mixin并不作为任何类的基类，也不关心与什么类一起使用，而是在运行时动态的同其他零散的类一起组合使用。使用mixin机制有如下好处：可以在不修改任何源代码的情况下，对已有类进行扩展；可以保证组件的划分；可以根据需要，使用已有的功能进行组合，来实现“新”类；很好的避免了类继承的局限性，因为新的业务需要可能就需要创建新的子类。

对于mixin的使用场景，摘自[stackoverflow](http://stackoverflow.com/questions/533631/what-is-a-mixin-and-why-are-they-useful)上的高票回答：There are two main situations where mixins are used:
- You want to provide a lot of optional features for a class.
- You want to use one particular feature in a lot of different classes.

我们来看下ThreadingMixIn类的定义：
```python
# SocketServer.py

class ThreadingMixIn:

    daemon_threads = False

    def process_request_thread(self, request, client_address):

        try:
            self.finish_request(request, client_address)
            self.shutdown_request(request)
        except:
            self.handle_error(request, client_address)
            self.shutdown_request(request)

    def process_request(self, request, client_address):

        t = threading.Thread(target = self.process_request_thread,
                             args = (request, client_address))
        t.daemon = self.daemon_threads
        t.start()
```
ThreadingMixIn类定义了两个方法，process_request新建一个线程来处理请求，process_request_thread方法是线程的回调函数，request表示请求，client_address表示客户端的ip信息。**通过ThreadingMixIn的定义，我们可以看出对于MinIn类来说，不是为了直接实例化而创建，而且它们的职责很单一，它们必须和另一个实现了所需的映射功能的类混合在一起用才行，也就是说ThreadingMixIn类必须和某个合适的server类混用才行，比如这里的WSGIServer。ThreadingMixIn类中并未定义finish_request、shutdown_request等方法，我们可以猜测WSGIServer类中有对这些函数的定义。其次MixIn类一般来说是没有状态的，这意味着MixIn类通常没有__init__方法，也没有实例变量**。

我们知道runserver一个web server，它的主要工作时接受request，进行处理，然后将处理的结果返回给客户的，那么在这之前一定会做一些初始化工作，比如绑定ip端口、创建socket进行监听等:
```python
httpd_cls = type(str('WSGIServer'), (socketserver.ThreadingMixIn, WSGIServer), {})  
httpd = httpd_cls(server_address, WSGIRequestHandler, ipv6=ipv6)
```
在创建httpd实例时，会调用它的__init__方法，而这里的httpd_cls继承自ThreadingMixIn和WSGIServer，前者未定义__init__方法，来看下WSGIServer是如何初始化的，先看下WSGIServer声明和它的继承体系，只展示类的__init__方法：
```python
class BaseServer:
    def __init__(self, server_address, RequestHandlerClass):
        """Constructor.  May be extended, do not override."""
        self.server_address = server_address
        self.RequestHandlerClass = RequestHandlerClass
        self.__is_shut_down = threading.Event()
        self.__shutdown_request = False
        
    # other functions ...
    
class TCPServer(BaseServer):
    def __init__(self, server_address, RequestHandlerClass, bind_and_activate=True):
        """Constructor.  May be extended, do not override."""
        BaseServer.__init__(self, server_address, RequestHandlerClass)
        self.socket = socket.socket(self.address_family,
                                    self.socket_type)
        if bind_and_activate:
            try:
                self.server_bind()
                self.server_activate()
            except:
                self.server_close()
                raise
    
    # other functions ...

class HTTPServer(SocketServer.TCPServer):

class WSGIServer(HTTPServer):

class WSGIServer(simple_server.WSGIServer, object):
    def __init__(self, *args, **kwargs):
        if kwargs.pop('ipv6', False):
            self.address_family = socket.AF_INET6
        super(WSGIServer, self).__init__(*args, **kwargs)
        
    # other functions ...
```
用一个简单的图来进行示例其继承关系如下：
```python
                    +------------+
                    | BaseServer |
                    +------------+
                          |
                          v
                    +-----------+
                    | TCPServer |
                    +-----------+
                          |
                          v
                    +-----------+
                    |HTTPServer |
                    +-----------+
                          |
                          v
+--------------+    +------------+
|ThreadingMixIn| +  | WSGIServer |
+--------------+    +------------+
                 |
                 v
           +-----------+
           | httpd_cls |
           +-----------+
```
从代码可以看出底层实际通过TCP连接来处理请求，在新建httpd_cls类实例时，会调用其父类的__init__进行初始化，依次设置信号量，创建TCP套接字，绑定地址与端口，进行监听。
```python
# SocketServer.py

class BaseServer:
    # other functions ...
    
    def serve_forever(self, poll_interval=0.5):
        self.__is_shut_down.clear()
        try:
            while not self.__shutdown_request:
                r, w, e = _eintr_retry(select.select, [self], [], [],
                                       poll_interval)
                if self in r:
                    self._handle_request_noblock()
        finally:
            self.__shutdown_request = False
            self.__is_shut_down.set()
            
    def _handle_request_noblock(self):
        try:
            request, client_address = self.get_request()
        except socket.error:
            return
            
        if self.verify_request(request, client_address):
            try:
                self.process_request(request, client_address)
            except:
                self.handle_error(request, client_address)
                self.shutdown_request(request)
                
    def process_request(self, request, client_address):
        self.finish_request(request, client_address)
        self.shutdown_request(request)
            
    # other functions ...
```
构造完成httpd实例后，通过其top基类BaseServer的serve_forever方法接受请求并处理，这里将实例本身绑定到可读fd集合，使用select方法完成多路复用，当可读的socket可读时，则表示有请求接入，函数返回，再通过_handle_request_noblock方法：首先通过get_request获取到请求的socket以及对端的地址信息，接着通过process_request处理请求，我们通过代码可以看到BaseServer实现了process_request，但是它的处理逻辑在同一个线程中，如果这时再有请求接入，那么就需要等待前一个请求处理完成之后，才能处理下一个，也就是说这是一种串行的执行方式。让我们再回到httpd实例的定义：
```python
httpd_cls = type(str('WSGIServer'), (socketserver.ThreadingMixIn, WSGIServer), {})  
httpd = httpd_cls(server_address, WSGIRequestHandler, ipv6=ipv6)
```
httpd_cls类继承自ThreadingMixIn和WSGIServer，之前看到过ThreadingMixIn类的定义，它实现了多线程的process_request方法，我们注意到在通过type生成WSGIServer类时，**继承列表中ThreadingMixIn类在WSGIServer前面，根据python的MRO特性，在httpd运行时会调用ThreadingMixIn类的process_request方法，以多线程方式来提高并发性**。

查看httpd_cls的__mro__属性，它表示多重继承时，属性的查找机制（顺序）：
```python
# httpd_cls.__mro__

(<class 'django.core.servers.basehttp.WSGIServer'>, <class SocketServer.ThreadingMixIn>, <class 'django.core.servers.basehttp.WSGIServer'>, <class wsgiref.simple_server.WSGIServer>, <class BaseHTTPServer.HTTPServer>, <class SocketServer.TCPServer>, <class SocketServer.BaseServer>, <type 'object'>)
```
ThreadingMixIn类中只重写了process_request方法，将串行的处理方式改为多线程方式，finish_request和shutdown_request等方法并未定义。根据MRO特性，finish_request会调用BaseServer类中的定义：
```python
# SocketServer.py

class BaseServer:
    # other functions ...
    
    def finish_request(self, request, client_address):
        """Finish one request by instantiating RequestHandlerClass."""
        self.RequestHandlerClass(request, client_address, self)
        
    # other functions ...
```
RequestHandlerClass类是在我们创建httpd实例时指定的：
```python
httpd = httpd_cls(server_address, WSGIRequestHandler, ipv6=ipv6)
```
在finish_request中创建了一个WSGIRequestHandler实例，它的参数request表示请求的socket，client_address表示对端的客户信息，self表示httpd实例本身。我们来看下WSGIRequestHandler的继承体系以及它的实例是如何初始化的，同样只展示__init__方法：
```python
class BaseRequestHandler:
    def __init__(self, request, client_address, server):
        self.request = request
        self.client_address = client_address
        self.server = server
        self.setup()
        try:
            self.handle()
        finally:
            self.finish()

class StreamRequestHandler(BaseRequestHandler):

class BaseHTTPRequestHandler(SocketServer.StreamRequestHandler):

class WSGIRequestHandler(BaseHTTPRequestHandler):

class WSGIRequestHandler(simple_server.WSGIRequestHandler, object):
    def __init__(self, *args, **kwargs):
        self.style = color_style()
        super(WSGIRequestHandler, self).__init__(*args, **kwargs)
```
在top基类BaseRequestHandler的初始化方法中，通过handle和finish方法完成对请求的处理。handle方法在WSGIRequestHandler中有重写：
```python
class WSGIRequestHandler(BaseHTTPRequestHandler):
    def handle(self):
        """Handle a single HTTP request"""

        self.raw_requestline = self.rfile.readline(65537)
        if len(self.raw_requestline) > 65536:
            self.requestline = ''
            self.request_version = ''
            self.command = ''
            self.send_error(414)
            return

        if not self.parse_request(): # An error code has been sent, just exit
            return

        handler = ServerHandler(
            self.rfile, self.wfile, self.get_stderr(), self.get_environ()
        )
        handler.request_handler = self      # backpointer for logging
        handler.run(self.server.get_app())
```
经由ServerHandler类实例handler，通过run方法处理请求。**创建handler实例时，这里用self.rfile和self.wfile实现对request请求的数据读写，get_environ获取和客户端请求相关的信息，包含了一些CGI规范要求的数据，运行run方法时，需要传递application实例，之前已经明确，它是WSGIHandler类对象，用来连接web server和python应用服务**。我们来看下ServerHandler的继承体系与run方法：
```python
class BaseHandler:
    # other functions ...
    
    def run(self, application):
        """Invoke the application"""
        # Note to self: don't move the close()!  Asynchronous servers shouldn't
        # call close() from finish_response(), so if you close() anywhere but
        # the double-error branch here, you'll break asynchronous servers by
        # prematurely closing.  Async servers must return from 'run()' without
        # closing if there might still be output to iterate over.
        try:
            self.setup_environ()
            self.result = application(self.environ, self.start_response)
            self.finish_response()
        except:
            try:
                self.handle_error()
            except:
                # If we get an error handling an error, just give up already!
                self.close()
                raise   # ...and let the actual server figure it out.
                
    # other functions ...
    
class SimpleHandler(BaseHandler):

class ServerHandler(SimpleHandler):

class ServerHandler(simple_server.ServerHandler, object):
```
setup_environ方法设置wsgi协议所需的一些配置信息，application实现wsgi协议的调用，我们知道application是一个WSGIHandler类的实例，那么它如何来完成函数调用呢，来看看WSGIHandler类的定义：
```python
class WSGIHandler(base.BaseHandler):
    initLock = Lock()
    request_class = WSGIRequest

    def __call__(self, environ, start_response):
        # Set up middleware if needed. We couldn't do this earlier, because
        # settings weren't available.
        if self._request_middleware is None:
            with self.initLock:
                try:
                    # Check that middleware is still uninitialized.
                    if self._request_middleware is None:
                        self.load_middleware()
                except:
                    # Unload whatever middleware we got
                    self._request_middleware = None
                    raise

        set_script_prefix(get_script_name(environ))
        signals.request_started.send(sender=self.__class__, environ=environ)
        try:
            request = self.request_class(environ)
        except UnicodeDecodeError:
            logger.warning('Bad Request (UnicodeDecodeError)',
                exc_info=sys.exc_info(),
                extra={
                    'status_code': 400,
                }
            )
            response = http.HttpResponseBadRequest()
        else:
            response = self.get_response(request)

        response._handler_class = self.__class__

        status = '%s %s' % (response.status_code, response.reason_phrase)
        response_headers = [(str(k), str(v)) for k, v in response.items()]
        for c in response.cookies.values():
            response_headers.append((str('Set-Cookie'), str(c.output(header=''))))
        start_response(force_str(status), response_headers)
        if getattr(response, 'file_to_stream', None) is not None and environ.get('wsgi.file_wrapper'):
            response = environ['wsgi.file_wrapper'](response.file_to_stream)
        return response
```
WSGIHandler类中实现了__call__方法，使其对象可以被调用，我们知道WSGI相当于是Web服务器和Python应用程序之间的桥梁，那么来看看分析下它是如何工作的。首先如果application实例尚未加载中间件，则会根据settings.MIDDLEWARE_CLASSES的配置加载中间件信息:
```python
self.load_middleware()
```
接着根据environ信息创建WSGIRequest实例
```python
request = self.request_class(environ)
```
并设置其相关属性，再通过这个WSGIRequest实例，返回一个HttpResponse实例：
```python
response = self.get_response(request)
```
这里主要由中间件来完成请求的匹配。接着设置response的header信息，通过start_response返回状态码和响应头，通过environ['wsgi.file_wrapper']封装最后将response返回。environ['wsgi.file_wrapper']表示FileWrapper类：
```python
class FileWrapper:
    """Wrapper to convert file-like objects to iterables"""

    def __init__(self, filelike, blksize=8192):
        self.filelike = filelike
        self.blksize = blksize
        if hasattr(filelike,'close'):
            self.close = filelike.close

    def __getitem__(self,key):
        data = self.filelike.read(self.blksize)
        if data:
            return data
        raise IndexError

    def __iter__(self):
        return self

    def next(self):
        data = self.filelike.read(self.blksize)
        if data:
            return data
        raise StopIteration
```
**FileWrapper支持迭代器协议，所以这里返回的response是一个迭代器对象，这里实际是字节流的封装。**

至此，handle方法处理了request，并且返回了response，再回到run方法中，通过finish_response将字节流发送出去。

整个runserver的处理流程和类的关系可以用下图来展示（转自网络）：
![img](http://static.zybuluo.com/rainybowe/vqcu9pqtjgwmv3h2b4g6ede0/xiong%20%282%29.png)

---
runserver是django自带的一个轻量级web server，更多的是用于开发过程中的调试，真正在生产环境中使用的方式是uwsgi+Nginx的方式，其中uWSGI是一个Web服务器，它实现了WSGI协议、uwsgi、http等协议。注意uwsgi是一种通信协议，而uWSGI是实现uwsgi协议和WSGI协议的Web服务器。Nginx通常作为代理服务器，实现负载均衡，处理静态文件，域名转发等功能。
# Flask

## overview

### 背景
Flask以及它所使用的wsgi库werkzeug和模板引擎jinja2都是由[Armin Ronacher](http://lucumr.pocoo.org/about/)和他的团队开发的。实际上Armin Ronacher早就开发出来了werkzeug开源库，旨在为框架封装一个良好的底层的API接口，但过了一段时间，Armin发现还是自己来先做一个吧，然后Flask就诞生了。

<!--break-->

### 任务
*分析从发起HTTP请求到响应请求之间的流程*

*涉及源代码文件：*

- Flask源码
	- app.py
- werkzeug源码
	- serving.py
- python基本库源码
	1. SocketServer.py
	2. BaseHTTPServer.py

	<br>

先来看一个最小的Flask应用：

	from flask import Flask
	app = Flask(__name__)

	@app.route('/')
	def hello_world():
    	return 'Hello World!'

	if __name__ == '__main__':
    	app.run()
    	
显而易见，`app.run`是核心代码。我们来看一下`app.py`中相关的代码：

	<app.py>
	def run(self, host=None, port=None, debug=None, **options):
		from werkzeug.serving import run_simple
		run_simple(host, port, self, **options)
		
	<serving.py>
	def run_simple(hostname, port, application, ...,
               request_handler=None, static_files=None, ...):
       srv = make_server(hostname, port, application, threaded,
                          processes, request_handler,
                          passthrough_errors, ssl_context,
                          fd=fd)
       srv.serve_forever()
       
    def make_server(host=None, port=None, app=None, threaded=False, processes=1,
                request_handler=None, passthrough_errors=False,
                ssl_context=None, fd=None):
       return BaseWSGIServer(host, port, app, request_handler,
                              passthrough_errors, ssl_context, fd=fd)         
       
可以看到实际上是make了一个`BaseWSGIServer`,这个server开始监听连接。下边来研究一下这个`BaseWSGIServer`。</br>
`BaseWSGIServer`继承自python基本库`BaseHTTPServer.py`中的`HTTPServer`, `HTTPServer`继承自`SocketServer.py`中的`TCPServer`。
	
	<serving.py>
	class BaseWSGIServer(HTTPServer, object):
		def __init__(self, host, port, app, handler=None, ...):
			if handler is None:
            	handler = WSGIRequestHandler
			HTTPServer.__init__(self, (host, int(port)), handler)
			self.app = app
			
	class HTTPServer(SocketServer.TCPServer):
		def server_bind(self):
			....
			
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
                
    <SocketServer.py>
    class BaseServer:
		def __init__(self, server_address, RequestHandlerClass):
        	"""Constructor.  May be extended, do not override."""
        	self.server_address = server_address
        	self.RequestHandlerClass = RequestHandlerClass
        	self.__is_shut_down = threading.Event()
        	self.__shutdown_request = False
	
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
            	
      	def finish_request(self, request, client_address):
      		# 这里非常重要，就是实例化请求处理类
      		self.RequestHandlerClass(request, client_address, self)
            	
            	
从这些代码可以看出，HTTP建立于TCP之上。<strong>`BaseServer`最需要注意的是在构造的时候设置了请求处理类`RequestHandlerClass`</strong>。也就是在`BaseWSGIServer`构造的时候设置了请求处理类。根据上边的了解，我们知道在`app.run()`的时候就把请求处理类设置好了，注意这里只是设置了请求处理类，并没有真正的去构建一个处理实例。那么它是在什么时候实例化呢？实际上它是在http请求真正到来的时候才会被实例化。随后我们会继续通过代码看到这个请求处理类在实例化的时候就会自动去处理这个请求。</br>
我们先来简单说一下一个请求的基本处理逻辑。

- 首先一个连接进来，`BaseServer`监听到连接，调用`self._handle_request_noblock()`处理
- `self._handle_request_noblock()`最终找到finish_request方法
- `finish_request`方法实例化请求处理类`RequestHandlerClass`

在`BaseWSGIServer`的构造函数中，我们看到如果我们不自己实现请求处理句柄的话，就会使用`werkzeug`库提供的`WSGIRequestHandler`。我们再来看一下这个类到底是一个什么东西。
	       
	
	<serving.py>
	class WSGIRequestHandler(BaseHTTPRequestHandler, object):
		def handle(self):
          rv = BaseHTTPRequestHandler.handle(self)
        	return rv
      	def handle_one_request(self):
      		self.raw_requestline = self.rfile.readline()
        	if not self.raw_requestline:
            	self.close_connection = 1
        	elif self.parse_request():		# 从tcp到http中间的处理逻辑
            	return self.run_wsgi()		# 真正的处理，随后我们会拿出来研究一下
      	def run_wsgi(self):
      		pass
    
    <BaseHTTPServer.py>  		
    class BaseHTTPRequestHandler(SocketServer.StreamRequestHandler):
	 	def parse_request(self):
	 		...
	 	def handle(self):
	 		self.handle_one_request()
	
	<SocketServer.py> 		
	class StreamRequestHandler(BaseRequestHandler):
		def setup(self):
			...
		def finish(self):
			...
			
	class BaseRequestHandler:
		def __init__(self, request, client_address, server):
        	self.request = request
        	self.client_address = client_address
        	self.server = server	# 这个非常重要，设置了request的server，我们随后会看到
        	self.setup()
        	try:
            	self.handle()
        	finally:
            	self.finish()
            	
一层一层的继承下来，实际上就是继承的python基本库里的基本请求处理类。这个类在实例化的时候会调用handle函数处理，handle函数我们在`WSGIRequestHandler`中复写了，handle函数会调用我们在`WSGIRequestHandler`中复写的`handle_one_request`函数。好了，终于找到头了。我们看一下这个`run_wsgi`是怎么处理的。函数稍微长一些，我们拣逻辑性的重要代码分析一下。

	def run_wsgi(self):
		def write(data):
			...
			self.wfile.write(data)		# 回复响应数据
		def start_response(status, response_headers, exc_info=None):
			...
			return write
		def execute(app):
			application_iter = app(environ, start_response)
			for data in application_iter:
				write(data)
		execute(self.server.app)

代码很清楚，就是调用了app来处理请求，然后app匹配url找到视图view，真正的处理。你说app是怎么来的？我们上边研究过了，它是在请求到来的时候，由server传递给请求处理类的。也就是说只有当请求连接的时候，才会实例化处理类，处理类在实例化的时候就会配置它的server。就是下边的代码。

	 <SocketServer.py>
    class BaseServer:
    	def finish_request(self, request, client_address):
      		# 这里非常重要，就是实例化请求处理类
      		self.RequestHandlerClass(request, client_address, self)
      		
    class BaseRequestHandler:
    	def __init__(self, request, client_address, server):
    		self.server = server
    
    <serving.py>		
    class BaseWSGIServer(HTTPServer, object):
    	def __init__(self, host, port, app, handler=None, ...):
    		self.app = app
    		
好了，以上的流程我们都清楚了。下边就开始研究`app(environ, start_response)`的奇妙了。   
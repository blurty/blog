# Flask

## overview

### 背景
Flask以及它所使用的wsgi库werkzeug和模板引擎jinja2都是由[Armin Ronacher](http://lucumr.pocoo.org/about/)和他的团队开发的。实际上Armin Ronacher早就开发出来了werkzeug开源库，旨在为框架封装一个良好的底层的API接口，但过了一段时间，Armin发现还是自己来先做一个吧，然后Flask就诞生了。

<!--break-->

### 任务
1. *分析`app(environ, start_response)`如何找到最终的处理请求函数*
2. *分析`url_map`和`url_rule`*

### 分析
首先还是来看一个最小的Flask应用：

	from flask import Flask
	app = Flask(__name__)

	@app.route('/')
	def hello_world():
    	return 'Hello World!'

	if __name__ == '__main__':
    	app.run()
    
#### 分析`app(environ, start_response)`	
<p>可以看到最核心的应用就是app这个东西。app是类Flask的一个实例。这个类会将很多东西集成进去，比如url_map, environ, config等等。这个随后再慢慢抽丝剥茧。现在先看一下`app(environ, start_response)`。 
<p>既然app是Flask的一个实例，那么app(...)说明了这个实例是可以被调用的，在python中只要实现了类的__call__方法，那么这个对象就是可调用的。 

	class Flask():
		def __call__(self, environ, start_response):
			return self.wsgi_app(environ, start_response)

	def wsgi_app(self, environ, start_response):
		ctx = self.request_context(environ)
      	ctx.push()
      	error = None
      	try:
           try:
               response = self.full_dispatch_request()
           except Exception as e:
               error = e
               response = self.handle_exception(e)
           except:
               error = sys.exc_info()[1]
               raise
           return response(environ, start_response)
      	finally:
           if self.should_ignore_error(error):
               error = None
           ctx.auto_pop(error)
           
分析一下wsgi_app方法，它主要做了三件事：

- 获取请求上下文
- 分发请求
- 响应请求

我们主要来看一下这个分发请求。
	
	def full_dispatch_request(self):
		# 触发第一个请求之前的hook。即装饰器before_first_request修饰的视图函数。
		self.try_trigger_before_first_request_functions()
		try:
            request_started.send(self)
            # 装饰器before_request修饰的视图函数。
            rv = self.preprocess_request()
            if rv is None:
                rv = self.dispatch_request()
        except Exception as e:
            rv = self.handle_user_exception(e)
        return self.finalize_request(rv)
        
    def dispatch_request(self):
    	req = _request_ctx_stack.top.request
      	if req.routing_exception is not None:
			self.raise_routing_exception(req)
    	rule = req.url_rule    # 最后边可以看到url_rule是如何生成的
    	return self.view_functions[rule.endpoint](**req.view_args)

<p>点击查看[url_rule的生成](#jump)</br>    	
`view_functions`存放的就是我们在程序中所有用`app.route`装饰过的视图函数。
<p>至此我们已经从app(environ, start_response)找到最终的处理请求的地方。接下来我们来看一下`url_map`。

#### url_map
url_map在Flask中被初始化为一个Map。Map是werkzeug库中的一个对象。我们来看一下：


#### url_rule
url_rule是类Rule的一个实例。当请求到来时，上下文环境会给request构造对应的rule用来匹配对应的视图函数。    	


### 分析路由的添加
由最上边的代码可以知道，在flask应用中添加一个新的路由的方式是使用`@app.route(...)`装饰器。看一下类Flask的这个方法。

    flask.app.py
	def route(self, rule, **options):
		def decorator(f):
			endpoint = options.pop('endpoint', None)
			self.add_url_rule(rule, endpoint, f, **options)
         	return f
     	return decorator
     	
    def add_url_rule(self, rule, endpoint=None, view_func=None, **options):
        if endpoint is None:
            # 将函数名作为endpoint
            endpoint = _endpoint_from_view_func(view_func)
        ...
        # 从字符串rule（'/main'）到rule类的转换
        rule = self.url_rule_class(rule, methods=methods, **options)
        self.url_map.add(rule)
        self.view_functions[endpoint] = view_func

***
        
    werkzeug.routing.py
    @implements_to_string
    class Rule(RuleFactory):
        def __init__(self, string, ...):
            self.rule = string
            self.endpoint = endpoint
            ...
            
    werkzeug._compat.py
    def implements_to_string(cls):
        cls.__unicode__ = cls.__str__
        cls.__str__ = lambda x: x.__unicode__().encode('utf-8')
        return cls
        
***

    werkzeug.routing.py
    class Map(object):
        def add(self, rulefactory):
            for rule in rulefactory.get_rules(self):
                rule.bind(self)
                self._rules.append(rule)
                self._rules_by_endpoint.setdefault(rule.endpoint, []).append(rule)
            self._remap = True
            
<strong>rule用来存放单条路由的规则，map用来存放所有的rule的集合。</strong>

<span id="jump">
<p>以下为在构造请求上下文的时候从request获取到url_rule的过程：

    app.py
    def create_url_adapter(self, request):
        return self.url_map.bind_to_environ(request.environ,
                server_name=self.config['SERVER_NAME'])
                
    ctx.py
    class RequestContext(object):
        def __init__(self, app, environ, request=None):
            self.app = app
            self.request = app.request_class(environ)
            self.url_adapter = app.create_url_adapter(self.request)
            self.match_request()
        def match_request(self):
            url_rule, self.request.view_args = \
                self.url_adapter.match(return_rule=True)
            self.request.url_rule = url_rule

<p>接下来要分析的是Flask的请求上下文和应用上下文。
</span>
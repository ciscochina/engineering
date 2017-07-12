---
layout: post
title:  "How to read OpenStack code - pastedeploy"
date:   2017-07-12 08:43:59
author: Mingwei Li
categories: openstack
---

本篇分为以下几个部分

>* paste 是什么
>* 怎样使用paste
>* paste of neutron


## paste 是什么

WSGI 是python 中application 和 web server互通的标准。 我们知道了wsgi 中包括 app, middleware ， server而且middleware可以有很多个。wsgi结构的系统最大的好处就是middleware像积木一样，可以灵活的添加组成不同的功能。

我们上一篇文章中，把2个middleware和一个app组合到了一起，但我们采用的写法是在代码中这样写：

    httpd = make_server('10.79.99.86', 8051, CheckError(AuthToken(check_number)))

把各个模块之间的关系硬编码到了代码中，这很显然不够灵活，假设有一天想重新组织middleware的结构，我们还需要找到相关的代码去修改代码。最好的办法是通过配置文件来组织各个模块之间的关系，通过修改配置文件来影响系统中各个模块的组合与调用。 Paste 就是解决这个问题的。

paste 是python的一个module，通过paste， 你可以把wsgi的模块写入ini风格的配置文件，灵活部署。 

下面我们用paste构建一个小系统，进一步了解

## 怎样使用paste

首先，我们将构建三个wsgi组件， 一个application叫check_number， 该application的功能是check http req中的数字是奇数还是偶数，如果是奇数返回odd否则返回even。 代码如下：

    class CheckNumber(object):
    
        def __call__(self, env, start_response):
            number = int(env.get('PATH_INFO').split('/')[-1])
            response_body = 'even' if number % 2 == 0 else 'odd'
    
            status = '200 OK'
            response_headers = [('Content-Type', 'text/plain'), ('Content-Length', str(len(response_body)))]
            start_response(status, response_headers)
            return [response_body]
    
        @classmethod
        def factory(cls, global_config, **local_config):
            return cls()

__call__是wsgi application的入口，逻辑非常简单，这里不做解释。

**why use factory**
看factory函数的逻辑，其实相当于一个构造函数。 下面两行代码实际上是一样的效果

    CheckNumber()
    CheckNumber.factory()

但为什么还要多加一个factory呢？两个原因

1. paste就这么规定的
2. factory 函数多出自java等静态类型语言的设计模式。对java，c/c++是一种很好的设计模式，但python中其实并不需要。不过很多python程序员有较深的java/c++ OOP背景，所以代码中延续了这种风格

不管怎样，我们知道这是个构造函数即可。 

接下来，我们构造一个middleware，名字叫AuthToken。 该middleware将验证http请求header中的token，如果满足要求则放行request继续传递给application，否则返回错误，代码如下：

    class AuthToken(object):
        def __init__(self, app):
            self.app = app
    
        def __call__(self, env, start_response):
            if env.get('HTTP_TOKEN') == '222':
                return self.app(env, start_response)
            else:
                response_body = 'Auth failed'
                status = '403 forbidden'
                response_headers = [('Content-Type', 'text/plain'), ('Content-Length', str(len(response_body)))]
                start_response(status, response_headers)
                return [response_body]
    
        @classmethod
        def factory(cls, global_config, **local_config):
            def _factory(app):
                return cls(app)
            return _factory

Middleware的特性就是可以调用application，同时自身可以像application一样被调用。所以__init__中接受一个app作为参数，并且在__call__中调用这个参数。因此

    AuthToken(CheckNumber)

的效果是返回一个可调用的object。 在调用该object的__call__方法时，首先会检查token，如果token符合要求则继续调用CheckNumber。同样要注意的是factory跟之前一样，完全可以用构造函数替代。


我们继续构造第三个middleware,叫CheckError。 该middleware 用于检查系统中其它application ， middleware返回的错误异常等信息，并转化成用户友好的信息。 逻辑如下：

	class CheckError(object):
		def __init__(self, app):
			self.app = app

		def __call__(self, env, start_response):
			try:
				return self.app(env, start_response)
			except Exception as e:
				response_body = 'Server is maintaining '
				status = '503 service unavailable now'
				response_headers = [('Content-Type', 'text/plain'), ('Content-Length', str(len(response_body)))]
				start_response(status, response_headers)
				return [response_body]

		@classmethod
		def factory(cls, global_config, **local_config):
			def _factory(app):
				return cls(app)

			return _factory

CheckError 的构造函数接受一个app作为参数并返回一个callable对象。调用该对象，即运行__call__方法时会用try catch去包裹一个对app的调用。如果调用有异常则进入异常处理返回，否则返回app调用

**如果没有paste deploy**，我们需要像下面这样写：

    CatchError(AuthToken(CheckNumber(env, callback)))

通过程序来组织各个wsgi组件。这样非常不方便，一旦想添加或者删除甚至更改各个component的关系，需要修改代码。 通过paste deploy，我们可以利用配置文件，如下：

	[DEFAULT]
	security = high

	[composite: my_pipeline]
	use = call:my_module:pipeline_factory
	auth = check_error auth_token check_number
	noauth = check_error check_number

	[app:check_number]
	paste.app_factory = my_module:CheckNumber.factory

	[filter:auth_token]
	paste.filter_factory = my_module:AuthToken.factory

	[filter:check_error]
	paste.filter_factory = my_module:CheckError.factory

paste 配置文件采用INI风格。每个中括号及其下面跟着的一个区域叫做 section 如:

    [filter:check_error]
    paste.filter_factory = my_module:CheckError.factory

这里filter为section type。paste中有很多种类型。app类型代表wsgi application， filter 代表wsgi middleware，server代表server。其它类型我们后面遇到再说。 check_error 是section的name。下面跟着的是section的变量，paste.filter_factory = my_module:CheckError.factory 告诉paste，可以用my_module模块的CheckError.factory load catch_error filter。 简单的说，这段配置告诉paste 我们有一个catch_error middleware 以及怎样加载这个middleware。 

paste中，每个section(除了[DEFAULT])代表一个wsgi组件，paste 通过配置文件可以加载该组件，在加载组件的同时可以读取该section下面的配置信息。这里的信息是该section对应的wsgi组件独享的。但是有个section除外，即[DEFAULT]，DEFAULT section不是wsgi组件，只是用来存配置。这里的配置是全局的，每个section都可以访问。

auth_token, check_number其实和这里一样，都相当于声明一个wsgi组件并且告诉paste如何加载。不同的是check_number是一个app。

我们重点看一下my_pipeline区域。这里我们用的是composite类型。 回顾wsgi 会发现我们有middle/app/server 但就是没有composite。composite不是wsgi原生的类型，它代表多个组件的组合。比如这里是两个pipeline auth和noauth。paste中pipeline也是一种类型，代表串联多个wsgi组件，比如：auth 相当于 CatchError(AuthToken（CheckNumber）) 而 noauth代表CatchError(CheckNumber)。 

    use = call:my_module:pipeline_factory

这一行表示对应的代码在my_module 模块的pipline_factory里。 表示从这行代码加载composite组件。看一下代码的详细信息：

	def pipeline_factory(loader, global_config, **local_config):

		if global_config.get('security') == 'high':
			pipeline = local_config.get('auth')
		else:
			pipeline = local_config.get('noauth')
		# space-separated list of filter and app names:
		pipeline = pipeline.split()
		filters = [loader.get_filter(n) for n in pipeline[:-1]]
		app = loader.get_app(pipeline[-1])
		filters.reverse()  # apply in reverse order!
		for filter in filters:
			app = filter(app)
		return app

因为配置文件中指定该区域是composite，所以该函数接受3个参数loader， global_config 和 local_config。这是paste语法规则指定的。
loader用于加载app，而global_config是配置文件中全局区的配置{security:high}。local_config是 {auth:"catch_error auth_token check_number"}和{noauth:"catch_error auth_token check_number"}。

		if global_config.get('security') == 'high':
			pipeline = local_config.get('auth')
		else:
			pipeline = local_config.get('noauth')

这段的逻辑是，如果全局配置中指定了security为high， 则pipeline中包含auth否则不包含。

		pipeline = pipeline.split()
		filters = [loader.get_filter(n) for n in pipeline[:-1]]

这段的逻辑是从pipeline中找到filter的列表（pipeline最后一项是app，其余都是filter）。


		app = loader.get_app(pipeline[-1])
		filters.reverse()  # apply in reverse order!
		for filter in filters:
			app = filter(app)

这段的逻辑是，获取app，并且把filter列表中的中间件反过来逐层包裹，达到

    CatchError(AuthToken(CheckNumber))

的效果。


接下来我们看一下如何使用。 我们把3个wsgi组件和这个pipeline_factory 放入config.ini。 然后运行如下代码：

	if __name__ == '__main__':
		from paste import httpserver
		from paste.deploy import loadapp
		httpserver.serve(loadapp('config:conf.ini', name='my_pipeline', relative_to='.'), host='127.0.0.1', port='8080')

现在通过下面的命令

    curl http://127.0.0.1/check_number/100 

访问会返回noauth错误，因为你没有加上 -H “token:222” 的验证信息。 但设置security=False重启服务器，继续访问，则不用加验证信息，因为根据composite section对应的代码逻辑，AuthToken组件并没有加到pipeline


## paste of neutron

neutron的paste配置文件在

	cat /usr/share/neutron/api-paste.ini 
	[composite:neutron]
	use = egg:Paste#urlmap
	/: neutronversions
	/v2.0: neutronapi_v2_0

	[composite:neutronapi_v2_0]
	use = call:neutron.auth:pipeline_factory
	noauth = request_id catch_errors extensions neutronapiapp_v2_0
	keystone = request_id catch_errors authtoken keystonecontext extensions neutronapiapp_v2_0

	[filter:request_id]
	paste.filter_factory = oslo_middleware:RequestId.factory

	[filter:catch_errors]
	paste.filter_factory = oslo_middleware:CatchErrors.factory

	[filter:keystonecontext]
	paste.filter_factory = neutron.auth:NeutronKeystoneContext.factory

	[filter:authtoken]
	paste.filter_factory = keystonemiddleware.auth_token:filter_factory

	[filter:extensions]
	paste.filter_factory = neutron.api.extensions:plugin_aware_extension_middleware_factory

	[app:neutronversions]
	paste.app_factory = neutron.api.versions:Versions.factory

	[app:neutronapiapp_v2_0]
	paste.app_factory = neutron.api.v2.router:APIRouter.factory

整个neutron api server就是一个wsgi server 加一堆 wsgi middleware 和 wsgi app组成的一个大的pipeline。 启动加载该配置文件是neutron 的wsgi server负责的事情。这部分我们后面再看。我们先看该配置文件。同样，从下向上看发现有两个APP neutronversions和neutronapiapp_v2_0 以及5个middleware。 在composite区域中又组合了两个pipeline。一个不带验证，一个带有keystone验证。最后composite部分通过paste包中的urlmap函数指定，如果URL是/则访问neutronversion中间件，如果是/v2.0则访问composite部分的pipeline，而这也正是我们真正API访问所走的路径。

大部分环境中所采用的pipeline是keystone这条。因为我们通过devstack或者官方文档安装的环境一般都会带有权限校验。 对于我们来说，真正需要关注的代码在extensions和neutron_app_v2_0这两部分，其他部分都和业务无关。 由配置文件可以很容易找到各个组件的代码位置，如neutron.api.v2.router代表 neutron/api/v2/router。从这里开始，就可以逐步阅读代码追踪每个API的调用流程。
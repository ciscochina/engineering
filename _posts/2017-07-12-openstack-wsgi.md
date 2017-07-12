---
layout: post
title:  "How to read OpenStack code - wsgi"
date:   2017-07-12 08:43:59
author: Mingwei Li
categories: openstack
---

要读懂本篇，你至少得写过一个python的web程序，并且把它部署到web服务器上过。

## 什么是wsgi

假设你写了一个python的web程序，并部署到了nginx上，那么你应该知道一个http request的处理流程一般是下面这样：

    client/浏览器（发送请求）  - - - - > web服务器(转发该请求) - - - - > 你的程序（1. 处理请求。2.生成结果。3.返回结果）
                                                                                                          |
                                                                                                          |
                                                                                                          V
                                   client/浏览器（收到结果） <- - - -web服务器（接受你的程序返回的结果，并返回给浏览器）

问题是，开发web程序的人和开发web服务器的人并没有沟通，为什么web application 能够部署在web 服务器上呢？ 这是因为他们都遵循了相同的规则 -- WSGI。 WSGI 全名叫 web server gateway interface， 是python 中web程序和web服务器沟通的标准。 简单的说，只要你写的web application 遵守这个规则，另一个人开发的web 服务器也遵循这个规则，那么你写的程序就能运行在他开发的web服务器上。

## wsgi application

WSGI对application和server做出了不同的规定，大部分人不需要写server,因此我们只了解一下application的规定即可。首先看一个标准的WSGI application。该application的功能是，返回client所使用的http method(GET/POST等)：

	def application(env, callback):
		"""
		web server 实际上只做两件事
		1. 传递请求给application
		2. 接受application的返回
		env 就是server 给 application 传递请求的方式。 env是一个字典，除了客户端的请求信息，还包括很多其他的环境变量如http header等
		callback 是server 传递给application 的callback函数。 application 通过调用该函数，返回一些信息给server
		"""

		# EVN中包含了client 的各种环境变量，从中可以获取http method
		response_body = 'The request method was %s' % env['REQUEST_METHOD']

		# HTTP response的结果通常会有一个状态值和message。
		# 比如， 状态有500/404/403/200等， message 有OK/Not Found/Internal error/Forbidden 等。
		# 这里我们返回200 OK. 代表成功处理请求
		status = '200 OK'

		# HTTP response 的header 会包含一些必要的信息以便浏览器方便处理response。 这些必要的header信息需要按照以下格式放入list中
		response_headers = [('Content-Type', 'text/plain'), ('Content-Length', str(len(response_body)))]

		# 利用callback 告诉server这次访问的状态信息以及 response header。
		callback(status, response_headers)

		# response 的body必须是一个iterable的对象。 我们这里把response body 放入一个list。（string虽然也是可迭代对象，但string的迭代次数显然远远大于这个list。会影响效率）
		return [response_body]


总结上面的代码，可知道WSGI 对application 只做了如下几点规定：

    1. application 必须是可调用的object, 如函数,实现了__call__的对象

    2. application 必须接收一个字典类型的参数用于保存环境变量，一个callback function

    3. application 内要使用callback 返回 http status 和 response header

    4. 返回值必须是iterable的


我们用一个简单的wsgi server来调用一下上面的 application， 看一下效果。

    from wsgiref.simple_server import make_server
    httpd = make_server('localhost', 8051, application)
    httpd.serve_forever()

上面代码启动了一个wsgi server监听在localhost:8051， 并且把我们的application 部署到了该server上。 我们访问localhost 8051,看一下效果

        [root@netflow-AIO ~]# curl -i http://127.0.0.1:8051
        HTTP/1.0 200 OK
        Date: Sat, 26 Nov 2016 07:34:38 GMT
        Server: WSGIServer/0.1 Python/2.7.5
        Content-Type: text/plain
        Content-Length: 26

        The request method was GET

可以看到， application 成功返回了代码中指定的response message，并且http code / http msg / response header 也都和代码中指定的一样。

## wsgi server

大多数情况我们不需要编写server，因此不必了解太深。 但至少我们应该知道，在server中一定有这样的代码

    callable(env, callback)

这里是server 调用application的地方。callable 是application对象的名字， env 是包含环境变量的字典，callback 是server传递给application的callback函数。想在applicaiton 被调用之前加一些逻辑，你可以在这行代码之前做改动。想修改application的返回结果，你可以在这行代码之后加逻辑

## wsgi 中间件

WSGI 标准中除了application / server 还有一个很重要的概念--middleware。 其实 middleware 很好理解。 我们知道 server 会调用 application 处理请求， application 会把请求结果返回给server. Middleware，顾名思义就是在server 和 application 中间的一个对象。  对于server 来说 middleware 是一个application, 对于 application 来说， middleware 是一个server。也就是说,middleware同时实现了WSGI中对 application 和 server所做的规定。

因为WSGI的这一特性，很多按照wsgi构建的系统都会有如下这样的结构

    app1 app2 app3 ... appN server

当一个http request发给server 时， 该请求会依次传递给 appN ... app3 app2 app1, 然后response 又会从app1 开始往回传递直到 server 最后返回给客户端。也就是说，WSGI的系统是可以无限堆叠中间件的，你可以把自己的业务逻辑包装成一个个的中间件，需要的时候部署上去，不需要就拿下来。 openstack中所有的服务都是依照wsgi写的，因此也遵循这种结构。 比如neutron 服务的结构：

    request_id catch_errors authtoken keystonecontext extensions neutronapiapp_v2_0

其中 catch_errors , authtoken, keystonecontext , extensions 都是WSGI的中间件。 了解了这些之后，我们可以做很多事情， 比如拿掉 authoken 和 keystonecontext, 这样调用neutron 的API就可以绕过权限校验了，因为这两个中间件是做权限校验用的。

## App 中间件 server示例

接下来我们用一个例子展示一下 wsgi application middleware server 到底是什么样的。

首先我们设计一个app，假设带着一个参数访问该app，它将告诉你该参数是奇数还是偶数。 api的url 假设是 check_number/<number>, 代码如下：

    def check_number(env, start_response):
    
        number = int(env.get('PATH_INFO').split('/')[-1])
        response_body = 'even' if number%2 == 0 else 'odd'
    
        status = '200 OK'
        response_headers = [('Content-Type', 'text/plain'), ('Content-Length', str(len(response_body)))]
        start_response(status, response_headers)
        return [response_body]

代码非常简单，因为我们假设url为check_number/<nunber>， 所以int(env.get('PATH_INFO').split('/')[-1])可以很轻松得到参数。这里我们不考虑异常。

接下来，我们设计一个middleware， 用于校验， 该middleware会检查http 访问的header， 如果带有特定的token，则认为是可信任用户，可以放行，继续访问check_number， 否则返回403 Forbidden。 代码如下：

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

这里稍微解释一下。 所谓中间件，肯定要能被server 调用， 所以必须提供__call__(env, start_response)接口，并且返回可迭代的return 如[response_body]。 但同时它又需要调用别的app。
看我们AuthToken的代码，可以看到，它的逻辑就是，如果token合法（等于222） ，则继续调用application， 否则返回错误。

有了验证中间件AuthToken, 有了业务逻辑check_number， 我们还可以再加一个中间件 CheckError。 该中间件捕捉所有的error / exception， 返回一个用户友好的消息，如 service maintain 等， 而不是直接把异常抛给用户。 代码如下：

    class CheckError(object):
        def __init__(self, app):
            self.app = app
    
        def __call__(self, env, start_response):
            try:
                return self.app(env, start_response)
            except Exception as e:
                response_body = 'Server is maintaining '
                status = '503  service maintain'
                response_headers = [('Content-Type', 'text/plain'), ('Content-Length', str(len(response_body)))]
                start_response(status, response_headers)
                return [response_body]

逻辑很简单，如果有错误，则返回  service maintain， 否则返回正常的业务处理结果。

接下来可以部署这些程序到服务器

    from wsgiref.simple_server import make_server
    httpd = make_server('10.79.99.86', 8051, CheckError(AuthToken(check_number)))
    httpd.serve_forever()

尝试访问：

	[root@netflow-AIO bin]# curl http://10.79.99.86:8051/check_number/100 -H "token:222"
	even
	
	[root@netflow-AIO bin]# curl http://10.79.99.86:8051/check_number/100 -H "token:wrong"
	Auth failed

	[root@netflow-AIO bin]# curl http://10.79.99.86:8051/check_number/aaa -H "token:222"
	Server is maintaining 

后面可以看到，网上对这种wsgi模块堆叠的部署方式叫pipeline。 但其实它和linux的pipeline不太一样， 其实是一种堆栈式的调用，调用流程图如下：


    request ----> server 生成 env 和 start_response
                ----> Check_Error.__call__
                    ---->AuthToken.__call__
                        ---->Check_Number(env,start_response)
                        <----Check_Number 返回
                    <----AuthToken 返回
                <----Check_Error 返回
    response <----
                            

## WSGI in openstack

前面提到，写一个wsgi application 有若干规则要遵守，比如提供两个参数 env 和 callback ， 再比如return 一个可迭代对象。 这些规则其实跟业务逻辑无关。作为一个程序员，你可能更关心业务逻辑而不是wsgi的语法。你可能更希望写一个下面这样的程序

    def application(req):
        # some logic
        return response

req 是客户的http 请求， response 是程序进行处理后的返回。这样对程序员来说就简单多了，只需要关心业务逻辑而不用关心wsgi稍显繁琐的语法。openstack中就实现了这样的简化，在openstack中，实现一个wsgi 程序可以这样写

    import webob
    import webob.dec
    from webob.response import Response

    @webob.dec.wsgify
    def myapp(req):
        return Response(body=req.url)

    from wsgiref.simple_server import make_server
    httpd = make_server('localhost', 8051, myapp)
    httpd.serve_forever()


webob是用于web编程的一个model，通过它的wsgify这个装饰器，你可以很方便的把一个 接受 request 参数，返回response参数的函数转成wsgi application。现在你可以不必关心wsgi的细节，只要实现一个接受request 返回 response的函数就可以了。 默认情况下， req 是webob.request.Request类型的对象。

## sumary

以上简单介绍了wsgi， 如果想对wsgi了解更多，可以参考pep3333
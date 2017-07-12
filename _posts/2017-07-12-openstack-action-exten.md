---
layout: post
title:  "How to read OpenStack code part 8 - action extension"
date:   2017-07-12 08:43:59
author: Mingwei Li
categories: openstack
---

之前我们看过了core plugin， service plugin 还有resource extension。

resource extension的作用是定义新的资源。而我们说过还有两种extension： action extension 跟 request extension。这一章我们将写一个action extension。

# action in RESTful

openstack中所有的网络服务都是RESTful风格的。但RESTful风格的URL有一个问题，如何表示动作。

像 network，tiger 这些单词都是名词，用在URL中以HTTP的POST/GET/DELETE/PUT来对应create get delete update没有任何问题。但有的时候我们需要在url中表示一些动词。比如，router是一个名词，但router中添加一个端口是一个动词。这时候我们有三种选择：

**把增加端口变成update router的一个分支功能**

这是一个好办法，但很多时候由于种种原因并不适用，比如这需要修改原来的update逻辑，需要修改原来的router数据结构，可能这个动作及其复杂，把它放入update不合适等等等等

**把这个动作变成一个独立的资源**

比如原来的url是api/v2/routers, 但现在我们用下面的URL表示这个动作

    api/v2/routers/{router id}/add_router_interface

这个方法也很不错。事实上，neutron中大多数情况下都是用的这个方法。不过，如果这么做的话，动作变成了资源，那么就不用action extension而用resource extension了。所以这种方法不是我们这里要讨论的

**用一个action资源来处理所有的动作**

比如原来的url是api/v2/routers,现在我们用下面的url

    api/v2/routers/action POST

既然是POST 那么肯定有body， body可能是

    '{"add_router_interface"："some value"}'

这样就可以通过一个action url处理所有的动作请求了。这正是neutron action extension所做的事。

**不过，奇妙的是，搜索遍了neutron的代码，没有看到一个功能是用这种extension来实现的。也许neutron正在努力用第二种方式替代第三种方式。**

# write action extension

虽然neutron中没有用这种extension，但它还是支持的。并且我们也很有必要了解这种extension，因为在neutron的启动加载过程中会尝试处理它，如果我们不了解，那么看到了这种代码会很懵

之前我们写过一个zoo plugin和zoo extension。并且实现了一个tiger资源。这里我们写一个action extension来为tiger增加一个动作:eat_wolf。

要实现action extension非常简单，只需要提供get_actions函数如下

	from neutron.api import extensions
	from neutron.api.v2 import attributes
	from neutron.api.v2 import base
	from neutron import manager
	from neutron.api.v2 import resource_helper
	from neutron.plugins.common import constants

	EXT_PREFIX = '/zoo'
	RESOURCE_NAME = 'tiger'
	COLLECTION_NAME = '%ss' % RESOURCE_NAME

	RESOURCE_ATTRIBUTE_MAP = {
		'tiger': {
			'id': {'allow_post': False, 'allow_put': False,
				   'validate': {'type:uuid': None},
				   'is_visible': True,
				   'primary_key': True},
			'name': {'allow_post': True,
					 'allow_put': False,
					 'is_visible': True,
					 'default': ''},
		# tenant_id is the user id used by keystone for authorisation
		# It's good to use the following as it is and it is necessary
		# for every extension
		'tenant_id': {'allow_post': True, 'allow_put': False,
					  'required_by_policy': True,
					  'validate': {'type:string': None},
					  'is_visible': True}

		}
	}


	class Zoo(extensions.ExtensionDescriptor):
		#   path_prefix = "zoo"
		@classmethod
		def get_name(cls):
			return "zoo"

		@classmethod
		def get_alias(cls):
			return 'zoo'

		@classmethod
		def get_description(cls):
			return "zoo"

		@classmethod
		def get_updated(cls):
			return "2017-02-08T10:00:00-00:00"

		@classmethod
		def get_resources(cls):
			# This method registers the URL and the dictionary  of
			# attributes on the neutron-server.
			exts = list()
			plugin = manager.NeutronManager.get_service_plugins()['ZOO']
			resource_name = RESOURCE_NAME
			collection_name = COLLECTION_NAME
			params = RESOURCE_ATTRIBUTE_MAP.get(resource_name)
			controller = base.create_resource(collection_name, resource_name,
											  plugin, params, allow_bulk=False)
			ex = extensions.ResourceExtension(collection_name, controller, path_prefix=EXT_PREFIX)
			exts.append(ex)
			return exts

		def get_actions(self):
			eat_wolf_action = extensions.ActionExtension('tigers', 'eat_wolf', self._eat_wolf_handler)
			return [eat_wolf_action]
			
		def _eat_wolf_handler(self, *args, **kwargs):
			return 'The tiger eat a wolf'

我们这里贴出了所有的代码。但实际上只需要 get_action函数和 _eat_wolf_handler函数。实现了第一个函数就是action extension，第二个函数只是为了处理action的逻辑。重启neutron server后我们访问一下

	curl -g -i -X POST http://liberty-controller01:9696/v2.0/tigers/40/action -H "Content-Type: application/json" -H "Accept: application/json" -H "X-Auth-Token: $token" -d '{"eat_wolf": "0730d3b8-9f1a-47e4-8d8b-365aef954160"}'
	HTTP/1.1 200 OK
	Content-Type: text/html; charset=UTF-8
	Content-Length: 20
	X-Openstack-Request-Id: req-d28c6e78-d8da-49fe-8fe8-deaac591a076
	Date: Thu, 09 Feb 2017 09:58:36 GMT

	The tiger eat a wolf

OK， 上面就是我们的action extension了
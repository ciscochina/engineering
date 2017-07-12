---
layout: post
title:  "How to read OpenStack code part 6 - core plugin and extension"
date:   2017-07-12 08:43:59
author: Mingwei Li
categories: openstack
---

本章我们将写一个自己的core plugin 和一个resource extension来加深理解。(阅读本文的前提是你已经理解了restful以及stevedore等内容)

# 什么是 core plugin

neutron的plugin有core和service两种。core plugin实现core resource的增删改查，service plugin我们在本文暂不讨论。

core resource 有network/subnet/port/subnet-pool。每种资源对应CURD和index5种操作。以network资源为例，相关操作和HTTP请求的对应关系如下

    create_network    POST networks
    delete_network    DELETE networks/network_id
    update_network    PUT networks/network_id
    get_network       GET networks/network_id
    get_networks      GET networks

因此，core plugin中对各种核心资源都有如下的函数对应：

    create_{resource_name}
    update_{resource_name}
    delete_{resource_name}
    get_{resource_name}
    get_{resource_names}

# how to write core plugin

既然是plugin，也就是我们前面说过的third-party code，那就应该定义一些接口，好让第三方的组织可以参与开发。

neutron中定义的接口在neutron.neutron_plugin_base_v2.NeutronPluginBaseV2。要实现一个core plugin，你需要实现这个class。


下面是我写的一个core plugin 代码

	from neutron import neutron_plugin_base_v2
	from oslo_log import log
	LOG = log.getLogger(__name__)


	class MyNeutronPlugin(neutron_plugin_base_v2.NeutronPluginBaseV2):
		supported_extension_aliases = ['gold']
		def __init__(self):
			super(MyNeutronPlugin,self).__init__()
		#### network ####
		def create_network(self, context, network):
			LOG.info("network is %s" %network)
			return network
		def update_network(self, context, id, network):
			return network
		def get_network(self, context, id, fields=None):
			network = {}
			return network
		def get_networks(self, context, filters=None, fields=None):
			network = {}
			LOG.info("return network %s" %network)
			return network
		def delete_network(self, context, id):
			return id
                ...
                #### gold ####
		def create_gold(self, context, gold):
			LOG.info("gold is %s" %gold)
			return gold
		def update_gold(self, context, id, gold):
			return gold
		def get_gold(self, context, id, fields=None):
			gold = {}
			return gold
		def get_golds(self, context, filters=None, fields=None):
			gold = {}
			LOG.info("return gold %s" %gold)
			return gold
		def delete_gold(self, context, id):
			return id

请注意，除了network等核心资源，我们还实现了一个gold资源。 上面粘贴的内容省略了subnet/port/subnet-pool的相关代码。 既然是thirty-party code,就应该可以独立安装。所以我们的代码结构如下：

	myPlugin/
	├── myPluginPKG
	│   ├── __init__.py
	│   └── myPluginModule.py
	└── setup.py

__init__.py的内容为

    import myPluginModule

setup.py内容如下：

	from setuptools import setup, find_packages

	setup(
		name='myPluginPKG',
		version='1.0',

		packages=find_packages(),

		entry_points={
			'neutron.core_plugins': [
				'myNeutronPlugin = myPluginPKG.myPluginModule:MyNeutronPlugin',
			],
		},
	)

这样在运行python setup.py install 后，我们的plugin就注册到了neutron.core_plugins这个namespace下。这部分内容其实就是前面stevedore中driver开发的内容。neutron通过stevedore在这个namespace下加载core_plugin

OK。在python setup.py install 后，我们的core plugin安装完成了，这时要修改/etc/neutron/neutron.conf，让neutron使用我们的core plugin。

    [default]
    core_plugin = myNeutronPlugin

然后重启neutron服务 

    systemctl restart neutron-server

接下来你就可以通过API 尝试了， 我们创建一个network 如下：

	curl -g -i -X POST "http://liberty-controller01:9696/v2.0/networks" -H "Content-Type: application/json" -H "Accept: application/json" -H "X-Auth-Token:$token" -d '{"network": {"name": "n2", "admin_state_up": true}}'
	HTTP/1.1 201 Created
	Content-Type: application/json; charset=UTF-8
	Content-Length: 15
	X-Openstack-Request-Id: req-a8cfde05-6425-42fe-b516-0bb2d264cf61
	Date: Tue, 07 Feb 2017 08:01:47 GMT

        {"network": {}}

可以看到成功返回，因为我们的plugin中什么也没做，所以返回的其实是一个空的字典。但至少证明该api成功了。


之前我们还在plugin中增加了gold资源对应的函数，我们试试访问gold资源：

    curl -g -i  "http://liberty-controller01:9696/v2.0/golds" -H "Content-Type: application/json" -H "Accept: application/json" -H "X-Auth-Token:$token" 
    HTTP/1.1 404 NOT FOUND

为什么gold资源的API不好使呢。这是因为要想在neutron core plugin中定义一个资源，不仅要提供该资源的CURD 函数，还要有对应的RESOURCE_ATTRIBUTE_MAP. 


# RESOURCE_ATTRIBUTE_MAP

RESOURCE_ATTRIBUTE_MAP是neutron/api/v2/attribute.py中的一个字典，其结构大概如下：

	RESOURCE_ATTRIBUTE_MAP = {
		NETWORKS: {
			'id': {'allow_post': False, 'allow_put': False,
				   'validate': {'type:uuid': None},
				   'is_visible': True,
				   'primary_key': True},
			'name': {'allow_post': True, 'allow_put': True,
					 'validate': {'type:string': NAME_MAX_LEN},
					 'default': '', 'is_visible': True},
			'subnets': {'allow_post': False, 'allow_put': False,
						'default': [],
						'is_visible': True},
			'admin_state_up': {'allow_post': True, 'allow_put': True,
							   'default': True,
							   'convert_to': convert_to_boolean,
							   'is_visible': True},
			'status': {'allow_post': False, 'allow_put': False,
					   'is_visible': True},
			'tenant_id': {'allow_post': True, 'allow_put': False,
						  'validate': {'type:string': TENANT_ID_MAX_LEN},
						  'required_by_policy': True,
						  'is_visible': True},
			SHARED: {'allow_post': True,
					 'allow_put': True,
					 'default': False,
					 'convert_to': convert_to_boolean,
					 'is_visible': True,
					 'required_by_policy': True,
					 'enforce_policy': True},
		},

该字典定义了资源以及资源对应的属性，我们这里只列出了network。所以，要想定义gold资源，除了在core plugin中添加对应的函数，还有在这个attribute map中添加相关的信息。不过，我们不建议直接修改这里的代码，正确的方法是通过resource extension来实现。


# how to code an extension (resource extension)

之前提到了extension有3种类型，resource，action，request。我们这里要实现一个新的资源，就是用resource extension。无论是哪种extension 都要遵守一些特定的规则，或者说接口，这些规则如下：

    1. extension应该放在neutron/extensions文件夹下，或者在配置文件中设置api_extensions_path
    2. extension的class名应该和文件同名，当然首字母应该大写
    3. 应该实现neutron.api.extensions.py中ExtensionDescriptor定义的接口
    4. 在对应的plugin的supported_extension_aliases 中增加我们extension的别名。前面我们写的core plugin就有这个属性，当时没有做说明，其实是这里应该添加的。所谓别名是该extension要实现的一个接口，后面会看到。

另外很重要的一点是，因为我们要实现的是resource extension，所以还要实现

    get_resources 

这个接口。 

下面我们实现一个自己的resource extension来增加 gold 资源。

	from neutron.api import extensions
	from neutron import manager
	from neutron.api.v2 import base

	# You have to specify the attributes neutron-server should expect when
	# someone invokes this plugin. Let's say you want
	# 'name', 'priority', 'credential' for your extension /golds
	# then following dictionary must be declared.
	# I am following the naming convention used by other extensions.

	RESOURCE_ATTRIBUTE_MAP = {
		'golds': {
		'name': {'allow_post': True, 'allow_put': True,
					 'is_visible': True},
		'priority': {'allow_post': True, 'allow_put': True,
					 'is_visible': True},
		'credential': {'allow_post': True, 'allow_put': True,
					 'is_visible': True},
		# tenant_id is the user id used by keystone for authorisation
		# It's good to use the following as it is and it is necessary
		# for every extension 
		'tenant_id': {'allow_post': True, 'allow_put': False,
					  'required_by_policy': True,
					  'validate': {'type:string': None},
					  'is_visible': True}
		}
	}

	# Great! Now you have the defined the attributes that you need for your
	# extensions. You need to store this dictionary in the neutron-server
	# by the following class

	class Golds(extensions.ExtensionDescriptor):
		# The name of this class should be the same as the file name
		# There are a couple of methods and their properties defined in the
		# parent class of this class, ExtensionDescriptor you can check them

		@classmethod
		def get_name(cls):
			# You can coin a name for this extension
			return "Name of golds"

		@classmethod
		def get_alias(cls):
			# This alias will be used by your core_plugin class to load
			# the extension
			return "gold"

		@classmethod
		def get_description(cls):
			# A small description about this extension
			return "A quick brown fox jumped over a lazy dog"

		@classmethod
		def get_namespace(cls):
			# The XML namespace for this extension
			# but as we move on to use JSON over XML based request
			# this is not that important, correct me if I am wrong.
			return "namespace of xml"

		@classmethod
		def get_updated(cls):
			# Specify when was this extension last updated,
			# good for management when there are changes in the design
			return "2017-02-07T10:00:00-00:00"

		@classmethod
		def get_resources(cls):
			# This method registers the URL and the dictionary  of
			# attributes on the neutron-server.
			exts = list()
			plugin = manager.NeutronManager.get_plugin()
			resource_name = 'gold'
			collection_name = resource_name + 's'
			params = RESOURCE_ATTRIBUTE_MAP.get(resource_name + 's', dict())
			controller = base.create_resource(collection_name, resource_name,
											  plugin, params, allow_bulk=False)
			ex = extensions.ResourceExtension(collection_name, controller)
			exts.append(ex)

			return exts


代码的大部分解释都包含在注释里，因此请详细阅读每一行。这里只重点说一下RESOURCE_ATTRIBUTE_MAP 和 get_resources。 我们的extension就是通过这个函数来在resource attribute map中增加了gold资源。
另外要注意的是get_alias，该函数返回extension的别名，plugin的supported_extension_alias中用该别名来找extension。

ok，我们现在尝试一下访问gold 资源

	curl -g -i http://liberty-controller01:9696/v2.0/golds.json -H "Content-Type: application/json" -H "Accept: application/json" -H "X-Auth-Token:$token"
	HTTP/1.1 200 OK
	Content-Type: application/json; charset=UTF-8
	Content-Length: 13
	X-Openstack-Request-Id: req-7b925f97-4df0-49ee-bb6a-fd7c86fda9aa
	Date: Tue, 07 Feb 2017 08:29:34 GMT

	{"golds": []}

这次成功了。 以上就是core plugin和resource extension
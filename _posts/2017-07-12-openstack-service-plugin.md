---
layout: post
title:  "How to read OpenStack code part 7 - service plugin"
date:   2017-07-12 08:43:59
author: Mingwei Li
categories: openstack
---

We have learned core plugin, service plugin and extension in last post. Now let`s review:

**Core Plugin**

Core plugin manage core resources which are network, subnet, port and subnetpool. 

**Service Plugin**

Service plugin manage higher services.

**extension**

Extensions are called API Extensions. There are three types of extension

>* resource extension which define new resources 
>* action extension which define actions for resource
>* request extension which can add more parameter to request

Normally a new feature will be implemented by extension first. When the feature is stable, the community will move it to official api and may implement it in plugin.

We write our core plugin in the previous post now we are going to write our service plugin.

# What is the difference between core plugin and service plugin

Core plugin manage core resource in neutron. The code structure is different from service plugin. But the community are considering transfer core plugin into one kind of service plugin. You will see the trend in code

# Design our service plugin

Our service plugin is called "ZOO". This plugin will manage some resource like "tiger". We are going to do API call like CREATE/UPDATE/DELETE/GET tiger with this service plugin.

# Write our service plugin

Service plugin must be inherited from the class neutron.services.service_base.ServicePluginBase

Below are my service plugin

	from neutron.services import service_base


	class ZooPlugin(service_base.ServicePluginBase):
		supported_extension_aliases = ["zoo"]

		def __init__(self):
			super(ZooPlugin, self).__init__()

		def get_plugin_name(self):
			return "ZOO"

		def get_plugin_type(self):
			# should be under neutron/plugins/common/constants.py
			return "ZOO"

		def get_plugin_description(self):
			return ("ZOO")

		def create_tiger(self, context, tiger):
			return "tiger created"

		def get_tigers(self, context, filters, fields):
			return {}

supported_extension_aliases is necessary since we need an extension to generate the resource.

We only support get_tigers and create_tiger here for simplicity purpose.

Because the plugin is third-party code, so we have to register it under certain entry point so neutron can load it. So our code structure will be like :

	[root@liberty-controller01 tmp]# tree zooServicePlugin
	zooServicePlugin/
	├── setup.py
	└── zoo
		├── __init__.py
		├── zoo_plugin.py

The content of setup.py is like

	from setuptools import setup, find_packages

	setup(
		name='zoo',
		version='1.0',

		packages=find_packages(),

		entry_points={
			'neutron.service_plugins': [
				'zoo = zoo.zoo_plugin:ZooPlugin',
			],
		},
	)

The key point here is to register the plugin under neutron.service_plugins namespace.



# Write the extension

We have service plugin ready to manage the tiger resource. But we do not have the tiger resource yet. One option is to modify the neutron/api/v2/attribute.py which is not suggested. The recommended way is to generate the resource by extension like below

    from neutron.api import extensions
    from neutron.api.v2 import base
    from neutron import manager
    
    
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
            'tenant_id': {'allow_post': True, 'allow_put': False,
                          'required_by_policy': True,
                          'validate': {'type:string': None},
                          'is_visible': True}
        }
    }
    
    
    class Zoo(extensions.ExtensionDescriptor):
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


The RESOURCE_ATTRIBUTE_MAP is used for define resource tiger. The tenant_id attribute is necessary for auth.

An extension must inherited from extensions.ExtensionDescriptor

get_alias method is really important because plugin will use this value to find the extension. This value must be in the supported_extension_alias of plugin

The get_resources method is necessary for an extension who define new resource. We will see the detail in later post.


# Config

Now we have our service plugin and extension we need to install our service plugin by python setup.py install and put the extension under neutron/extensions

Also config the /etc/neutron/neutron.conf

    service_plugins = ...,zoo

Restart your neutron server and run below API

	curl -g -i  "http://liberty-controller01:9696/v2.0/zoo/tigers" -H "Content-Type: application/json" -H "Accept: application/json" -H "X-Auth-Token:$token"
	HTTP/1.1 200 OK
	Content-Type: application/json; charset=UTF-8
	Content-Length: 14
	X-Openstack-Request-Id: req-82ed8ccc-da9d-46d9-8fd9-beb01a24385b
	Date: Wed, 08 Feb 2017 11:09:13 GMT

	{"tigers": []}

It work
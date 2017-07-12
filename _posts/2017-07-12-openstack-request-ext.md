---
layout: post
title:  "How to read OpenStack code part 9 - request extension"
date:   2017-07-12 08:43:59
author: Mingwei Li
categories: openstack
---

We have learned resource extension and action extension. This post we will write a request extension

First see two API call


	curl -X POST http://liberty-controller01:9696/v2.0/networks.json -H "Content-Type: application/json" -H "Accept: application/json" -H "X-Auth-Token: $token" -d '{"network": {"name": "net3", "admin_state_up": true}}'

	{"network": {"status": "ACTIVE", "subnets": [], "name": "net3", "provider:physical_network": "physnet1", "admin_state_up": true, "tenant_id": "8c5f13ee6a404759839e48537bdf69ac", "mtu": 0, "router:external": false, "shared": false, "port_security_enabled": true, "provider:network_type": "vlan", "id": "95c955ac-c963-4f53-ab4a-d721fa0cda51", "provider:segmentation_id": 490}}


It run successful. Nothing to say. Then this one

	curl -X POST http://liberty-controller01:9696/v2.0/networks.json -H "Content-Type: application/json" -H "Accept: application/json" -H "X-Auth-Token: $token" -d '{"network": {"name": "net3", "admin_state_up": true, "some_attr":"some_value"}}'

	{"NeutronError": {"message": "Unrecognized attribute(s) 'some_attr'", "type": "HTTPBadRequest", "detail": ""}}[root@liberty-controller01 myPluginPKG]# 



This one is bad request. But the only difference is the POST body. Neutron think the some_attr is not a attribute of network and it is right. Because this is a core resource and the attribute map of this resource do not have some_attr

To solve this we need an request extension which actually update the resource attribute map of network. Below are the code

    from neutron.api import extensions
    
    EXTENDED_ATTRIBUTES_2_0 = {
        'networks': {
            'some_attr': {'allow_post': True,
                     'allow_put': False,
                     'is_visible': True,
                     'default': ''}
        }
    }
    
    
    class Myreq(extensions.ExtensionDescriptor):
        @classmethod
        def get_name(cls):
            return "myreq"
    
        @classmethod
        def get_alias(cls):
            return 'myreq'
    
        @classmethod
        def get_description(cls):
            return "myreq"
    
        @classmethod
        def get_updated(cls):
            return "2017-02-08T10:00:00-00:00"
    
        def get_extended_resources(self, *args, **kwargs):
            return EXTENDED_ATTRIBUTES_2_0

You can see we defined a dict in the extension and return it with method get_extended_resources.

So to implement an request extension is very easy. Define a method called get_extended_resources and return some attributes that you want to added to the original reosurce
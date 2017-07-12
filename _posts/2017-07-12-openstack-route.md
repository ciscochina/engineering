---
layout: post
title:  "How to read OpenStack code part 4 - route"
date:   2017-07-12 08:43:59
author: Mingwei Li
categories: openstack
---

When coding a web system, you have to think about an important problem, how to map urls to logic.

Openstack use routes to solve this problem.

# What is routes

Routes is a python package used to map urls to program logic. Normally this is the web framework`s responsibility, so besides openstack, routes is also used in several other python web frameworks.

# Quick start

	>>> from routes import Mapper
	>>> map = Mapper()
	>>> map.connect('route_name_update_net', '/nets/{net_id}', controller='nets_controller', conditions=dict(method=['PUT']), self_defined_key='self_defined_value')
        # Above code created a route named "route_name_update_net"
        # It match any two component urls started with /nets
        # Controller which will process the request is nets_controller
        # Conditions parameter restrict the route only accept PUT request

	>>> map.match('/nets/123')
	{'controller': u'nets_controller', 'net_id': u'123', "self_defined_key":"self_defined_value"}
        # As you can see, the url which match the route will return a dict. The dict contains the controller information and necessary var
        # The self_defined_key and self_defined_value also included. 

	>>> print map
	Route name            Methods Path          
	route_name_update_net PUT     /nets/{net_id}
	>>> 


# Basic usage

## Requirements

Some times you want to restrict the path var, you can use the requirements parameter in below ways

    map.connect(R"/download/{platform:windows|mac}/{filename}")

This line restrict the platform var could only be windows or mac. You can also write it in below way

    map.connect("/download/{platform}/{filename}", requirements={"platform": R"windows|mac"})

The R char here is very important. Without it you may need to double your slash in your url.

Once you restrict your route, you can try it like below:

    # The platform must be windows or mac so below url match
    >>> map.match('/download/windows/myfile')
    {'platform': u'windows', 'filename': u'myfile'}
    # The linux is not valid here so does not match
    >>> map.match('/download/linux/myfile')

## PATH_INFO

In WSIG env, the url information is stored in the environ dict with key PATH_INFO. 
Routes treat the path_info specially. When the “path_info” variable is used at the end of the URL, Routes moves everything preceding it into the “SCRIPT_NAME” environment variable. This is useful when delegating to another WSGI application that does its own routing: the subapplication will route on the remainder of the URL rather than the entire URL

For example

    >>> from routes import Mapper
    >>> map=Mapper()
    >>> map.connect('/classes/{path_info:.*}')
    >>> map.match('/classes/students/tom')
    {'path_info': 'students/tom'}
    >>> 
    
The subsequent system will see PATH_INFO: /students/tom and SCRIPT_NAME:classes. So the sub system can use its own route system.


## Format extensions

Consider to design a url /neutron/networks/{network_id} which will used to access the network resource. The network can be returned in JSON format or XML format. So we can do it in routes like this:

	>>> from routes import Mapper
	>>> map=Mapper()
	>>> 
	>>> map.connect('/neutron/networks/{network_id}{.format:json|xml}')
	>>> map.match('/neutron/networks/123.json')
	{'network_id': u'123', 'format': u'json'}
	>>> map.match('/neutron/networks/123.xml')
	{'network_id': u'123', 'format': u'xml'}
	>>> map.match('/neutron/networks/123.html')
	{'network_id': u'123.html', 'format': None}


Use the format parameter to specify the resource format

# RESTful

Use routes can generate restful routes very conveniently

	>>> from routes import Mapper
	>>> map=Mapper()
	>>> map.resource('net','nets')
	>>> print map
	Route name         Methods Path                      
					   POST    /nets.:(format)           
					   POST    /nets                     
	formatted_nets     GET     /nets.:(format)           
	nets               GET     /nets                     
	formatted_new_net  GET     /nets/new.:(format)       
	new_net            GET     /nets/new                 
					   PUT     /nets/:(id).:(format)     
					   PUT     /nets/:(id)               
					   DELETE  /nets/:(id).:(format)     
					   DELETE  /nets/:(id)               
	formatted_edit_net GET     /nets/:(id)/edit.:(format)
	edit_net           GET     /nets/:(id)/edit          
	formatted_net      GET     /nets/:(id).:(format)     
	net                GET     /nets/:(id)

The first parameter is resource name also called member in RESTful 
The second is the plural of resource name which is also called collection in RESTful

the resource function have several parameters and we will learn them one by one

## controller

This parameter specify the controller to handle the request. For example:

	>>> map=Mapper()
	>>> map.resource('net','nets', controller='net_controller')
	>>> map.match('/nets/123')
	{'action': u'update', 'controller': u'net_controller', 'id': u'123'}

## collection

This parameter add more additional urls for the collection operations. For example:

Before use the parameter the map is like below

	>>> map=Mapper()
	>>> map.resource('net','nets', controller='net_controller')
	>>> print map
	Route name         Methods Path                      
					   POST    /nets.:(format)           
					   POST    /nets                     
	formatted_nets     GET     /nets.:(format)           
	nets               GET     /nets                     
	formatted_new_net  GET     /nets/new.:(format)       
	new_net            GET     /nets/new                 
					   PUT     /nets/:(id).:(format)     
					   PUT     /nets/:(id)               
					   DELETE  /nets/:(id).:(format)     
					   DELETE  /nets/:(id)               
	formatted_edit_net GET     /nets/:(id)/edit.:(format)
	edit_net           GET     /nets/:(id)/edit          
	formatted_net      GET     /nets/:(id).:(format)     
	net                GET     /nets/:(id)               

After we add the collection parameter

	>>> map.resource('net','nets', controller='net_controller', collection={'subnets':'POST'})
	>>> print map
	Route name             Methods Path                      
	formatted_subnets_nets POST    /nets/subnets.:(format)   
	subnets_nets           POST    /nets/subnets             
						   POST    /nets.:(format)           
						   POST    /nets                     
	formatted_nets         GET     /nets.:(format)           
	nets                   GET     /nets                     
	formatted_new_net      GET     /nets/new.:(format)       
	new_net                GET     /nets/new                 
						   PUT     /nets/:(id).:(format)     
						   PUT     /nets/:(id)               
						   DELETE  /nets/:(id).:(format)     
						   DELETE  /nets/:(id)               
	formatted_edit_net     GET     /nets/:(id)/edit.:(format)
	edit_net               GET     /nets/:(id)/edit          
	formatted_net          GET     /nets/:(id).:(format)     
	net                    GET     /nets/:(id)            

The collection operation can mathc /nets/subnets on POST method.


## member

	>>> map=Mapper()
	>>> map.resource('net','nets', controller='net_controller', member={'subnets':'POST'})
	>>> print map
	Route name            Methods Path                         
						  POST    /nets.:(format)              
						  POST    /nets                        
	formatted_nets        GET     /nets.:(format)              
	nets                  GET     /nets                        
	formatted_new_net     GET     /nets/new.:(format)          
	new_net               GET     /nets/new                    
						  PUT     /nets/:(id).:(format)        
						  PUT     /nets/:(id)                  
	formatted_subnets_net POST    /nets/:(id)/subnets.:(format)
	subnets_net           POST    /nets/:(id)/subnets          
						  DELETE  /nets/:(id).:(format)        
						  DELETE  /nets/:(id)                  
	formatted_edit_net    GET     /nets/:(id)/edit.:(format)   
	edit_net              GET     /nets/:(id)/edit             
	formatted_net         GET     /nets/:(id).:(format)        
	net                   GET     /nets/:(id)                  

After use member parameter, the member operation can match /nets/{id}/subnets

## path_prefix and name_prefix

This two parameter are used together. For example:

	>>> map=Mapper()
	>>> map.resource("message", "messages", controller="categories", path_prefix="/category/{category_id}", name_prefix="category_")
	>>> print map
	Route name                      Methods Path                                                 
									POST    /category/{category_id}/messages.:(format)           
									POST    /category/{category_id}/messages                     
	formatted_category_messages     GET     /category/{category_id}/messages.:(format)           
	category_messages               GET     /category/{category_id}/messages                     
	formatted_category_new_message  GET     /category/{category_id}/messages/new.:(format)       
	category_new_message            GET     /category/{category_id}/messages/new                 
									PUT     /category/{category_id}/messages/:(id).:(format)     
									PUT     /category/{category_id}/messages/:(id)               
									DELETE  /category/{category_id}/messages/:(id).:(format)     
									DELETE  /category/{category_id}/messages/:(id)               
	formatted_category_edit_message GET     /category/{category_id}/messages/:(id)/edit.:(format)
	category_edit_message           GET     /category/{category_id}/messages/:(id)/edit          
	formatted_category_message      GET     /category/{category_id}/messages/:(id).:(format)     
	category_message                GET     /category/{category_id}/messages/:(id)           

The path_prefix add a url prefix to the urls that can be matched. The name_prefix is the prefix of route name.

They actually are parent resource of message. 

## Parent resource

	>>> from routes import Mapper
	>>> map=Mapper()
	>>> map.resource('location', 'locations', parent_resource=dict(member_name='region', collection_name='regions'))
	>>> print map
	Route name                     Methods Path                                              
								   POST    /regions/:region_id/locations.:(format)           
								   POST    /regions/:region_id/locations                     
	formatted_region_locations     GET     /regions/:region_id/locations.:(format)           
	region_locations               GET     /regions/:region_id/locations                     
	formatted_region_new_location  GET     /regions/:region_id/locations/new.:(format)       
	region_new_location            GET     /regions/:region_id/locations/new                 
								   PUT     /regions/:region_id/locations/:(id).:(format)     
								   PUT     /regions/:region_id/locations/:(id)               
								   DELETE  /regions/:region_id/locations/:(id).:(format)     
								   DELETE  /regions/:region_id/locations/:(id)               
	formatted_region_edit_location GET     /regions/:region_id/locations/:(id)/edit.:(format)
	region_edit_location           GET     /regions/:region_id/locations/:(id)/edit          
	formatted_region_location      GET     /regions/:region_id/locations/:(id).:(format)     
	region_location                GET     /regions/:region_id/locations/:(id)               

This is the same as path_prefix and name_prefix. The path_prefix here is /regions/{region_id} and name_prefix is region_

openstack use routes to map url to controller, so it is necessary to know about routes. Routes are came from Ruby on rails. It is rewritten in python. So docs about routes are not very clear might because the author think we should learn Ruby on rails routes first.



## Submapper

submapper is a lazy way to write code. See examples below:

You want to generate a series route， they have common attributes like 

    controller: common_controller
    action: index
    path_prefix: api/v2
    conditions: {'method':'GET'}

You can write the code in this way

    >>> from routes import Mapper
    >>> map = Mapper()
    >>> with map.submapper(controller='common_controller', action='index', path_prefix='api/v2/', contidions={'method':'GET'}) as submapper:
    ...     submapper.connect('api/v2/'+'nets', 'nets')
    ...     submapper.connect('api/v2/'+'subnets', 'subnets')
    ... 
    >>> print map
    Route name     Methods Path          
    api/v2/nets            api/v2/nets   
    api/v2/subnets         api/v2/subnets

In this way you can put the common attributes in submapper.
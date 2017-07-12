---
layout: post
title:  "How to read OpenStack code part 3 - stevedore"
date:   2017-07-12 08:43:59
author: Mingwei Li
categories: openstack
---

学习了WSGI/Paste deploy后，还需要对一些在openstack中一些package有一些了解，才能更好的理解openstack的代码

# What is stevedore

我们在写代码的时候通常把一个一个的功能块独立编写，甚至发布一定的规则和接口由第三方编写，然后在运行时根据实际情况来选择加载哪些功能模块。这样的好处是松耦合，灵活，而且便于协作。

stevedore就是一个很好的帮助动态加载代码的工具，openstack中很多plugin就是通过stevedore加载的。

下面通过一些应用场景看一下如何使用stevedore。

# Driver 场景

假设我们有一个开源系统，我们不希望限定用户对数据库的选择。用户可以选择mysql/oracle/mongo等任意数据库。这就意味着我们需要对很多种数据库提供接口。这个工作量很大，并且要求开发人员对每一种数据库非常熟悉。最好的做法就是把数据库的接口设计成driver，并发布出该driver所需要实现的接口。这样任何第三方的组织或个人都可以参与开发。

比如，我们定义下面的接口。任何开发者，只要编写一个实现了下列接口的class，就可以实现一个driver  
    
    @six.add_metaclass(abc.ABCMeta)
    class Base(object):
        """
        The Base class for db driver
        """
    
        @abc.abstractmethod
        def create_user(self, user_name):
            """
    
            :param user_name:
                The name of user
            :return:
                A user object
            """

基于上述接口，我们尝试开发一个mysql的driver如下：

    # mysql_driver.py
    import base
    
    
    class MysqlDriver(base.Base):
        def create_user(self, user_name):
            return "create user %s in mysql" % user_name

该package的目录结构如下

	[root@controller POTUS]# tree MysqlDriver/
	MysqlDriver/
	├── mysql_driver
	│   ├── __init__.py
	│   └── mysql_driver.py
	└── setup.py

要注意在mysql_driver/__init__.py中要有 import mysql_driver这句话

setup.py的内容如下：

    from setuptools import setup, find_packages
    
    setup(
        name='mysql_driver',
        version='1.0',
    
        packages=find_packages(),
    
        entry_points={
            'my_system.db_driver': [
                'mysql = mysql_driver.mysql_driver:MysqlDriver',
            ],
        },
    )

运行python setup.py install 后，会在系统目录中安装mysql_driver这个egg包，egg中有entry_point文件中记录了如下内容：

    [my_system.db_driver]

    mysql = mysql_driver.mysql_driver:MysqlDriver

my_system.db_driver是操作系统全局唯一的namespace。mysql是entry point的名字。通过这两个组合，可以定位到需要加载的代码，也就是我们的driver。

我们同样的再开发一个oracle_driver，如下：

	[root@controller POTUS]# tree OracleDriver/
	OracleDriver/
	├── oracle_driver
	│   ├── __init__.py
	│   └── oracle_driver.py
	└── setup.py

oracle_driver.py内容如下

    # mysql_driver.py
    import base
    
    
    class OracleDriver(base.Base):
        def create_user(self, user_name):
            return "create user %s in oracle" % user_name

setup.py内容如下：

    from setuptools import setup, find_packages
    
    setup(
        name='oracle_driver',
        version='1.0',
        packages=find_packages(),
        entry_points={
            'my_system.db_driver': [
                'oracle = oracle_driver.oracle_driver:OracleDriver',
            ],
        },
    )

不要忘了在__init__.py中加上 import oracle_driver。 


ok， 在python setup.py install 后，我们的系统中有了mysql_driver 和 oracle_driver 下面就是应用了。 

    >>> from stevedore import driver
    >>> mgr = driver.DriverManager(namespace = "my_system.db_driver", name = "mysql", invoke_on_load = True, invoke_args = ())
    >>> mgr.driver.create_user('Tom')
    'create user Tom in mysql'
    >>> mgr = driver.DriverManager(namespace = "my_system.db_driver", name = "oracle", invoke_on_load = True, invoke_args = ())
    >>> mgr.driver.create_user('Tom')
    'create user Tom in oracle'

可以看到，当name=mysql时，调用的是mysql_driver, 当name=oracle时，调用的是oracle_driver


Driver的方式，让不同的driver开发过程相互独立，调用也更加灵活

# Hook场景

Hook 顾名思义，钩子。就像window开发中的hook一样，每一个hook都是一个功能组件，会被一定的事件驱动。一个事件可以驱动一个hook，也可以驱动多个hook。同一个hook可以被多个不同的事件驱动。

假设我们开发了一个IDE工具，在鼠标点击关闭按钮的时候，会产生CLOSE_WINDOW EVENT， 这时候需要save_content组件保存内容。在鼠标点击commit按钮的时候，会产生COMMIT EVENT，这时候需要save_content发挥作用，也需要upload_content组件把内容上传到服务器。

这种场景下，我们可以开发save_content, upload_content两个组件，并且把save_content注册在CLOSE_WINDOW 和 COMMIT 两个event下，把upload_content注册在COMMIT event下。开发过程如下：

首先是save_content,


	[root@controller POTUS]# tree IDE_HOOKA/
	IDE_HOOKA/
	├── save_content
	│   ├── __init__.py
	│   └── save_content.py
	└── setup.py

save_content 内容如下：

    class SaveContent(object):
        def __call__(self, *args, **kwargs):
            return "content are saved"

setup.py 内容如下：

    from setuptools import setup, find_packages
    
    setup(
        name='save_content',
        version='1.0',
    
        packages=find_packages(),
    
        entry_points={
            'ide.hooks': [
                'CLOSE_WINDOW = save_content.save_content:SaveContent',
                'COMMIT = save_content.save_content:SaveContent'
            ],
        },
    )

__init__.py中有import save_content这句不要落下。

在python setup.py install 后，save_content.save_content:SaveContent 会被注册到CLOSE_WINDOW 和 COMMIT这两个event下。

我们再开发一个upload_content 组件如下：

	[root@controller POTUS]# tree IDE_HOOKB/
	IDE_HOOKB/
	├── setup.py
	└── upload_content
		├── __init__.py
		└── upload_content.py

主要看一下setup.py 
    from setuptools import setup, find_packages
    
    setup(
        name='upload_content',
        version='1.0',
    
        packages=find_packages(),
    
        entry_points={
            'ide.hooks': [
                'COMMIT = upload_content.upload_content:UploadContent'
            ],
        },
    )

这次，我们只注册了upload_content到COMMIT下。


在安装后，我们有了save_content 注册在CLOSE_WINDOW 和 COMMIT下， upload_content注册在COMMIT下。这样，当COMMIT事件发生，系统会保存内容并upload内容到服务器，当CLOSE_WINDOW事件发生，会保存内容。

运行如下：

    # import hookmanager
    # 定义运行hook的函数
    >>> from stevedore import HookManager
    >>> def run(hook):
    ...     return (hook.name, hook.obj())
    ... 

    # 根据Close Window调用hook
    >>> mgr = HookManager(namespace = "ide.hooks", name = "CLOSE_WINDOW", invoke_on_load = True, invoke_args = ())
    >>> mgr.map(run)
    [('CLOSE_WINDOW', 'content are saved')]

    # 根据commit调用hook 
    >>> mgr = HookManager(namespace = "ide.hooks", name = "COMMIT", invoke_on_load = True, invoke_args = ())
    >>> mgr.map(run)
    [('COMMIT', 'content are saved'), ('COMMIT', 'content are uploaded')]

# extension 场景

对比driver hook，他们都是通过setuptools的entry point来调用代码。driver是根据一个名字加载一个driver， hook是根据一个名字加载多个组件，而extension则是不根据名字，把namespace下所有的组件加载。


	[root@controller POTUS]# tree EXT/
	EXT/
	├── ext
	│   ├── ext1.py
	│   ├── ext2.py
	│   ├── ext3.py
	│   └── __init__.py
	└── setup.py

ext1的内容如下

    class EXT1(object):
        def __call__(self, *args, **kwargs):
            return "ext1 is called"

setup.py内容如下：

    from setuptools import setup, find_packages
    
    setup(
        name='ext',
        version='1.0',
    
        packages=find_packages(),
    
        entry_points={
            'my_namespace': [
                'ext1 = ext.ext1:EXT1',
                'ext2 = ext.ext2:EXT2',
                'ext3 = ext.ext3:EXT3'
            ],
        },
    )

按照完成后，ext1/ext2/ext3都被注册到my_namespace下。 调用方式如下：

    >>> from stevedore import ExtensionManager
    >>> mgr=ExtensionManager(namespace='my_namespace', invoke_on_load=True, invoke_args=())
    >>> def run(hook):
    ...     return (hook.name, hook.obj)
    ... 
    >>> mgr.map(run)
    [('ext3', <ext.ext3.EXT3 object at 0xd62a50>), ('ext2', <ext.ext2.EXT2 object at 0x102b6d0>), ('ext1', <ext.ext1.EXT1 object at 0x1034c10>)]

调用方式可以看到，通过namespace可以把其下所有组件加载


以上是stevedore的主要用法，下一篇讲展示routes的用法
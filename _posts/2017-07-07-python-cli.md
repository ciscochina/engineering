---
layout: post
title:  "Python实现命令行程序"
date:   2017-07-07 08:43:59
author: Peng Xiao
categories: python
---

使用Python写命令行程序，以argparse是基础，但是有两个更好的工具可以选择，click和oslo.config 

###  click

click可以用于简单的命令行程序，下面是我写的一个demo

https://github.com/xiaopeng163/click-demo 

```
$ cd click-demo
$ python setup.py install
$ clickctl
Usage: clickctl [OPTIONS] COMMAND [ARGS]...

  Click Demo Command Line Interface

Options:
  -v, --verbose  show debug message.
  --help         Show this message and exit.

Commands:
  init    Initializes a controller cluster on master node.
  join    join the controller cluster as agent node
  status  Get cluster node list
$ clickctl init --help
Usage: clickctl init [OPTIONS]

Options:
  --advertise-addr TEXT  The REST Server advertise address  [required]
  --help                 Show this message and exit.

$ clickctl init --advertise-addr=1.1.1.1:80
Try to initialized the cluster
$ clickctl -v init --advertise-addr=1.1.1.1:80
Try to initialized the cluster
The REST Server advertise address: 1.1.1.1:80
```

###  oslo.config

可以用于大型复杂命令行程序的开发，特别是命令行参数和ini格式配置文件同时结合使用的命令行程序
我在yabgp/yabmp里使用了oslo.config, 命令行的效果如下：

```
python bin/yabmpd -h
usage: yabmpd [-h] [--config-dir DIR] [--config-file PATH]
              [--log-backup-count LOG_BACKUP_COUNT]
              [--log-config-file LOG_CONFIG_FILE] [--log-dir LOG_DIR]
              [--log-file LOG_FILE] [--log-file-mode LOG_FILE_MODE]
              [--nouse-stderr] [--noverbose] [--use-stderr] [--verbose]
              [--version] [--bmp-bind_host BMP_BIND_HOST]
              [--bmp-bind_port BMP_BIND_PORT] [--bmp-write_dir BMP_WRITE_DIR]
              [--bmp-write_msg_max_size BMP_WRITE_MSG_MAX_SIZE]

optional arguments:
  -h, --help            show this help message and exit
  --config-dir DIR      Path to a config directory to pull *.conf files from.
                        This file set is sorted, so as to provide a
                        predictable parse order if individual options are
                        over-ridden. The set is parsed after the file(s)
                        specified via previous --config-file, arguments hence
                        over-ridden options in the directory take precedence.
  --config-file PATH    Path to a config file to use. Multiple config files
                        can be specified, with values in later files taking
                        precedence. Defaults to None.
  --log-backup-count LOG_BACKUP_COUNT
                        the number of backup log file
  --log-config-file LOG_CONFIG_FILE
                        Path to a logging config file to use
  --log-dir LOG_DIR     log file directory
  --log-file LOG_FILE   log file name
  --log-file-mode LOG_FILE_MODE
                        default log file permission
  --nouse-stderr        The inverse of --use-stderr
  --noverbose           The inverse of --verbose
  --use-stderr          log to standard error
  --verbose             show debug output
  --version             show program's version number and exit

bmp options:
  --bmp-bind_host BMP_BIND_HOST
                        Address to bind the BMP server to
  --bmp-bind_port BMP_BIND_PORT
                        Port the bind the BMP server to
  --bmp-write_dir BMP_WRITE_DIR
                        The BMP messages storage path
```

### 环境变量

另外因为如今容器非常流行，对于容器应用程序的配置，命令行不是很方便，一般采用环境变量的传递方式，如

```python
os.environ.get('MONGODB_URL', 'mongodb://127.0.0.1:27017')
```
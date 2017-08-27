---
layout: post
title: "Sharing of Selenium Testing Framework"
date: 2017-08-26 20:35:33
author: Ronghui Zhou
categories: python
---

# What is Selenium? 

From the http://www.seleniumhq.org/, it gives a clear definition: Selenium automates browsers. That's it!

Please check out below links if you are interest: 

### Getting started

http://selenium-python.readthedocs.io/getting-started.html (For Python)

### Source Code

https://github.com/SeleniumHQ/selenium

```
In generally, if your application is web-oriented and with using the Selenium for functional testing and unit test for data/backend logical testing,
You would get an complete web application testing solution.
```

#### Also want to share some python coding tips that I got during the refactor of vots project

##### 1. 

We know that everything is an object in python, so it makes easy for us to write ‘table-driven’ methods in python.
For example, we get many ‘if-else’ and complex logic in each ‘if’, then we can use table-driven, extract the ‘if’ condition into ‘key’, and the ‘logic into ‘value’ of a map.
Likes below : 

```        
def do_print(in_str: str):
    print(in_str)
                
def do_print_with_pre(in_str: str):
    print('pre - ' + in_str)


def do_print_with_end(in_str: str):
    print(in_str + ' - end')


operations = {'1': do_print, '2': do_print_with_pre, '3': do_print_with_end}

operations['1']('CISCO')
operations['2']('CISCO')
operations['3']('CISCO')
```

##### 2. 

From above screenshot we can see python support for type hints already. This is new in version 3.5 and I think is very helpful especially when the input parameter is an complex object .
The function below takes and returns a string and is annotated as follows: 

```                             
def greeting(name: str) -> str:
    return 'Hello ' + name
```

#### 3. 

Write unit test for your code.
For example, Vots project has two threads with loop and this makes it hard to debug when we get a bug.
Unit test and ‘mock’ can solve this issue.
First we reproduce bug and record real response into a txt file.
Second we write unit test for our code, and mock the return value of the related methods with the real response in the txt file.
Then we can begin debug from the unit test.
Except this, unit test can improve the quality of your code and make it easy to maintenance your code for continue delivery.

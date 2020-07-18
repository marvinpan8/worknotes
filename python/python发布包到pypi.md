# [python发布包到pypi的踩坑记录](https://www.cnblogs.com/rongpmcu/p/7662821.html)



## 前言

　　突然想玩玩python了`^_^`　这篇博文记录了我打算发布包到pypi的踩坑经历．python更新太快了，甚至连这种发布上传机制都在不断的更新，这导致网上的一些关于python发布上传到pypi的教程都过时了，按着博文操作会失败，所以请记住，我这篇博文的介绍也许在你看到时也过时了`^_^`

## 本地发布
示例
```bash
#!/usr/bin/bash

rm -rf dist jrtzcloud_python_sdk_core.egg-info
python setup.py sdist
cd dist/
tar -zxvf jrtzcloud-python-sdk-core-1.0.0.tar.gz
cd jrtzcloud-python-sdk-core-1.0.0
python setup.py install
cd ../../
```


## pypi相关概念介绍

#### 关于pypi本身

　　pypi是专门用于存放第三方python包的地方，你可以在这里找别人分享的模块，也可以自己分享模块给别人。可以通过easy_install或者pip进行安装。pypi针对分享提供了两个平台，一个是测试发布平台，一个是正式发布平台，我们正式发布前可以先用测试发布平台发布，看是否正确，然后再采用正式发布平台．

#### 关于python打包发布工具

　　python的打包安装工具也经历了很多次变化，由最早的distutils到setuptools到distribute又回到setuptools，后来还有disutils2以及distlib等，其中distutils是python标准库的一部分，它提出了采用setup.py机制安装和打包发布上传机制．setuptools(操作系统发布版本可能没有自带安装,需要自己额外安装)基于它扩展了很多功能，也是采用setup.py机制，针对安装额外提供了easy_install命令．distribute是setuptools的一个分之，后来又合并到setuptools了，所以姑且就把它看做是最新的setuptools吧！和我们打包最相关的貌似就是distutils或者setuptools，两者都可以用来打包发布并上传到pypi，后面介绍采用distutils，如果想更多的功能，比如想通过entry points扩展的一些功能，那么就要使用setuptools了．另外，还有一个工具可以用来发布到pypi，叫twine，需要额外安装．最后,需要确保自己的工具都是尽量新的,官方给出的版本参考:twine v1.8.0+ (recommended tool), setuptools 27+, or the distutils included with Python 3.4.6+,Python 3.5.3+, Python 3.6+, and 2.7.13+,升级的参考命令`sudo -H pip install -U pip setuptools twine`

## 打包发布到pypi

基本流程：

- 注册[pypi](https://pypi.python.org/pypi?:action=register_form)账号，如果期望测试发布，同时需要注册[pypitest](https://testpypi.python.org/pypi?:action=register_form)账号（可以采用相同的用户名和密码）
- 创建配置文件, 该配置文件里面记录了你的账号信息以及要发布的平台信息，参考如下配置文件创建即可

```
[distutils]
index-servers =
  pypi
  pypitest

[pypi]
repository: https://upload.pypi.org/legacy/
username: your_username
password: your_password

[pypitest]
repository: https://test.pypi.org/legacy/
username: your_username
password: your_password
```

该配置文件可以保存到~/.pypirc中,以后基本就不需要修改了。注意，这一步的配置文件编写由于pypi的发布机制更新导致有一些坑的出现，后面会讲述

- 配置工程，这一步会因为工程的复杂度而有较大不同，主要是工程配置文件setup.py，这里用个简单的工程进行讲述.创建工程文件夹,并创建helloxxxxx.py模块(里面的内容随意，创建一个打印hello的函数也行)以及setup.py文件，setup.py参考如下配置文件创建即可

```
from distutils.core import setup

setup(
    name = "helloxxxxx",
    author = "name",
    version = "1.0.0",
    author_email = "xxx@gmail.com",
    py_modules = ['helloxxxxx'],
    url = "http://www.xxx.com"
    )
```

注意，这里只是最基本的参考例子，执行打包会报警告，说缺少一些需要的文件，比如MANIFEST.in、readme.txt等等，暂时忽略即可。正式的项目中会复杂很多，甚至需要用到setuptools来扩展。这部分可以参考其他文档

- 为了保证效果，在打包之前我们可以验证setup.py的正确性。执行代码`python3 setup.py check`，输出一般是running check，如果有错误或者警告，就会在此之后显示.没有任何显示表示Distutils认可你这个setup.py文件
- 执行`python3 setup.py sdist upload -r pypi`创建发布并上传,如果想先上传到测试平台，可以执行python setup.py sdist upload -r pypitest，成功后再执行上面命令上传到正式平台。注意，这一步的配置文件里面由于pypi的发布机制更新导致有一些坑的出现，后面会讲述

## 踩坑记录

之所以有这些坑, 是因为python更新太快以及自己没有认真看最新官方文档导致的！

### 坑1

410错误，这个是pypi上传机制变更导致的,我虽然参考的是最新的blog，但是还是过时了！！！

首先，.pypirc的repository过时了，很多博客说的`repository: https://pypi.python.org/pypi`会导致后面步骤操作出现410错误

其次，很多博客说的两步走方案

```bash
#python setup.py register -r pypi
python setup.py sdist upload -r pypi
python setup.py sdist upload -r https://test.pypi.org/legacy/
```

也过时了，现在不需要第一步了

### 坑2

400错误，setup.py配置文件里面某些字段我写的太随意，导致上传的时候出错，比如url那里我没有加`http://`，这里也需要大家注意

### 坑3

403错误，这是因为项目和已有的项目重名了，可以先到`https://pypi.python.org/simple/`上搜一下看看是否重名。解决的方法自然就是修改一下setup.py中setup函数中的name参数，删除之前生成的dist文件夹并重新生成，然后再upload

## 参考

- [Migrating to PyPI.org](https://packaging.python.org/guides/migrating-to-pypi-org/#uploading)
- [发布你自己的轮子 - PyPI打包上传实践](https://segmentfault.com/a/1190000008663126)
- [Python模块的打包与发布](http://python.jobbole.com/86443/)

完！
2020年4月
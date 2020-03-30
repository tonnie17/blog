+++
title = "Flask 10天开发一个网站"
date = 2018-01-07T00:00:00+08:00
tags = ["python"]
categories = [""]
draft = false
+++

pkyx是一个用Flask+MongoDB开发的比较（维基）网站。

## **Day 1：配置远程开发环境**

首先在 Paralles Desktop下安装了64位的Ubuntu 15.04版本，里面配置了nginx和virtualenv。

1.在Ubuntu中新建一个目录，用virtualenv创建好虚拟环境，用pip安装flask，接着测试一下。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-e4685802020515e730d7343bad6eba37_1440w.jpg)

2.在Pycharm下新建一个项目，编译器选择虚拟机中用virtualenv里的python解释器。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-dd7cc3b4abfdf5f7929384e863740b56_1440w.jpg)

3.在虚拟机中使用ifconfig命令查看ip地址，然后配置好ssh的选项，连接前记得已经在虚拟机中安装好ssh-server，因为默认它是只有ssh-client而没有ssh的服务器的，用apt-get install openssh-server安装之，用户和密码为虚拟机中的用户名和密码。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-5a2ca36a00e492e44c04753cb9b51500_1440w.jpg)

3.创建好项目后，在Tools-Deployment-Configuation中配置sftp选项，在这里要注意Root Path是指在远程主机（ubuntu）中的顶层路径。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-d9179423db19ae1f18f337ea713b06a9_1440w.jpg)

在Mapping选项卡中，配置项目路径映射到远程主机的项目路径。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-6db0a5c357562e392a65d94e2ca652be_1440w.jpg)

4.在Tools-Deployment-Option中配置Upload changed files automatically to the default server。这里是选择host的项目和远程主机项目的同步时刻，Always是一直保持同步，Ctrl+s是指保存时同步。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-b34fc1802d2ff341a740c44cc0f72f16_1440w.jpg)

5.编写代码，这里写了一个返回hello world的路由。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-0eaf991284d3a09935ddb07548a2f5c5_1440w.jpg)

6.把代码同步到远程主机上，Upload to直接把代码推送到虚拟机的项目路径中，Sync with Deployed to..查看项目部署的文件状态和选择同步的文件。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-3ac2adfef2999c7dc62d173c42202176_1440w.jpg)

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-b7b6d28aa497b911270825f1a4f98d0e_1440w.jpg)

到这里基本上配置已经算完成了，下面直接Run运行代码。

7.运行代码。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-f632f5d9064e3ab3491458a3e2dbade9_1440w.jpg)

可以看到，flask的程序已经开始启动，但是这里要注意，在本机是不能够直接访问虚拟机上的localhost的，所以这里的[http://127.0.0.1:5000/](https://link.zhihu.com/?target=http%3A//127.0.0.1%3A5000/)是指在虚拟机的服务，而在host这边是无法通过此路径查看的。那怎么办？之前我们说过在虚拟机中配置了nginx，此时它的作用就来了，总所周知，nginx的其中一个常见用途就是作反向代理，于是，在这里我们也用nginx代理flask程序。

（其实这里也有另外一种方法，就是在Paralles中的网络设置里面配置端口映射，这样就有办法在host主机中访问到虚拟机的localhost了。）

8.修改nginx配置文件（/etc/nginx/site-available/[conf]）。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-723460814c10d96e95f7585098af3077_1440w.jpg)

我们的nginx服务器监听了虚拟机的80端口，把跟路径的request转发到flask绑定的5000端口，而静态文件路径（/static）的请求则绕过flask，直接访问虚拟机上的文件目录，这样也有效地减轻了flask程序的负荷。

9.获取虚拟机的ip地址，通过浏览器访问web程序。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-e5470aaee47fddfd524b93c4765f92c3_1440w.jpg)

perfect

10.别忘了把项目同步到git上了。

## **Day 2：编写应用配置和视图**

在编写配置和视图之前，首先为应用规划目录结构。

```text
code/pkyx/
├── app（应用目录）
│ ├── config.py（配置文件）
│ ├── forms.py（表单文件）
│ ├── init.py
│ ├── main（主体模块）
│ │ ├── errors.py（错误视图）
│ │ ├── init.py
│ │ └── views.py（主体视图）
│ ├── static（静态文件目录）
│ │ └── style.css
│ ├── templates（模版文件目录）
│ │ ├── 404.html
│ │ ├── 500.html
│ │ ├── index.html
│ │ └── pk.html
│ └── users（用户模块）
│ ├── init.py
│ └── views.py
├── manage.py
```

接下来我们再做一些准备工作，用pip安装如下几个扩展。

- Flask-Mail：Flask的邮件扩展，利用它可以方便快捷地给用户发送邮件。
- Flask-WTF：Flask的表单扩展，用它可以在代码中编写表单类和基础属性，在模版中渲染表单。
- Flask-PyMongo：Flask的基于Pymongo的扩展，在为Flask应用部署MongoDB的连接时更加的快捷方便。

1.接下来，编写表单文件。

首先从WTF扩展中导入Form类，我们要定义的表单类会继承到这个Form类，接下来简单地为表单定义三个域，两个文本输入框，validators中加入了wtforms.validators中的DataRequired的实例，它将会把这两个文本框设置为必填项。最后还有一个提交域，也就是提交该表单的按钮。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-309b08231b5925309372fee69abe1f03_1440w.jpg)

2.编写蓝本文件。

蓝本（Flask-Blueprint）有许多用途，其中一个常见的用途即是为应用的模块做url的划分。

```text
在一个应用的 URL 前缀和（或）子域上注册一个蓝图。 URL 前缀和（或）子域的参数 成为蓝图中所有视图的通用视图参数（缺省情况下）。
```

关于蓝本的详细说明：[http://dormousehole.readthedocs.org/en/latest/blueprints.html#blueprints](https://link.zhihu.com/?target=http%3A//dormousehole.readthedocs.org/en/latest/blueprints.html%23blueprints)

在Blueprint的参数中还可以指定模块的静态文件路径以及模版文件路径。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-cc37b3efad19fee922804a443c67f06d_1440w.jpg)

3.编写错误响应视图。

为应用编写错误的响应视图十分重要，这里简单地定义了两个视图，分别对应错误404和错误500的响应，注意这里使用的main为上一个中定义的蓝本对象，若在英语中注册了蓝本，被装饰器包装的视图函数都会注册到应用中，它会把 构建Blueprint时所使用的名称（在本例为simple_page）作为函数端点 的前缀。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-79fa201635784f4f659ab743bd637f38_1440w.jpg)

4.编写主体视图。

这里定义了首页（／）和比较页（／pk）的对应视图，在index中，我们把之前定义的PkForm表单实例化，作为渲染模版函数render_template的上下文参数传入到模版中渲染。在pk视图中，它接受来自index表单中post提交的数据，因此要在它的装饰器的methods参数中加上'POST属性（默认只有GET），注意要把request.form作为表单的构造函数的参数传入，否则表单将不能接收到任何数据，然后我们用表单到validate_on_submit方法来判断表单的数据是否合法，若正确，则把输入框的两个数据拿出，渲染到比较页中，否则，便简单地返回一个显示错误的响应。

![img](data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='1762' height='962'></svg>)

5.编写模版。

这里简单地在模版定义一个表单，它指向之前定义的pk视图，当然了，只有这么简单的显示是远远不够的，在static中定义css样式文件，在页面的头部的link标签的href属性指定用url_for()反向解析得到的静态文件路径（这里要感谢强大的jinjia2模版:））。

![img](data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='1920' height='964'></svg>)

这里的效果大概如下：

![img](data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='2784' height='1648'></svg>)

接着编写比较页的模版。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-8ac0c8141d3c0d6b54d74ad67081595b_1440w.jpg)

6.定义配置文件。

到这里，应用的框架基本清晰，但我们需要更加灵活的启动和运行应用，在app目录下编写全局的配置文件。

在BaseConfig中定义了应用的一些基本配置，例如秘钥，邮箱配置等等，下面所有的其它配置都会继承BaseConfig，扩展出的其它配置，这里定义了一个DevConfig（开发配置），顾名思义是在开发中的配置，除此之外，还可以定义其它类型的配置（如生产配置，测试配置等），在DevConfig中我们扩展了关于MongoDB连接（Flask-PyMongo）的配置，以及一个静态方法init_app，它会应用进行一些配置的初始化（如建立数据库的连接）。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-db0a2ca9c1bb1335747f7e1b6749fd6e_1440w.jpg)

7.编写创建应用的函数。

这里编写了一个创建和初始化应用的函数，它将负责为应用初始化传入的配置，使用配置的init_app初始化自身，以及注册前面编写的蓝本。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-a1061bc29d7b1f7e760b900f28400f38_1440w.jpg)

8.建立管理应用脚本（manage.py）。

最后应用还需要一个全局的管理脚本，这里暂时只需要加上启动应用的代码。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-1800c7c9b274cc741a9080595ead802d_1440w.jpg)

9.启动和运行应用。

完成上面的步骤，应用就能跑起来了。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-dce6cafd927fde47aec8728e5a8b088c_1440w.jpg)

## **Day 3：编写RESTful API和测试数据库**

前面已经完成了视图，模板，表单的工作，应用也已经可以运行了。

接下来开始编写API了，REST（表现层状态转换）设计风格是当前最流行的设计模式，接下来将会为应用编写REST风格的API。

[关于RESTful](https://link.zhihu.com/?target=http%3A//www.ruanyifeng.com/blog/2011/09/restful)

1.还是老规矩，新建API模块的目录，和**init**.py，定义Blueprint。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-4a3f1626cac713c25a5955741c72d335_1440w.jpg)

2.测试数据库

在app模块的初始化中，我们创建了MongoDB的连接实例，但这个实例是还没有绑定到当前的应用上下文的，因此还要在create_app中为mongo实例用创建出来的app添加到连接实例的init_app方法，这时候，MongoDB就正式可以在应用中工作了，只需要在其他文件中导入app模块的mongo实例`（ from app import mongo）`。

![img](data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='1466' height='924'></svg>)

接下来，在mongo shell中新建一个存放条目的集合，插入一条数据。。

3.编写工具函数（utils）。

这里的两个工具函数（bson_to_json，bson_obj_id）是待会在编写API视图的时候要用到的，其作用分别是把MongoDB的BSON（文档的数据格式）转换为JSON和把id转换为MongoDB中的ObjectId形式，因为这两个功能都难以用python内置的函数实现，而pymongo为我们提供的bson模块提供了很好用的json_util，使得我们很方便地去实现MongoDB和Python之间的数据格式转换。

![img](data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='1504' height='878'></svg>)

一张图就能解释其中的流程。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-3541e400ca47cec48813119348a8f757_1440w.jpg)

4.编写REST API。

终于到了重中之重的步骤了，在Python的Web框架中实现REST API不是一件难事，Python社区也有许多关于REST的包（EVE，REST framework等），而在Flask里，flask默认为开发者提供了可组装视图（Pluggable View），其中里面有一个MethodView，就是专门为开发者设计REST风格的视图的，这种类视图有一个as_view()方法，使用它可以直接把类视图转换成平时使用的普通视图，在有重用视图的需求时，更是比普通函数视图更加地灵活。

首先，编写好一个API类，继承MethodView，简单地实现get、post、put、delete四个方法，分别对应四个HTTP方法对应的处理句柄。在代码的最后加上路由的规则，映射到不同的方法中，注意get方法中有两种情况，一种是提供id，只返回特定的资源，不提供id则返回所有（或前N条）资源，用add_url_route()方法动态地添加路由。

最后先简单地实现一下get方法，用pymongo把资源从数据库拉取，find()方法返回的是结果游标，注意这里用到了bson_to_json()方法，前面说过了，这是把MongoDB的文档格式从bson转换为json格式，拿到存放json数据的列表之后，再用json.dumps()返回之。最后客户端得到的就是一个json格式的对象数组了。

最后的最后要注意的是在find()方法中传入了一个params字典，这个字典是存放GET请求后面带的参数的键值对的，有了条件查询，我们构建的API会更加灵活。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-7e53224caa7b0f252d765dc800525dd5_1440w.jpg)

其他的方法就按照各自的条件实现。

5.测试REST API。

这里用Postman向服务器的api地址发送了一个GET请求，参数是之前插入到MongoDB中的文档的id字符串，最后得到一条结果。（若不加参数的话，则得到所有结果）

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-1497c4506604028df28b910fe67b8e5c_1440w.jpg)

再测试了另外一条GET请求，这次则是加上了数据的属性与值作为query string，发送查询请求，结果得到一条记录。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-abfefe4c00e564bb3a9f4442f5b886ac_1440w.jpg)

## **Day 4：使用Supervisor和Gunicorn优化应用**

Sueprvisor是Linux上的一个可以监控应用和进程的工具，我们用它来作为守护进程，自动化地启动和停止应用。

首先在系统上用sudo apt-get install supervisor安装它。

接着再用pip install gunicorn安装gunicorn，gunicorn是用python实现的高性能wsgi服务器，flask自带的wsgi服务器不适合在生产环境使用，我们使用gunicorn作为flask应用的服务器，提高应用的吞吐量和响应速度。

接下来在应用的目录下新建一个gunicorn的配置文件，里面配置了四个工作进程（wokers）和绑定的端口（bind）。

除此之外还要创建应用专属的supervisor的配置文件，其中主要的参数有几个：

- [program:[app]]：指定应用的名称
- command：指定启动应用的命令
- directory：应用所处的工作目录
- stdout_logfile：标准输出日志
- stderr_logfile：标准错误日志

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-35076f93ec12b0c47c3579e32597038b_1440w.jpg)

用supervisor启动应用进程也非常简单，只需要在supervisorctl的控制台里输入对应的命令即可运行应用。

注意的是，在使用supervisor之前，要先要用supervisord -c [conf_path]和supervisorctl -c [conf_path]命令指定好supervisor自身的配置文件。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-017a8b6e943a5cd24d87d80aa920a4ca_1440w.jpg)

当然了，在开发调试环境下还是不太适宜用gunicorn和supervisor来启动应用的，因为这样做不便于查看应用的输出和错误信息，而要在日志中观察应用的运行状态。

## **Day 5：编写用户和认证模块**

使用Flask-Login扩展能够很方便地为你的应用实现用户的会话和登录功能。

首先用pip install Flask-Login安装之。

除此之外，在认证模块要用到Flask-httpauth这个包，我们将在用户的REST API用认证的方式来管理请求。

安装方式：pip install Flask-httpauth。

1.说到用户模块，当然离不开登录／注册功能，那么我们首先编写登录和注册表单。

代码包括了最常用的几个域，这里就没什么好说的了。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-6910383fde0115e0e87ba06d98e40ec9_1440w.jpg)

2.编写用户模型（Model）。

尽管我们的应用没有采用ORM模型的形式，而是用pymongo来直接与MongoDB交互，但我们还是需要编写一个通用的用户模型，一是因为Flask-Login中要用到关于用户的模型对象，二是方便在认证模块中管理用户。

新建一个models.py文件，定义一个User类，添加Flask-Login模块里的UserMixin，这个Mixin会为我们定义的用户类声明一些通用的用户状态的property，如is_anonymous, is_authorized, is_active等等，混入UserMixin后，User类便能作为Flask-Login的用户模型一样被对待。

在这个用户类定义了四个方法。

1）gen_passwd_hash(password)

返回用哈希算法加密后的密码，因为我们的密码是不能让它明文地保存在数据库的，在这里使用werkzeug.security中的generate_password_hash方法来加密密码。

2）verify_passwd(passwd_hash, passwd)

把输入的密码与加密的哈希密码做对比，验证其正确性。

3）gen_auth_token(self, expiration)

生成一个带有过期验证的访问令牌。

4）verify_auth_token(token)

验证访问令牌，若成功，返回用户信息。

![img](data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='2784' height='1674'></svg>)

3.初始化LoginManager。

在应用模块中新建Flask-Login的LoginManager的实例，用它当前绑定应用，指定用户登录的视图（这里是'users.login'）。你必须提供一个 user_loader回调。这个回调用于从会话中存储的用户 ID 重新加载用户对象。它应该接受一个用户的ID 作为参数，并且返回相应的用户对象。

[https://flask-login.readthedocs.org/en/latest/](https://link.zhihu.com/?target=https%3A//flask-login.readthedocs.org/en/latest/)

![img](data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='2784' height='1674'></svg>)

4.编写用户主要视图。

这里分别定义了register(注册),login(登录),profile(用户资料),logout(注销)四个视图，这里要注意的是错误的判断以及用户验证流程，用flask-login中相应的login_user(user)和logout_user()来实现登录／登出功能，最后记得在需要保护的视图中添加上login_required装饰器，防止未经登录的访问。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-dc96d6d1e40702e48b43ea04677724e0_1440w.jpg)

5.添加/修改登录注册模版。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-27a4da7a4862fb0f2bff95df48b5a640_1440w.jpg)

6.测试登录／注册等功能。

前面已经实现了用户的登录功能，现在注册一个用户并登录测试应用。

注册。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-099b898474dba4dcb7d5406dfeced2e5_1440w.jpg)

接着登录。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-dd1f87ee92e1fbfa6a4860c98ac5f5ec_1440w.jpg)

登录成功。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-666aa9e8f4921bdc54743b890bb4282f_1440w.jpg)

7.编写用户认证模块。

前面在定义用户类的时候就已经写好了一些认证的方法。

现在我们使用flask-httpauth来构建带有用户认证的REST API。

在api模块目录下新建users.py。

创建一个HTTPBasicAuth实例，定义核心的verify_password函数，它将完成用户认证的功能，这里需要用auth的verify_password装饰这个函数。

verify_password中提供了两种认证方式，首先是用token认证，如果不通过则用用户＋密码的方式入库验证。

接着用login_requried包装一个获取token的视图和一个资源视图。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-85188409b6d0ab3747e5ebf387b73e8a_1440w.jpg)

接着开始测试api。

如果不用认证的方式去访问资源的话，会得到一个access denied的响应。

我们用邮箱(用户名):密码的方式访问资源，成功返回一个资源。

然后用认证的方式请求token所在的视图，获取到带有过期时间和token的json，下面不用用户名跟密码，而是用token代替去访问应用资源，同样正确地返回资源。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-c7f8758fb3ead6af861a8508f7d3c3e9_1440w.jpg)

## **Day 6：用模版继承组件化应用**

今天来为应用完善之前写好的模板。

在完善页面前最好先为模板添加上一些样式，不然页面看起来不美观，也没有层次感，不便于调整页面元素。因此在原来的基础上，可以添加上自己写的样式，用link的方式导出到前端，或者直接使用开源的css框架，在这里使用的是semantic-ui，用bower安装到应用的资源目录，然后就可以编写模板了。

由于一个网站中通常会有一些重复的基础组件（如导航栏，顶部，通用样式等），这样一来在每个页面文件中我们都要把这些几乎相同的代码拷贝，而且当要改动元素时得每个文件都要进行修改，给开发带来许多的不便，这时候便要用到jinjia2的模板继承功能，使用模板，我们能把这些通用的部分封装出来作为模板，需要替换的地方只需要在特定的位置添加一个块，在继承的子模板中填充块的内容即可。

1.下面在模板目录下新建一个基础文件(base.html)作为需要继承的通用模板。

在该目标中包含了一些基本样式和脚本，定义了三个需要填充的块，分别是head头部，主体内容和js文件。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-1ee5c7ea061b6f7ca964c66b1d780286_1440w.jpg)

2.继承父模板。

修改之前定义的首页文件（index.html），用extends的方式继承了父模板，然后只需要在相应的块中补充内容，子模板在渲染的时候便会把块中的内容导出到自身的对应块中，构造和实现页面，这里还把顶部栏（header.html）以及登录框（login.html）分别独立出一个组件，只供特定的模板使用，这样的导入方式会带来更大的灵活性。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-4a1eb9bebe89c594fdfe440725ebbc1f_1440w.jpg)

3.编写组件。

可以把这里的组件理解为页面中的一部分，因此在写代码的时候只需吧对应部分的html元素完成即可。

顶部栏组件

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-a999ca03b5c27080a2e47f84b786dee0_1440w.jpg)

登录框（模态框）组件

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-06df4bc82dd81f38e851856a30f74186_1440w.jpg)

4.整合模板。

现在处理过后的组件可以像积木一样装载在应用上了，这里简单地应用在几个页面，查看效果。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-8ac4395917c0f184c2991fda71577668_1440w.jpg)

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-ec8ac671c9f4e45240cbb0b23b87dc96_1440w.jpg)

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-8ab2cbeea5b529914fab1043207c5a54_1440w.jpg)

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-75e437d59205b88e1c6bad25d6e90944_1440w.jpg)

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-3784ef82473698326d29bf6b5af7ecee_1440w.jpg)

## **Day 7：编写主功能**

接着上次的地方开始做。

1.建立创建条目的页面，一个表单搞定，server端向数据库插入一条文档，so easy。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-bc13d263a389fa7160dd834021a1df42_1440w.jpg)

2.创建条目之后，系统会重定向到条目的信息页面，编写信息页面的布局。

如下所示，条目内容将会由一个表格来展示，刚创建的条目只有类型一个属性，页面底下会有一个添加属性的按钮，动态地向该条目插入属性。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-ddc3e9c21a5a15d387f13f67359de227_1440w.jpg)

由于表格的内容是由python端进行渲染的，而一个条目的属性可能有多种类型。如下图所示：

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-1e51aaad02b33936986388dc296ec8ed_1440w.jpg)

这就带来一个问题，因为属性类型不一定是纯文本，因此不能简单地从数据库取数据然后再直接渲染。

这里要实现动态的渲染，要在ajax和server端之间定义一套规则，向条目添加属性的时候会按照不同的类型来构造出特定的数据格式，然后python端在知道格式的情况下，用自定义的渲染器实现html的插入。

有了这个思路，马上开始编写代码。

当点击添加属性按钮后，底下会折叠处一个标签页菜单，选择不同的类型时，下面的输入框也会相应地变化。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-3ca872aa8fb4f6bda85a19ca2d2ab50a_1440w.jpg)

当然，js也要相应地配合，用ajax请求服务器端，进行数据更新。

根据选择类型获取属性值。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-0070a0ef708d18c161b0b31db41dc110_1440w.jpg)

Ajax请求

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-fa17a3129841839449b5493d092f34a0_1440w.jpg)

3.编写渲染器。

jinjia2的灵活性使得我们可以方便地在模板中使用python代码，下面定义了一个简单的类型渲染器，根据传入的【属性名，属性值，属性类型】构造出html。

![img](data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='2560' height='1600'></svg>)

然后在页面中这样调用，记得使用safe过滤器取消转义。

![img](data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='2560' height='1600'></svg>)

4.增加编辑/删除属性方法。

添加属性有了，怎么可以没有编辑和删除属性的方法呢？

一开始想用可编辑表格的方式对表格的数据进行即时修改，却发现这样做会带来一些问题，最终放弃之。

改成了双击表格列，用模态框修改之。

修改成功之后会刷新页面，看到修改后的表格。

![img](data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='2784' height='1672'></svg>)

5.完善表单验证。

因为之前做的表单验证实在是太简陋了，出于安全考虑，一定要对表单验证（尤其是用户信息方面）加以完善。

编辑表单文件，在之前的基础上，我们改用一个validators字典来存放不同表单域的规则，在用户名和密码中都加上了长度以及正则表表达式加以限制。

![img](data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='2560' height='1600'></svg>)

在页面中也要提示用户怎么填写表单。

![img](data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='2784' height='1770'></svg>)

6.完善样式。

最后再修饰一下页面。

![img](data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='2784' height='1770'></svg>)

![img](data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='1600' height='1017'></svg>)

## **Day 8：使用GridFS实现文件上传**

文件上传有许多种方法，一般用文件系统的io即可，这里使用了mongodb的GridFS系统，mongodb推荐使用它来保存大型文件，

这里先尝鲜试用一下，用gridfs实现用户头像的上传。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-f74c11c6601a158d7fca351ef07c95d4_1440w.jpg)

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-08d40af8495a04eaa6a1ffca61d6dbbf_1440w.jpg)

在flask端，用request.files来接收上传的头像图片，判断图片的扩展名是否符合格式，如果合法，用werkzurg.utils的secure_filename方法来替换和过滤文件名的特殊字符，接下来实例化GridFS类，它接受一个database和集合作为参数，用fs对象的put方法上传到GridFS，返回的Object Id指向的是db.avatar.files插入的文档，把头像id以及用户的资料信息一并保存到数据库。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-52885b06202a17862d98b918e9ff5b9f_1440w.jpg)

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-bafe65d94a26af3cd7b58e3bb2cdf329_1440w.jpg)

在前端的页面用url_for()方法反向解析出图片的地址，首先要编写好获取头像的路由，它接受头像的Object Id作为参数，从fs系统中取出头像图片的数据，图片的二进制数据会保存在db.avatar.chunks中，这里获取图片之后，构造一个content-type为图片格式的response，否则当打开url时，图片数据不能正确被浏览器解析，在img的src属性中填上路由即可显示图片。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-25adc0b1730442b6b42c29493d8c9f56_1440w.jpg)

用户的资料可能有不全的情况，用jinja2的Environment Filter来编写自定义的过滤器，使数据的显示更加人性化。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-da2a3a150ffcdfcb3fe62598055aa575_1440w.jpg)

最后编写比较页面，这里要注意属性的顺序排置，实现正确地渲染。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-b63a879bf0992d6759b43712973805e3_1440w.jpg)

到此网站页面端初步完成。

## **Day 9：配置Celery&Redis运行后台任务**

有些时候，我们的应用会执行一些后台任务，例如一些不会与用户直接交互，实时性要求较低的动作。例如用户注册的时候，通常会发送一封带有认证token链接的邮件到用户的邮箱，因为发送邮件这个动作会比较耗时，如果同一时间有大量注册的请求，就可能会出现阻塞，影响用户浏览的体验，这时候我们更希望把任务放到后台进行，那么Celery会是一个合适的选择，Celery是一个分布式的任务队列，负责任务的执行与调度。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-692ae401f9a84105dd9d3b569dc96e96_1440w.jpg)

Celery的架构由三部分组成，消息中间件（message broker），任务执行单元（worker）和任务执行结果存储（task result store）组成。

这里用Redis作为Celery的Broker，负责传递通讯消息。

用pip install flask-celery-helper安装Celery和它的Flask扩展，用pip install redis安装Redis的python扩展。

安装好之后要在配置文件中添加上celery和redis的相关配置。

首先重构目录的结构，把扩展移到extensions文件下，在**init**中用工厂函数初始化应用。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-6e23b69986d84ac33f38df168811430e_1440w.jpg)

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-4e2d271cf7e41d71a4d04fe61dfff7fb_1440w.jpg)

1.在app目录下新建一个tasks目录，里面放的就是Celery要处理的任务文件。

这里新建一个异步发送邮件的任务，用@celery.task装饰之。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-179d7083e3858ff98eb94a43ffd0248c_1440w.jpg)

2.在视图函数中封装一个发送邮件的函数，以及编写用户认证的视图。

认证的方式用带有时间戳的token。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-dfdfe6a9dc92d6496dcd70bb6f2439d3_1440w.jpg)

3.运行Celery。

加上-A参数后Celery会去识别用户自定义的配置文件，后面接一个celery实例所在的模块文件。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-99c6f5c62973f0d7dde2a1ac16f928c4_1440w.jpg)

运行之后，去注册一个账号。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-0d3cb4be9bdf0afbcdd8953e5197c4e5_1440w.jpg)

点击邮件中的激活连接，验证token：

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-59f26cc5a91c3dafe423713f9a88e50b_1440w.jpg)

## **Day 10：编写Dockerfile**

最后一步:编写部署环境的脚本

首先把项目用到的配置文件都放在项目的conf目录下，如下图显示了项目的supervisor的配置文件。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-2c80984d6bc3fe82f4f2bdc0830b0b27_1440w.jpg)

接着编写Dockerfile文件，方便快速地用docker部署好项目的容器环境。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-5ce4e4ab411232be75fa0fd4b92f7cc8_1440w.jpg)

最后就可以在生产的服务器上测试运行项目了。
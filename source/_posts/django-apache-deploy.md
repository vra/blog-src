title: 在Apache上部署Django项目
date: 2016-07-19 16:44:59
tags:
 - Python
 - Django
 - Apache
 - Web编程
---
###0.概述
Django是一个基于Python的web开发框架，在实际生产环境中部署的时候，还需要用Apache容器来部署。这里记录下如何在Debian系统中用Aapche和[mod_wsgi模块](https://pypi.python.org/pypi/mod_wsgi)来部署Django项目。
<!--more-->
###1.系统信息
```bash
$ uname -a  
Linux iZ284ov0vfwZ 3.2.0-4-amd64 #1 SMP Debian 3.2.81-1 x86_64 GNU/Linux  
$ lsb_release -a  
No LSB modules are available.  
Distributor ID: Debian  
Description:    Debian GNU/Linux 7.11 (wheezy) 
Release:        7.11  
Codename:       wheezy  
$ sudo apachectl -v  
Server version: Apache/2.2.22 (Debian)  
Server built:   Aug 18 2015 09:49:50  
```
**我用的是Debian发行版，Apache的配置与别的发行版有较大不同，这里以Debian为例进行说明，别的发行版需要进行一定的修改。**
###2. 安装Django和Apache
Django可以通过如下命令安装:
```bash
sudo pip install Django==1.9.0 #设置版本号为1.9.0
 ```
 Apache通过不同发行版的包管理命令安装。在debian下，是:
 ```bash
 sudo apt-get install apache2
 ```
###3. 安装mod_wsgi模块
mod_wsgi可以通过pip安装，但是需要提前在系统安装`apache-dev`包，但是在Debian发行版上，这个包名叫`apache2-prefork-dev`，详情参考[这里](http://stackoverflow.com/a/16869017/2932001)。通过如下命令安装
```bash 
sudo apt-get install apache2-prefork-dev
```
然后pip 安装mod_wsgi:
```bash
sudo pip install mod_wsgi
```
此外也可以自己编译mod_wsgi。
首先从[这里](https://github.com/GrahamDumpleton/mod_wsgi/releases)下载文件包，然后解压，编译。假设版本是4.5.3，全部命令如下:
```bash
wget https://github.com/GrahamDumpleton/mod_wsgi/archive/4.5.3.tar.gz  
tar -xvf 4.5.3.tar.gz  
cd mod_wsgi-4.5.3  
./configure  
make  
sudo make install  
```
如果没有报错，那么mod_wsgi就编译好了!
**编译好后，会在apache的模块目录`/usr/lib/apache2/modules/`生成mod_wsgi.so文件。**
###4. 写配置文件
Apache的配置文件目录是`/etc/apache2`，下面有好几个文件夹，所有的配置文件都会包含在最终的`apache2.conf`中，所以我们写在.conf的文件默认对全局都起作用，所以这样配置并不规范，但作为一个尝试性的配置，这里姑妄错之。默认使用80端口作为我们网站的监听端口。我们需要在`mods-available`目录下新建`mod_wsgi`的load文件和conf文件，具体操作如下:
```bash
cd /etc/apache2/mod-available  
sudo echo " LoadModule wsgi_module /usr/lib/apache2/modules/mod_wsgi.so" >> wsgi.load  
sudo vi wsgi.conf  
```
然后在`wsgi.conf`中写入如下内容:
```bash
WSGIScriptAlias / /path/to/mysit.com/mysite/wsgi.py  
WSGIPythonPath /path/to/mysite.com  
<Directory /path/to/mysite.com/mysite>  
<Files wsgi.py>  
 	Order deny,allow  
    Allow from all  
</Files>  
</Directory>  
```
**注意对于版本大于2.4的Apache，需要将`<Files>`标签中的两句话改为`Require all granted`。**
此外，静态文件和媒体文件也都由Apache来托管，故还需要将`media`目录和`static`目录的配置路径也加入到wsgi.conf配置文件中，命令如下:
```bash
Alias /media/ /path/to/media/  
Alias /static/ /path/to/static/  
<Directory /path/to/static>  
    Order deny,allow  
    Allow from all  
</Directory>  
<Directory /path/to/media>  
    Order deny,allow  
    Allow from all  
</Directory>  
```
**同样地，对于2.4以上的Apache版本，需要将`Order deny,allow` 和`Allow from all`改为`Require all granted`。**
最后wsgi.conf的内容如下:
```bash
WSGIScriptAlias / /path/to/mysite.com/mysite/wsgi.py
WSGIPythonPath /path/to/mysite.com
<Directory /path/to/mysite.com/mysite>
	<Files wsgi.py>
 		Order deny,allow
    	Allow from all
	</Files>
</Directory>
Alias /media/ /path/to/media/
Alias /static/ /path/to/static/
<Directory /path/to/static>
    Order deny,allow
    Allow from all
</Directory>
<Directory /path/to/media>
    Order deny,allow
    Allow from all
</Directory>
```
Django项目的配置文件`settings.py`也需要修改下面这几个地方:
 1. `DEBUG=False`
 2. `ALLOWED_HOSTS` 需要设置，不可以为空。如果是本地，可以写成`127.0.0.0`，有IP地址的，可以直接写IP地址
 3. `TEMPLATE中`的`DIRS`变量需要设置一个绝对路径，而不能继续用相对路径
###5. 启用wsgi模块
配置完后，启用wsgi模块，重启apache服务器
```bash
sudo a2enmod wsgi
sudo service apache2 restart
```
打开浏览器，输入`localhost`或者IP地址，应该就可以看到你的网站内容了。
如果要想监听别的端口，可以在`ports.conf`里面输入如下几行:
```bash
NameVirtualHost *:8020
Listen 8020
```
这里举例来监听8020端口。之后应该就能在8020端口访问你的网站了。

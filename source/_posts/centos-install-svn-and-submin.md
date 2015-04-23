title: 阿里云centos安装svn和submin
---
#概述
没有找到可以让团队方便使用的云盘，暂时搭建一个svn凑合用一下

svn有三种安装方式

安装方式       	  	| 服务程序 			| 服务协议       	|  用户和密码    				| 授权				| 系统配置
-------------- 		| -------------		| ------------  |---------------------  	| --------------	|----------
svn独立安装    	  	|  svnserve  		|  svn		    |   passwd文件(明文密码)		| authz文件			| svnserve.conf文件
apache+svn安装	  	|  httpd 			|  http WebDAV  |   htpasswd命令(密文密码)	| authz文件			| httpd.conf文件
apache+svn+submin 	|  httpd+ pythonCGI |  http WebDAV  |	WebUI(sqlite3)			| WebUI(authz文件)	| submin2-admin命令

<!--more-->
#安装apache
1. 检查apache是否安装
		
		rpm -qa|grep httpd

2. 使用yum安装apache

		yum -y install httpd

3. 记住安装的版本号
  
  		httpd.x86_64 0:2.4.6-31.el7.centos

4. 启动apache测试apache是否可用
注意：在centos7中使用systemctl替换了service

		systemctl start httpd.service
		systemctl status httpd.service
浏览器输入IP查看是否能显示以下页面
<img src="/img/submin/apache-test.png" width="500" height="300"/>

5. apache安装路径
	/etc/httpd
	
#安装SVN
1. 检查svn是否安装

		rpm -qa|grep subversion
阿里云已经安装了svn，如果没有安装使用 yum install subversion 命令安装

2. 使用命令查看版本
	
		svnserve --version
记住版本号svnserve，版本 1.7.14 (r1542130)


3. 安装apache对svn的支持模块

		yum install mod_dav_svn

		#安装完成后apache的modules目录下会多两个文件
		mod_authz_svn.so
		mod_dav_svn.so	

4. 安装python对svn的支持

		yum install subversion-python

#安装submin
可以参照 https://ssl.supermind.nl/collab/projects/submin/browser/INSTALL

1. submin依赖
	>1. If you want subversion, you also need apache. If only git is needed, you can also install nginx.
	>2. Python 2.x	Python 2.7 preferred, but 2.6 should work,使用python --version查看python 版本
	>3. Subversion
	
2. 下载最新版本 http://supermind.nl/submin/current/submin-2.2.1-1.tar.gz

3. 上传到服务器 sftp

4. 解压文件 
	
		tar -xzvf submin-2.2.1-1.tar.gz
5. 安装
	
		cd submin-2.2.1-1/
		python setup.py install
	
6. 验证安装 
	执行 submin2-admin 成功显示当前版本
	
7. 配置submin
	
		submin2-admin /opt/submin/ initenv your@email.address
邮箱很重要， submin会将管理员设置初始口令的链接发到这邮箱中
/opt/submin这个目录不要提前建，安装命令的向导一步步设置就可以了，说明很清楚.
这一步需要注意

> Please provide a location for the Subversion repositories. For new Subversion
repositories, the default setting is ok. If the path is not absolute, it will
be relative to the submin environment. If you want to use an existing
repository, please provide the full pathname to the Subversion parent
directory (ie. /var/lib/svn).
	Path to the repository? [svn]>
	
这个目录我设置的时 /opt/svn,注意这个目录apache一定要有写权限,否则会报以下错误

> E165002 /opt/svn is an existing repository
	
因为submin是用apache用户启动的，最简单的方式是将该目录所有者设置为apache，执行以下命令
		
		chown apache:apache  /opt/svn/

8. 配置apache
	生成配置文件 
	
		submin2-admin /opt/submin/ apacheconf create all
		
	建立软链接配置apache,注意Apache版本
		
		ln -s /opt/submin/conf/apache-2.4-webui-cgi.conf /etc/httpd/conf.d/
		ln -s /opt/submin/conf/apache-2.4-svn.conf /etc/httpd/conf.d/
		
9. 重启apache
	
		systemctl restart httpd.service
报错 Can't load driver file apr_dbd_sqlite3.so
submin2默认需要sqlite3做数据库

		yum -y install apr-util-sqlite apr-util
	再次重启OK
	
#邮箱设置
1. 配置 submin时,需要配置管理员邮箱
2. /usr/lib/python2.7/site-packages/submin/email/fallback.py

		def sendmail(sender, receiver, message):
        msg_e = message.encode('utf-8')
        try:
                smtp.send(sender, receiver, msg_e)
        except SendEmailError:
                # this can still raise SendEmailError
                local.send(sender, receiver, msg_e)
                
   优先使用stmp发邮件。 异常时使用本地的sendmail,配置smtp
   
   		submin2-admin /opt/submin  config set smtp_hostname  smtp.exmail.qq.com
		submin2-admin /opt/submin  config set smtp_port 25
		submin2-admin /opt/submin  config set smtp_username svn@xxxxx.com
		submin2-admin /opt/submin  config set smtp_password  xxxxxx
		submin2-admin /opt/submin  config set smtp_from "svn <svn@xxxx.com>"
		submin2-admin /opt/submin  config set commit_email_from "svn <svn@xxxx.com>"

#诊断submin
执行以下命令
		submin2-admin /opt/submin/ diagnostics
如果有问题参照说明修改对应错误
我设置出现了以下问题：

> To disable, run the following command: submin2-admin /opt/submin config set vcs_plugins svn

如果不禁用git，以后的操作都会报git没有设置的错误

#管理员重置密码

1. 访问系统进入登录界面

<img src="/img/submin/submin-login.png" width="300" height="200"/>

2. 点击forgot your password

<img src="/img/submin/admin-reset.png" width="200" height="150"/>

输入admin，点击重置,以下命令配置的邮箱将会受到密码重置邮件

		submin2-admin /opt/submin/ initenv your@email.address
		
3. 点击重置邮件进入系统,点击admin菜单进入用户设置界面，修改密码

<img src="/img/submin/admin-change-password.png" width="400" height="200"/>

#新建仓库sharing
1. 点击左侧菜单右下角的新建仓库按钮

<img src="/img/submin/new-repo.png" width="120" height="100"/>


2. 进入新建页面

<img src="/img/submin/new-repo2.png" width="300" height="150"/>
输入名称，选择svn创建仓库

#授权
1. 点击左侧菜单最下面中间两个人的按钮，新建组

	<img src="/img/submin/new-group.png" width="300" height="150"/>
2. 点击左侧菜单最下面的左侧一个人的按钮，新建用户
	
	<img src="/img/submin/new-user.png" width="300" height="150"/>
3. 新建完成用户后，用户会收到密码重置邮件，同时系统进入修改用户信息页面，可以将用户添加到developer组

	<img src="/img/submin/add-member.png" width="200" height="100"/>
	
4. 设置权限，点击左侧需要授权的仓库按钮
	
	<img src="/img/submin/set-permission.png" width="500" height="300"/>
	
针对特定的路径设置组或用户并设置对应的读写权限，这里我给developer设置root的读写权限

5. 使用浏览器访问svn地址测试

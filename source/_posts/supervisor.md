title: 阿里云  Supervisor 使用笔记
date: 2015-11-19

---
# 概述
supervisor 提供了服务异常关闭时能自动重启的功能

# 安装
pip install supervisor

# 配置
<!--more-->

1.复制默认文件

echo_supervisord_conf > /etc/supervisor/supervisord.conf

2.注意

UNIX Domain Socket 

```
[unix_http_server]

file=/tmp/supervisor.sock   ; (the path to the socket file)

[supervisorctl]

serverurl=unix:///tmp/supervisor.sock ; use a unix:// URL  for a unix socket

```

3.编辑文件

```

将以下内容

[include]

files = relative/directory/*.ini;

修改为

[include]

files = /etc/supervisor/conf.d/*.conf

```

4.创建软连接

ln -s /etc/supervisor/supervisord.conf supervisord.conf


官网说明： 
![supervisord.conf](/img/supervisor/supervisord.conf.png)

5.创建配置子文件tomcat-8-tools.conf

vim /etc/supervisor/conf.d/tomcat-8-tools.conf

```
[program:tomcat-8-tools]

command=/opt/apache-tomcat-8-tools/bin/catalina.sh run

autostart=true

autorestart=true

stopsignal=INT

stdout_logfile=/opt/apache-tomcat-8-tools/logs/supervisor.log

user=jenkins

environment=HOME="/home/jenkins",USER="jenkins",JAVA_HOME="/opt/jdk1.7.0_79",M2_HOME="/opt/apache-maven-3.3.3",JENKINS_HOME="/opt/jenkins-workspace",GRADLE_HOME="/opt/gradle-2.8",ANDROID_HOME="/opt/android-sdk-linux/",PATH="$PATH:/bin:/usr/bin:/usr/local/bin:$HOME/.local/bin:$HOME/bin:$JAVA_HOME/bin:$M2_HOME/bin:/opt/qiniu-devtools/:$GRADLE_HOME/bin:$ANDROID_HOME/tools:$ANDROID_HOME/platform-tools"
```

# supervisord  开机启动

1.linux init知识

man init

systemd upstart sysV

centos 7 systemd systemctl


2.systemctl命令

systemctl enable supervisord

systemctl start supervisord

systemctl status supervisord


3.文件supervisord.service

vim /usr/lib/systemd/system/supervisord.service

```
[Unit]

Description=supervisord - Supervisor process control system for UNIX

Documentation=http://supervisord.org

After=network.target


[Service]

Type=forking

ExecStart=/usr/bin/supervisord -c /etc/supervisor/supervisord.conf

ExecReload=/usr/bin/supervisorctl reload

ExecStop=/usr/bin/supervisorctl shutdown

[Install]

WantedBy=multi-user.target
```

# supervisorctl命令

```
supervisorctl start tomcat-8-tools

supervisorctl restart tomcat-8-tools

supervisorctl stop tomcat-8-tools

supervisorctl status tomcat-8-tools
```


# sudo配置

**文件supervisor**

visudo -f /etc/sudoers.d/supervisor

```
Cmnd_Alias TOMCAT_TOOLS_CMDS = /usr/bin/supervisorctl start tomcat-8-tools, /usr/bin/supervisorctl restart tomcat-8-tools, /usr/bin/supervisorctl stop tomcat-8-tools, /usr/bin/supervisorctl status tomcat-8-tools

jenkins ALL = (root) NOPASSWD: TOMCAT_TOOLS_CMDS
```

# sudo注意问题

>ssh远程执行 sudo ，执行时提示如下错误：
>sudo: sorry, you must have a tty to run sudo
>导致这问题的原因是 sudo默认需要在 tty终端里才能正确被调用，我们可以通过修改 /etc/sudoers配置文件来解决这个问题：
>visudo /etc/sudoers
>注释掉 Default requiretty 一行
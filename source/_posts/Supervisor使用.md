---
title: 使用Supervisor部署Django项目
date: 2020-12-22 18:38:49
tags: [Django,supervisor]
categories: Python
---

supervisor是一个python开发的、通用的进程监控、管理工具，可以将一个普通的命令行进程变为后台daemon,并且监控进程状态，异常自动重启。相较于nohup的方式，这种部署方式使用起来更方便简洁，并且运行也更稳定。

<!--more-->

## 1.安装
`sudo apt-get install -y supervisor`
## 2.创建项目
`cd /etc/supervisor/conf.d/`
`sudo vim mipin.conf`


- uwsgi启动配置示例：
```bash
[program:go2home]
directory = /home/ubuntu/go2home/GoToHome ;程序的启动目录
command= /home/ubuntu/.pyenv/versions/go2home/bin/uwsgi --ini uwsgi.ini ;启动命令
autostart = true     ; 在 supervisord 启动的时候也自动启动
startsecs = 5        ; 启动 5 秒后没有异常退出，就当作已经正常启动了
autorestart = true   ; 程序异常退出后自动重启
startretries = 3     ; 启动失败自动重试次数，默认是 3
user = ubuntu          ; 用哪个用户启动
redirect_stderr = true  ; 把 stderr 重定向到 stdout，默认 false
stdout_logfile_maxbytes = 20MB  ; stdout 日志文件大小，默认 50MB
stdout_logfile_backups = 20     ; stdout 日志文件备份数
;stdout 日志文件，需要注意当指定目录不存在时无法正常启动，所以需要手动创建目录(supervisord 会自动创建日志文件)
stdout_logfile = /home/ubuntu/go2home/logs/stdout.log
;然后确保杀死主进程后，子进程也可以停止
stopasgroup=true
killasgroup=true
```

- daphne启动配置示例：
```bash
[program:mipin]
directory=/home/ubuntu/mipin/MiPin
command=/home/ubuntu/.virtualenvs/mipin/bin/daphne -b 127.0.0.1 -p 8090 --proxy-headers MiPin.asgi:application
autostart=false
autorestart=false
stdout_logfile=/home/ubuntu/mipin/logs/out.log
loglevel=debug
redirect_stderr=true
[supervisord]
```

- 启动项目:

`sudo supervisord -c mipin.conf`


## 3.supervisor基本操作
```
# 停止某一个进程，program_name 为 [program:x] 里的 x
supervisorctl stop program_name
# 启动某个进程
supervisorctl start program_name
# 重启某个进程
supervisorctl restart program_name
# 结束所有属于名为 groupworker 这个分组的进程 (start，restart 同理)
supervisorctl stop groupworker:
# 结束 groupworker:name1 这个进程 (start，restart 同理)
supervisorctl stop groupworker:name1
# 停止全部进程，注：start、restart、stop 都不会载入最新的配置文件
supervisorctl stop all
# 载入最新的配置文件，停止原有进程并按新的配置启动、管理所有进程
supervisorctl reload
# 根据最新的配置文件，启动新配置或有改动的进程，配置没有改动的进程不会受影响而重启
supervisorctl update
```




## 4.问题

- `error: <class 'socket.error'>, [Errno 2] No such file or directory: file: /usr/lib/python2.7/socket.py line: 228`

使用命令 `supervisord -c /etc/supervisor/supervisord.conf`启动
查看状态   `sudo supervisorctl status`

- 权限不足

`sudo vim /etc/supervisor/supervisord.conf`
在`[unix_http_server]` 和 `[unix_http_server]` 2处配置下，加入：
username=name
password=password


## 5.参考链接
[https://www.cnblogs.com/xiao987334176/p/11329923.html](https://www.cnblogs.com/xiao987334176/p/11329923.html)

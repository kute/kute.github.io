---
published: true
layout: post
title: ssh连接以及sftp下载上传模块-sshconnctor
category: Python
tags: Python ssh sftp
time: 2017.01.05 14:22:00
excerpt: 方便ssh连接以及sftp服务器日志下载
keywords: 
description: 

---

## 介绍

  `sshconnector`是基于`paramiko`编写的ssh终端连接以及sftp下载上传脚本模块.
  使用场景与特点:
  - 批量连接多台服务器, 执行命令或者下载上传文件(我经常是下载日志)
  - 可使用`密码`或者`公私钥`认证连接
  - 与[Eventor](https://kute.github.io/2017/01/05/Python-Eventor.html)结合加快日志下载上传
  
## 例子

  认证连接
  
    >>> import sshconnector
    >>> sshconnector = SSHConnector(host="192.168.157.2", port=22, username="root", password="root",
                                    pub_key_file_path="/path/of/my-private-openssh-key")
                                   
  执行命令
  
    sshconnector.execute_command(command="ls -l")
    
  下载服务器日志
  
    >>> sshconnector.sftp_get("/remote/server/save/file", "/home/my/local/file")
    
  使用协程加快下载(目前是没有集成Eventor, 后续更改下集成)
  
    >>> import SFTPMutilthread
    >>> serverdatasource = [("192.168.157.1", 22), ("192.168.157.2", 22), ("192.168.157.3", 22), ]
    >>> remotefile = "/remote/file"
    >>> localfile = "E:/save/local/file"
    >>> mutilhelper = SFTPMutilthread(serverdatasource, remotefile, localfile, "root", "root", "/path/of/my-private-openssh-key")
    >>> mutilhelper.start()

  
项目地址以及API: [https://github.com/kute/sshconnector/](https://github.com/kute/sshconnector/)

持续更新....
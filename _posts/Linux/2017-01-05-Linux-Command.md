---
published: true
layout: post
title: 常用的Linux命令脚本
category: Linux
tags: Command
time: 2017.01.05 14:22:00
excerpt: 记录常用的Linux命令，主要偏向文本处理，服务器监控等。
keywords: 
description: 

---


#### 1. linux添加path脚本函数

  
    prepend() { [ -d "$2" ] && eval $1=\"$2\$\{$1:+':'\$$1\}\" && export $1 ; }
  
将此函数写入 .bashrc中，使用：
    
    prepend PATH /myapp/conf
    
#### 2. 标准错误输出
    
    o---stdin
    1---stdout
    2---stderr
    > 等同于 1>
    >> 等同于 1>>
    
将错误转换为标准输出，重定向到文件
    
    cmd 2>&1 stdout.txt
    cmd &> stdout.txt
    
错误重定向到 `/dev/null`
    
    cmd 2> /dev/null
    
重定向文本到文件，并复制一份副本用作管道stdin(tee只能从stdin中读取)
    
    cat "words" | tee stdout.txt | head -n
    
#### 3. ssh 公钥，本地、远程端口转发
公钥链接服务器
    
    ssh username@host -p port -i /path/of/privatekey
    
本地端口转发（transhost是转发的机器）
    
    ssh -L localport:targethost:targetport transhost
    ssh -L localport:targethost:targetport transusername@transhost -p thransport -i /path/of/thranshost/privatekey
    
开启本地端口后可以直接连本地端口进而通过转发连接到目标机器
    
    ssh localhost -p localport
    
#### 4. 在终端输入密码不直接显示密码文本(stty)
    
    #!/bin/sh
    echo "please enter password"
    stty -echo # 禁止将输出发送到终端
    read mypwd
    stty echo
    echo
    echo "pwd is readed", $mypwd
    
#### 5. fork炸弹（不断生成子进程，后台运行，很危险，可修改件/etc/security/limits.conf来限制可生成的最大进程数来避开）
    
    :(){ :|:& };:
    
#### 6. 管道以及子shell以及反引用
    
    管道：    ls | cat -n > stdout.txt
    子shell： cmd_output = $(commands) 如 cmd_output = $(ls | cat -n)
    反引用：  cmd_output = `ls | cat -n`
    
#### 7. 重复执行命令直至成功
方式一：
    
    repeat() { while true; do $@ && return; done }
    
更高效的方式：
    
    repeat() { while :; do $@ && return; done }
    
加入延时，防止服务器拉黑
    
    repeat() { while :; do $@ && return; sleep 30; done }
    
使用方式：
    
    repeat wget -c http://www.example.com/softer-1.0.0.tar.gz
    
#### 8. 操作符（字符串比较最好使用双中括号）
    
    -gt: 大于
    -lt: 小于
    -ge: 大于等于
    -lt: 小于等于
    -ne: 不等于
    -eq: 等于
    [ -f $file_var ]：如果给定的变量包含正常的文件路径或文件名，则返回真。
    [ -x $var ]：如果给定的变量包含的文件可执行，则返回真。
    [ -d $var ]：如果给定的变量包含的是目录，则返回真。
    [ -e $var ]：如果给定的变量包含的文件存在，则返回真。
    [ -c $var ]：如果给定的变量包含的是一个字符设备文件的路径，则返回真。
    [ -b $var ]：如果给定的变量包含的是一个块设备文件的路径，则返回真。
    [ -w $var ]：如果给定的变量包含的文件可写，则返回真。
    [ -r $var ]：如果给定的变量包含的文件可读，则返回真。
    [ -L $var ]：如果给定的变量包含的是一个符号链接，则返回真。
    
#### 9. 常用命令
> （find / grep / sort / cut / awk / sed / uniq / tee / tr / diff / cmp / split / xargs/ echo/ cat/ tail/ head/ chmod/ chown）

    
    find . -name "*.txt"
    find . ! -name "*.txt" # 否定匹配
    find . -maxdepth 1 -name "*.txt" # 限制查找深度匹配
    find . -maxdepth 1 -type d # 文件类型匹配， d: 目录，f：文件
    # -atime 访问时间；-mtime 修改时间；-ctime 变化时间，单位：天，配合  + -:- 表示小于，+ 表示大于
    # -amin；-mmin；-cmin  单位：分钟
    find . -type f -atime -7 # 最近7天内被访问的文件
    find . -type f -atime 7 # 在7天前被访问的文件
    find . -type f -newer file.txt # 比file.txt文件修改时间更近的文件
    find . -type f -size +2k # 大于2k的文件
    find . -iname "abc*.txt" # 忽略大小写
    find . \( -name "*.txt" -o -name "*.pdf" \) # 匹配任意一个,注意空格
    find . -path "*/mypath/*.txt" # 匹配路径
    find . -type f -name "*.swap" -delete #删除匹配文件
    find . -type f -name "*.txt" -exec cat {} \;  # 对匹配文件执行命令操作，命名可以使shell脚本
    find . "*.txt" -print0 | xargs -0 rm -f #找出匹配的文件并删除
    find . -name "*.txt" -print0 | xargs -0 wc -l # 统计匹配的每个文件的行数以及总行数
    echo "asdKJAIKAJAjkasdf" | tr 'a-z' 'A-Z' # 大小写转换
    echo "a kjad9 akj23k 23j21jksd9u" | tr -d '{0-9}' # 删除指定集合的字符
    echo "a kjad9 akj23k 23j21jksd9u" | tr -d -c '{0-9}' # 删除指定集合的补集的字符
    echo "aaa   sd   eew   sdf    as" | tr -s ' ' # 压缩指定集合中的重复字符
    cat myfile | sort | uniq -c | sort -r | more # 统计每行出现的次数，倒序输出
    cat myfile | sort | uniq -d # 输出重复行
    tail -n +100 data.txt | more # 从100行开始查看data.txt文件
    filename = `mktemp`  # 创建临时文件
    filedir = `mktemp -d` # 创建临时目录
    split -b 20k data.txt -d data_prefix # 将data.txt分隔为每个20k大小的文件,并以 数字后缀命名
    split -l 50000 data.txt data_prefix # 分隔文件，每个文件 50000 行
    filename="mypng.jepg"; echo ${filename%.*} # 提取文件名称： mypng
    filename="mypng.jepg"; echo ${filename#*.} # 提取文件后缀：jepg
    head -n 80 /dev/urandom | tr -dc 0-9 | head -c 20 # 纯数字20位长度的随机数
    head -n 80 /dev/urandom | tr -dc 0-9a-zA-Z | head -c 20 # 字母大小写+数字的20位长度的随机数
    LC_CTYPE=C tr -dc A-Za-z0-9 < /dev/urandom | head -c 20 # mac版本下的方式
    sudo chown $(whoami) /usr/local/bin  # 更改 /usr/local/bin的所属用户
    

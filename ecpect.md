# expect
## 概述
我们通过Shell可以实现简单的控制流功能，如：循环、判断等。但是对于需要交互的场合则必须通过人工来干预，有时候我们可能会需要实现和交互程序如telnet服务器等进行交互的功能。而expect就使用来实现这种功能的工具。  
expect是一个免费的编程工具语言，用来实现自动和交互式任务进行通信，而无需人的干预。expect是不断发展的，随着时间的流逝，其功能越来越强大，已经成为系统管理员的的一个强大助手。expect需要Tcl编程语言的支持，要在系统上运行expect必须首先安装Tcl。  

## expect的安装
```
[root@manager ~]# yum install expect -y
```

## expect的使用
### expect的工作流程
expect的工作流程可以理解为：spawn启动进程->>expect期待关键字->>send向进程发送字符->>退出结束

### expect的简单使用
```
[root@manager expect]# cat test1.sh 
#!/usr/bin/expect
spawn ssh root@192.168.1.31 ifconfig ens33
set timeout 60
expect "*password:"
send "1qaz!QAZ\n"
expect eof
exit

[root@manager expect]# expect test1.sh 
spawn ssh root@192.168.1.31 ifconfig ens33
root@192.168.1.31's password: 
ens33: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.1.31  netmask 255.255.255.0  broadcast 192.168.1.255
        inet6 fe80::20c:29ff:fe2a:4de5  prefixlen 64  scopeid 0x20<link>
        ether 00:0c:29:2a:4d:e5  txqueuelen 1000  (Ethernet)
        RX packets 64987  bytes 95343538 (90.9 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 6922  bytes 708207 (691.6 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

```

### expect 语法
#### spawn
apawn命令是expect的初始命令。它用于启动一个进程，之后所有expect操作都在这个进程中进行，如果没有spawn语句，整个expect就无法执行
```
spawn ssh root@192.168.1.31 ifconfig ens33
```
在spawn命令后面，直接加上要启动的进程、命令等信息。除此之外，spawn还支持其他选项如：  
- -open    启动文件进程  
- -ignore	忽略某些信号

#### expect
使用方法：expect 表达式 动作 表达式 动作 ......  

expect命令用于等候一个相匹配内容的输出，一旦匹配上就执行expect后面的动作或命令，这个命令接受几个特有参数，用的最多的就是-re，表示正则表达式的方式匹配，使用起来像这样：
```
spawn ssh root@192.168.1.31
expect "*password:" ｛send "1qaz!QAZ\r"｝
```
expect是依附与spawn命令的，当执行ssh命令后，expect就匹配命令执行后的关键字："password:"，如果匹配到关键字就会执行后面包含在{}括号中exp_send动作，匹配以及动作可以放在两行，这样就不需要使用{}括号了，比如这样：
```
spawn ssh root@192.168.1.31
expect "*password:"
send "1qaz!QAZ\r"
```
expect命令还有一种使用方法，它可以在一个expect匹配中多次匹配关键字，并给出处理动作，只需要将关键字放在一个大括号中就可以了，当然还要有exp_continue。
```
spawn ssh root@192.168.1.32 ifconfig ens33
expect {
	"yes/no" {exp_send "yes\r"; exp_continue }
	"*password:" {exp_send "1qaz!QAZ\r";}
}
```

#### exp_send和send
exp_send命令是expect中的动作命令，它还有一个完成共同工作的同胞:send。exp_send可以发送一些特殊符号，如\r(回车)，\n(换行)，\t(制表符)等等，这些都与TCL中的特殊符号相同。
```
spawn ssh root@192.168.1.32 ifconfig ens33
expect {
	"yes/no" {send "yes\r"; exp_continue }
	"*password:" {exp_send "1qaz!QAZ\r";}
}
```
sed命令有几个可用的参数  
- -i 	指定spawn_id，这个参数用来向不同的spawn_id的进程发送命令，是进行多程序控制的关键参数。  
- -s 	s代表slowly，也就是控制发送的速度，这个参数使用的时候要与expect中的变量send_slow相关联。

#### exp_continue
这个命令一般用在动作中，它被使用的条件比较苛刻。
```
#!/usr/bin/expect
spawn ssh root@192.168.1.32 ifconfig ens33
set timeout 60
expect {
	"yes/no" {exp_send "yes\r"; exp_continue }
	"*password:" {exp_send "1qaz!QAZ\r";}
}
expect eof
exit
```

#### send_user
send_user命令用来把后面的参数输出到标准输出中去，默认的send、exp_send命令都是将参数输出到程序中去的，用起来像这样：
```
send_user "Please input password:"
```
这句语句就可以在标准输出中打印Please input password:字符了
```
[root@manager expect]# cat test4.sh 
#!/usr/bin/expect
if { $argc != 3 } {
    send_user "usage:expect scp-expect.exp file host dir\n"
    exit
}

#define var
set file [lindex $argv 0]
set host [lindex $argv 1]
set dir [lindex $argv 2]
set password "1qaz!QAZ"
spawn scp $file root@$host:$dir
expect {
	-timeout 5
	"yes/no" {send "yes\r";exp_continue}
	"*password:" {send "$password"}
	timeout {puts "expect connect time,pls connect jichengxi."; return}
}
expect eof
exit -onexit {
	send_user "bye bye"
}

[root@manager expect]# expect test4.sh 
usage:expect scp-expect.exp file host dir
```

#### exit
exit命令很简单，就是直接退出脚本，但是你可以利用这个命令对脚本做一些扫尾工作，比如下面这样：
```
exit -onexit {
	send_user "bye bye"
}
```

### expect 变量
expect中有很多有用的变量，他们的使用方法和TCL语言中的变量相同，比如：  
set 变量名 变量值 #设置变量的方法  
puts $变量名 #读取变量的方法
```
#define var
set file [lindex $argv 0]
set host [lindex $argv 1]
set dir [lindex $argv 2]
set password "1qaz!QAZ"
```

### expect 关键字
#### eof
eof(end-of-file)关键字用于匹配结束符，比如文件的结束符，ftp传输停止等情况，在这个关键字后跟上动作来进行进一步的控制，特别是ftp交互操作方面，它的作用很大，用一个例子看看：
```
spawn ftp test@192.168.1.31
expect {
	"password:" {exp_send "test1"}
	eof {ftp connect close}
}
interact {}
```

#### timeout
timeout是expect中一个重要变量，它是一个全局性的时间控制开关，可以通过为这个变量赋值来规定整个expect操作的时间，注意这个变量是服务与expect全局的，它不会纠缠于某一条命令，即使命令没有任何错误，到时间仍然会激活这个变量，但这个时间到达以后除了激活一个开关之外不会做其他事情，如何处理是脚本编写人员的事情，看看它的实际用法：
```
spawn ssh root@192.168.1.32 ifconfig ens33
set timeout 60
expect {
	"*password:" {exp_send "1qaz!QAZ\r";}
}
expect {
	timeout {puts "expect was timeout"; return}
}
```
在上面的处理中，首先将timeout变量设置为60秒，当出现问题的时候程序可能会停止下来，只要到60秒，就会激活下面的timeout动作。  
在另一种expect格式中，还有一种设置timeout变量的方法，看看下面的例子：
```
spawn ssh root@192.168.1.32 ifconfig ens33
expect {
        -timeout 60
        -re "*password:" {exp_send "1qaz!QAZ\r";}
        timeout {puts "expect was timeout"; return}
}
```
在expect命令中间加上一个小横杠，也可以设置timeout变量。  
**timeout变量中，设置为0表示立即超时，设置为-1则表示永不超时。**
```
if { $argc != 3 } {
    send_user "usage:expect scp-expect.exp file host dir\n"
    exit
}
#define var
set file [lindex $argv 0]
set host [lindex $argv 1]
set dir [lindex $argv 2]
set password "1qaz!QAZ"
spawn scp $file root@$host:$dir
expect {
	-timeout 5
	"yes/no" {send "yes\r";exp_continue}
	"*password:" {send "$password"}
	timeout {puts "expect connect time,pls connect jichengxi."; return}
}
expect eof
exit -onexit {
	send_user "bye bye"
}
```

### 批量分发ssh_key
```
[root@manager expect]# cat fenfa_ssh.sh 
#!/bin/bash
for i in `cat ip_list`
do
    /usr/bin/expect -c "
	spawn ssh $i mkdir -p /root/.ssh
	expect {
		\"yes/no\" {send \"yes\r\"; exp_continue}
		\"*password:\" {send \"1qaz!QAZ\r;\"; exp_continue}
	}
	spawn scp /root/.ssh/id_rsa.pub $i:/root/.ssh/
	expect {
                \"yes/no\" {send \"yes\r\"; exp_continue}
                \"*password:\" {send \"1qaz!QAZ\r;\"; exp_continue}
        }
        spawn ssh $i mv /root/.ssh/id_rsa.pub /root/.ssh/authorized_keys
                expect {
                \"yes/no\" {send \"yes\r\"; exp_continue}
                \"*password:\" {send \"1qaz!QAZ\r;\"; exp_continue}
        }
    "
done
```


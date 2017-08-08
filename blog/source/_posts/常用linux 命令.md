title: linux常用命令

date: 2015-02-14

categories: [linux] 

---

> linux 常用命令整理

<!--more-->

### 一般命令
* more
```
空格：显示下一页
Enter：显示下一行
/<字符串> : 查找字符串
q：退出
:f ： 显示当前行数和文件名
```
* less
```
空格：显示下一页
Enter：显示下一行
/<字符串> : 查找字符串
q：退出
:f ： 显示当前行数和文件名
```
* touch  新建文件
* locate 查找文件
```
locate -i <文件名or部分文件名> （忽略大小写）
```
* find 查找文件
```
find <path> -mtime 0   查找24小时之内的指定目录被修改过的文件
``` 
* chmod 改变文件属性
```
chmod <属性值> <filename>
```


### 进程相关 ps

各参数意义如下：
```
-A 显示所有进程（等价于-e）(utility)
-a 显示一个终端的所有进程，除了会话引线
-N 忽略选择。
-d 显示所有进程，但省略所有的会话引线(utility)
-x 显示没有控制终端的进程，同时显示各个命令的具体路径。dx不可合用。（utility）
-p pid 进程使用cpu的时间
-u uid or username 选择有效的用户id或者是用户名
-g gid or groupname 显示组的所有进程。
U username 显示该用户下的所有进程，且显示各个命令的详细路径。如:ps U zhang;(utility)
-f 全部列出，通常和其他选项联用。如：ps -fa or ps -fx and so on.
-l 长格式（有F,wchan,C 等字段）
-j 作业格式
-o 用户自定义格式。
v 以虚拟存储器格式显示
s 以信号格式显示
-m 显示所有的线程
-H 显示进程的层次(和其它的命令合用，如：ps -Ha)（utility）
e 命令之后显示环境（如：ps -d e; ps -a e）(utility)
h 不显示第一行

```
* 根据进程名称查看进程信息
```
ps -ef | grep <pName>  or

ps-aux | grep <pName>
```
* 根据进程号查看进程信息
```
ps -ef | grep <pid>   or
ps -aux | grep <pid>
```
### 端口相关  netstat
其相关参数如下：
```
-a (all)显示所有选项，默认不显示LISTEN相关
-t (tcp)仅显示tcp相关选项
-u (udp)仅显示udp相关选项
-n 拒绝显示别名，能显示数字的全部转化成数字。
-l 仅列出有在 Listen (监听) 的服務状态

-p 显示建立相关链接的程序名
-r 显示路由信息，路由表
-e 显示扩展信息，例如uid等
-s 按各个协议进行统计
-c 每隔一个固定时间，执行该netstat命令。
```
* 根据进程号 查看其占用端口
```
netstat -pt | grep --color <pid>

```
* 根据端口号查看
```
netstat -pt | grep --color <port>
```
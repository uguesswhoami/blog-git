title: git命令

date: 2015-04-01

categories: [git] 

---

> git是非常好用的代码托管工具,github也是世界上最伟大的开源社区,熟悉git命令会给平时的工作带来事半功倍的效果。

 <!--more-->

* 生成ssh公钥、秘钥

  > 关于ssh的作用可以[点这里](https://segmentfault.com/q/1010000000118744)
 
  ```
  ssh-keygen -t rsa -C "your name"
  ```
 
*  修改全局的用户名、邮箱
 ```
 git config --global user.name "xxx"
 
 git config --global user.email "xxx@xxx.xx"
 ```
 
* 修改某repo用户名、邮箱, 进入repo下：
 ```
 git config  user.name "xxx"
 
 git config  user.email "xxx@xxx.xx"
 ```
 
 * 查看用户名、邮箱等配置信息
 ```
 git config --list
 ```
 
 * 克隆远程代码到本地
 ```
 git clone <ssh/http路径>
 ```

* 添加文件
 ```
 git add <filename>
 ```
 
 * 添加当前目录所有文件
 ```
 git add .
 ```
 
 * 提交文件到本地
 ```
 git commit -m "xxx"
 ```
 
  * 查看当前分支状态
 ```
 git status
 ```
 
 * 提交文件到服务器
 ```
 git push 
 ```
 
 * 更新远程分支到本地
 ```
 git fetch
 ```
 
  * 拉取远程分支到本地
 ```
 git pull orgin/<分支名>
 ```
 
 * 合并分支到当前分支
 ```
 git merge <分支名称>
 ```
 

 
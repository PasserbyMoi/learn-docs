# [linux-Centos7安装python3并与python2共存](https://www.cnblogs.com/JahanGu/p/7452527.html)



1.查看是否已经安装Python

CentOS 7.2 默认安装了python2.7.5 因为一些命令要用它比如yum 它使用的是python2.7.5。

使用 python -V 命令查看一下是否安装Python

然后使用命令 which python 查看一下Python可执行文件的位置

![img](https://images2017.cnblogs.com/blog/1226698/201708/1226698-20170830120844358-1686107415.png)

可见执行文件在/usr/bin/ 目录下，切换到该目录下执行 ll python* 命令查看 

![img](https://images2017.cnblogs.com/blog/1226698/201708/1226698-20170830121355483-715442589.png)

python 指向的是python2.7

因为我们要安装python3版本，所以python要指向python3才行，目前还没有安装python3，先备份,备份之前先安装相关包，用于下载编译python3

yum install zlib-devel bzip2-devel openssl-devel ncurses-devel sqlite-devel readline-devel tk-devel gcc make

不能忽略相关包，我之前就没有安装readline-devel导致执行python模式无法使用键盘的上下左右键；

然后备份

mv python python.bak

2.开始编译安装python3

去官网下载编译安装包或者直接执行以下命令下载

wget https://www.python.org/ftp/python/3.6.2/Python-3.6.2.tar.xz

解压

tar -xvJf  Python-3.6.2.tar.xz

切换进入

cd Python-3.6.2

编译安装

./configure prefix=/usr/local/python3

make && make install



如果此时报错 ModuleNotFoundError: No module named '_ctypes'，是因为3.7版本需要一个新的包libffi-devel，安装此包之后再次进行编译安装即可。

    yum install libffi-devel -y （包在base）
---------------------
若在安装前移除了/usr/bin下python的文件链接依赖，此时yum无法正常使用，需要自己下载相关软件包安装，为节省读者时间，放上链接

#wget http://mirror.centos.org/centos/7/os/x86_64/Packages/libffi-devel-3.0.13-18.el7.x86_64.rpm

#rpm -ivh libffi-devel-3.0.13-18.el7.x86_64.rpm

*官网网址：https://pkgs.org/download/libffi-devel*



安装完毕，/usr/local/目录下就会有python3了

因此我们可以添加软链到执行目录下/usr/bin

ln -s /usr/local/python3/bin/python3 /usr/bin/python

![img](https://images2017.cnblogs.com/blog/1226698/201708/1226698-20170830123437999-1379416720.png)

可以看到软链创建完成

测试安装成功了没，执行

python -V  看看输出的是不是python3的版本

执行python2 -V  看到的就是python2的版本

因为执行yum需要python2版本，所以我们还要修改yum的配置，执行：

vi /usr/bin/yum

把#! /usr/bin/python修改为#! /usr/bin/python2

![img](https://images2017.cnblogs.com/blog/1226698/201708/1226698-20170830123843405-856911492.png)

 

同理 vi /usr/libexec/urlgrabber-ext-down 文件里面的#! /usr/bin/python 也要修改为#! /usr/bin/python2

 

这样python3版本就安装完成；同时python2也存在

python -V   版本3 

python2 -V 版本2
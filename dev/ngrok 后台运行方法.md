# ngrok 后台运行方法

ngrok 用 & 不能后台运行 

这就要使用screen这个命令

首先安装screen

apt-get install screen

之后运行 screen -S 任意名字（例如：keepngork）

然后运行ngrok启动命令

最后按快捷键 ctrl+A+D 既可以保持ngrok后台运行
```table-of-contents
```
# 背景
## 前提
在服务器上生成ssh-key以后，需要把公钥放在github上，但是，这个公钥只能放在一个账户里，如果放在第二个账户里，就会提示这个key已经被用了，这是前提。

## 场景
你们公司有好几个项目都托管在git上，然后有一台公用的服务器，这时候，你的同事在服务器上搞了一套ssh-key，添加到他的项目里用于ssh验证拉代码了，这时候你也用到这台服务器，然后把公钥往自己的项目里加的时候，发现加不进去了，这时候最暴力的解决方法就是，你重新生成一套加在自己的仓库里，但是这么搞的化，你同事之前添加的那个肯定是不能用了，然后他发现自己的不能用了，上了服务器一看key被改了，他又重新生成一下，然后你又不能用了，如此反复，相爱相杀。


# 解决
## 新建个人用户
```c
  1) 创建：
  mv /home/sam /home/sam-bak
  userdel sam
  useradd –d  /home/sam -m sam 或者 默认 useradd sam

  检查：
  cat /etc/passwd
  ls -al /home/sam
  
  2）sudo 权限

  3）切换用户 su:
  切换: [sudo] su - sam
```

## 生成自己的ssh-key
```c
cd /home/sam
ssh-keygen -t rsa -P '' -f ./id_rsa_myself
```
这个命令会在当前目录生成  id_rsa_myself 和id_rsa_myself.pub 的一对

## 设置ssh 环境
**拷贝文件**
```c
mkdir -p /home/sam/.ssh
cp /home/sam/id_rsa_myself /home/sam/.ssh
cp /home/sam/id_rsa_myself.pub /home/sam/.ssh
```

**设置git_ssh脚本**
```c
cat /home/sam/ssh-git.sh
 
#!/bin/sh
# private ssh key
PKEY=/home/sam/.ssh/id_rsa_myself
 
if [ ! -f "$PKEY" ]; then
    # if PKEY is not specified, run ssh using default keyfile
    ssh "$@"
else
    ssh  -i "$PKEY" "$@"
fi
 
 
设置权限：
chmod +x /home/sam/ssh-git.sh
 
设置环境变量：
echo "export GIT_SSH=/home/sam/ssh-git.sh" >> /home/sam/.bashrc
```

## 贝公钥到github
cat /home/sam/.ssh/id_rsa_myself.pub

将相关内容拷贝到 自己的gibhub 账号的 的 ssh keys中

## 下载测试
在 本地的 sam 用户下，git clone 自己账号下的 project

# 使用场景
```c
（1) mac本地
    本地 source insight 中开发，新建分支，上传到 gitlab 上（可以不管是否能够编译通过，以及是否存在bug）；
    （涉及git的使用；此中不进行讨论）；
 
（2）编译设备
    2.1）编译设备上 从 gitlab 上下载源码，然后编译（涉及到创建自己用户，以及配置 ssh 从 gitlab下载；见下面配置);
    2.2）编译通过后，将bin文件拷贝到测试设备上运行；
 
（3）测试设备
    测试设备上gdb调试程序；
        为了调试时查看源码；编译设备上 从 gitlab 上下载源码，然后编译（涉及到创建自己用户，以及配置 ssh 从 gitlab下载）；
 
（4）其他
（4.1）linux上创建用户以及配置
      1) 创建：
      mv /home/sam /home/sam-bak
      userdel sam
      useradd –d  /home/sam -m sam
 
      2）查看：
      cat /etc/passwd
      ls -al /home/sam
 
      3）切换用户 su:
      切换: [sudo] su - sam
 
（4.2）linux机器上设置ssh从gitlab下载代码
```

# 参考
```c
# linux 设备上设置ssh从gitlab下载代码-转
https://blog.csdn.net/legend050709/article/details/112794949
```
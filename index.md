## 通过ssh实现与服务器的sftp文件传输

sftp与ftp在语法、功能等特性上差不太多，相较于ftp有更好的传输安全性（加/解密），但传输效率相对较低。这里主要提供sftp的实现方式。之前在服务器上测试过传800MB左右的h5文件大约可以跑到平均300MB/s的速率，需要用服务器上的预训练模型在本地测试验证、或者本地的数据集上传服务器进行训练的话比插拔U盘、硬盘啥的还是要方便不少的（doge

### 配置前提/目标

这里只考虑打通sftp/ssh文件信道，不考虑除管理员外的用户用ssh从本地直接访问服务器的需求。有ssh直接访问需求的话可以参考VNC-Server的搭建，这篇博客对此就暂且按下不表～

### Steps
#### 0. 安装、配置openssh
首先检查是否安装过openssh-sever和openssh-client。因为两者版本需尽可能对应，若只安装了两者中的1个，可以先删除，再两个一起安装
```cmd
dpkg --get-selections | grep ssh
```
若只返回libssh-4:amd64、ssh-import-id两个包，则未安装。而安装openssh时一般只需要安装openssh-server，openssh-client会被默认安装
```cmd
sudo apt-get install openssh-client
ssh -V # 验证是否安装完成
```
#### 1. 创建sftp用户、用户组
```cmd
sudo adduser transimission
sudo addgroup sftp-users
# 将transimission从所有其他用户组中移除并加入到sftp-users组，并且关闭其Shell访问
# /bin/false也可以替换为/sbin/nologin，目的都是不允许该用户登录到系统中
sudo usermod -G sftp-users -s /bin/false transmission
sudo adduser <admin>
sudo addgroup ssh-users
# -a 表示以追加形式将 <admin> 加入 ssh-users 
sudo usermod -a -G ssh-users <admin>
```
#### 2. 创建文件服务器目录
```cmd
# 创建监狱目录
sudo mkdir /home/sftp_root
# 普通用户能够写入的共享文件目录
sudo mkdir /home/sftp_root/shared
# 设置共享文件夹的拥有者为管理员、用户组为 sftp-users
sudo chown <admin>:sftp-users /home/sftp_root/shared
# 拥有者、sftp用户组的成员具有一切权限
sudo chmod 770 /home/sftp_root/shared
```
这里默认允许所有用户或用户组登录

#### 3. 用户权限配置
```cmd
sudo vim /etc/ssh/sshd_config
```
在sshd_config文件最后添加如下内容
```cmd
AllowGroups ssh-users sftp-users
Match Group sftp-users
ChrootDirectory /home/sftp_root
AllowTcpForwarding no
X11Forwarding no
ForceCommand internal-sftp
```
即可

#### 4. 重启以生效
```cmd
service ssh restart
```

### 测试
本人目前只有macOS平台供测试：打开Terminal，Shell选择新建远程连接，将服务器的IP添加进去，连接（需要输入密码，通常为配置ssh时设置的密码）。部分操作命令参考如下：
```cmd
cd shared # 进入普通用户能够写入的shared文件夹
get <filedir_server> <filedir_client> # 将服务器上的文件下载到本地指定路径
put <filedir_client> <filedir_server> # 将本地文件上传到服务器指定路径
```

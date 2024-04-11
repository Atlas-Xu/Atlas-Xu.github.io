---
title: RHCSA题解
tags: Linux
description: 原本地Obsidian记录的RHCSA做题的笔记
password: RHCSA exam
message: 仅供个人考试使用
abbrlink: 52c231e4
date: 2024-04-10 00:28:26
---
# 在 servera.lab.example.com 上执行以下任务
## 1 配置网络设置
将 servera 配置为具有以下网络配置：
- [ ] 主机名：servera.lab.example.com
- [ ] IP 地址：172.25.250.10
- [ ] 子网掩码：255.255.255.0
- [ ] 网关：172.25.250.254
- [ ] DNS：172.25.250.254
### answer
由于servera一开始无网络，因此ping不通，需要在控制台（VM control）→ 选择虚拟机 → console → 进入控制台
未启动servera的情况下，无法console进入（个人尝试）
#### nmcli做法
```shell
nmcli con show
nmcli mod 'Wired connection 1' conn.autoconnection yes \
ipv4.method manual ipv4.address 172.25.250.10/24 \
ipv4.gateway 172.25.250.254 ipv4.dns 172.25.250.254
nmcli con up 'Wired connection 1'
hostnamectl set-hostname servera.lab.example.com
```

#### nmtui做法（首选）
1. edit connection → ipv4 configuration - manual → 填对应信息+Automaticlly connection → OK
2. 手动启动网卡：active connection
3. 设置主机名：单独的第三个选项
4. 以上做完可以在主机shell进行ping验证

#### 登录shell
RHEL 9.0默认不允许root通过ssh登录，因此2种做法
##### 创建普通用户
```shell
adduser demo
passwd demo
```
后通过`su-` 切换root用户
##### 修改配置文件允许root登录
```shell
vi /etc/ssh/sshd_config 
	PermitRootLogin yes
systemctl restart sshd
```
做完关闭VM control，从shell做题
## 2 配置您的系统以使用默认存储库
yum 存储库已可以从：
- [ ] http://content/rhel9.0/x86_64/dvd/BaseOS 和
- [ ] http://content/rhel9.0/x86_64/dvd/AppStream 使用
配置您的系统以使用这些位置用作默认存储库
### answer
1. 创建存储库配置文件，注意后缀必须为` .repo`并写入相关配置
```shell
vi /etc/yum.repos.d/local.repo
	[BaseOS] 
	baseurl=http://content/rhel9.0/x86_64/dvd/BaseOS 
	enabled=1 
	gpgcheck=0 
	[AppStream]
	baseurl=http://content/rhel9.0/x86_64/dvd/AppStream
	enabled=1 
	gpgcheck=0
```
2. 验证存储库元数据并安装考试中可能需要的实用工具
```shell
### Verify ###
dnf repolist -v # 获取仓库元数据
dnf install -y vim bash-completion # 尝试安装软件

source /etc/profile.d/bash-completion.sh # 生效 Tab 键补齐
```
**注** ：如果常用软件在系统中已经预装，则无需安装，只需要看到仓库元数据即可

## 3 调试SELinux
非标准端口上运行的 Web 服务器在提供内容时遇到问题。根据需要调试并解决问题，使其满足以下条件：
- [ ] 系统上的 Web 服务器能够提供 `/var/www/html` 中所有现有的 HTML 文件（注：不要删除或以其他方式改动现有的文件内容）
- [ ] Web 服务器在非标准端口上提供此内容
- [ ] Web 服务器在系统启动时自动启动
### answer
#### 查看httpd服务的状态，通过日志以得知运行的端口号

1. 查看状态`systemctl status httpd`
2. 重新执行查看报错`systemctl restart httpd`
#### 输出以下报错信息
	Job for httpd.service failed because the control process exited with error code.
	See "systemctl status httpd.service" and "journalctl -xeu httpd.service" for details.
3. 执行报错提示`journalctl -xeu httpd.service`
4. 向上滑动查看报错信息：`could not bind to address`
5. 验证报错内容，可以看到监听82端口`grep -i listen /etc/httpd/conf/httpd.conf`
####  配置SELinux端口类型标签并重启httpd服务（==开机自启动==）
1. `semanage`命令如果不存在情况下，查找安装软件包`dnf provides semanage`
2. 查看man手册，搜索example（`man semanage port`）；或者`man semanage port | grep \# `，第二条即为答案，修改端口
3. `semanage port -a -t http_port_t -p tcp 82`
4. 重启服务并查看状态`systemctl restart httpd`
5. 设置自启动`systemctl enable httpd`
#### 更新/var/www/html目录下的文件标签
1. `ls -l /var/www/html`检查三个文件所属的用户和用户组，如均为apache则无问题，如为root需要都改为apache
2. `chown apache:apache *`
3. 分别验证file1、2、3：`curl http://localhost:82/file*`，会发现仅file2权限正确
4. `ll -Z`查看标签，发现仅有file2标签正确
5. `restorecon -RFv /var/www/html`刷新权限
6. `ll -Z`查看标签，发现还有一个标签不正确
7. 手动刷新，标签为对的标签 `chcon -t httpd_sys_content_t /var/www/html/file3`
#### 开启防火墙82端口
```shell
firewall-cmd --add-port=82/tcp --permanent
firewall-cmd --reload
```
#### 验证
```shell
### Verify ###
systemctl status httpd #运行和是否自启动
ls -Z /var/www/html #所有标签是否为httpd_sys_content_t

###在考试机上而非servera执行（开启防火墙82端口才可以成功）###
curl http://servera:82/file1
curl http://servera:82/file2
curl http://servera:82/file3
```


## 4 创建用户账户
创建下列用户、组和组成员资格：
- [ ] 名为`sysmgrs`的组
- [ ] 用户`natasha`，作为次要组从属于`sysmgrs`
- [ ] 用户`harry`，作为次要组还从属于`sysmgrs`
- [ ] 用户 `sarah`，无权访问系统上的交互式 shell 且不是 `sysmgrs` 的成员
- [ ] `natasha`、`harry` 和 `sarah` 的密码应当都是 `flectrag`
### answer
1. 本题无陷阱，按要求做题即可
```shell
groupadd sysmgrs
useradd natasha -G sysmgrs # 次要组需要 -G
useradd harry -G sysmgrs
adduser sarah -s /sbin/nologin
# 交互式配密码
passwd natasha

# echo方式（推荐）
echo 'flectrag' | passwd --stdin natasha
echo 'flectrag' | passwd --stdin harry
echo 'flectrag' | passwd --stdin sarah
```
2. 验证
```shell
### Verify ###
id natasha
ssh natasha@localhost #需能正常登录
ssh sarah@localhost #需拒绝登录
```

## 5 配置计划任务
- [ ] 配置 `cron` 作业，以用户 `harry` 身份在每天 14:32 分执行` logger "EX200 in progress"`
- [ ] 配置 `cron` 作业，以用户 `natasha` 身份每隔 2 分钟执行 `echo "EX200 in progress"`
### answer
1. 计划任务是守护进程，检查守护进程是否在运行`systemctl status crond`
2. 启动crond：`systemctl enable --now crond`
3. 参考`cat /etc/crontab`填写时间
4. 题目二选一考
```shell
# 第一题
crontab -e -u harry
32 14 * * * logger "EX200 in progress"
# 第二题
crontab -e -u natasha
*/2 * * * * `echo "EX200 in progress"
```
5. 验证：`crontab -l -u harry`，log会打到日志，可以通过`journalctl -r`验证


## 6 创建协作目录
创建具有以下特征的协作目录 `/home/managers`
- [ ] `/home/managers` 的组所有权是 `sysmgrs`
- [ ] 目录应当可以被 `sysmgrs` 组的成员读取、写入和访问，但任何其他用户不具备这些权限。（注：root 用户有权访问系统上的所有文件和目录）
- [ ] `/home/managers` 中创建的文件自动将组所有权设置到 `sysmgrs` 组
### answer
本题无陷阱
1. 创建目录：`mkdir -p /home/managers`
2. 配置组所有权：`chown root:sysmgrs /home/managers/`
3. 配置权限，组可读写访问→7，others无权限→0，root有权（所有者）→7，因此目录权限为770；自动设置到sysmgrs→特殊权限位sgid：2（suid=4 sgid=2 stick bit=1），综上，权限为2770→`chmod 2770 /home/managers`
4. 检查：`ll -d /home/managers`，或者`stat /home/managers
5. 验证：`touch file`，查看创建的文件组为是否默认为`sysmgrs`
## 7 配置NTP
- [ ] 配置您的系统，使其成为`materials.example.com`的 NTP 客户端。
*注*：`materials.example.com`是`classroom.example.com`的 DNS 别名
### answer
1. 检查是否安装`chrony`，如果忘了软件包名字则`dnf search ntp`：`rpm -qa | grep chrony`
2. 查看配置文件：`vim /etc/chrony.conf`，如果已安装会发现进去后配置了server条目，该条目需删掉，自己配置的菜生效
3. `server materials.example.com iburst`
4. 重启`systemctl enable --now chronyd`
5. 检查同步：`chronyc sources`，看到 `*` 即为同步成功,`timedatectl`
## 8 配置autofs
配置 autofs，按照以下所述自动挂载远程用户的主目录：
- [ ] `classroom.example.com`(172.25.254.254) NFS 导出 `/rhome` 到您的系统。此文件系统包含为用户 `remoteuser1` 预配置的主目录
- [ ] remoteuser1 的主目录是`classroom.example.com:/rhome/remoteuser1`
- [ ] `remoteuser1` 的主目录应自动挂载到本地 `/rhome` 下的 `/rhome/remoteuser1`
- [ ] 主目录必须可供其用户写入
- [ ] remoteuser1 的密码是 flectrag
### answer
1. 安装autofs软件包：`dnf install autofs`
2. 修改配置文件`vim /etc/auto.master.d/aaa.autofs`，配置内容：`/rhome /etc/auto.aaa`
3.  创建挂载文件：`vim /etc/auto.aaa`，文件内容：`* -rw,sync classroom.example.com:/rhome/&`
4. 设置自启动服务：`systemctl enable --now autofs`
5. 验证:
```shell
su - remoteuser1
df -hT # 查看文件系统是否有 NFS
touch file # 创建一个文件检查是否可写
rm -rf file # 删除刚刚创建的文件
exit # 务必退出回到 root 用户继续做题
```

## 9 配置用户账户
- [ ] 配置用户 manalo，其用户 ID 为 3533。
- [ ] 此用户的密码应当为 flectrag
### answer
1. 本题无陷阱，正常做题
```shell
adduser manalo -u 3533
echo 'flectrag' | passwd --stdin manalo
```
2. 验证：`id manalo`
## 10 查找文件
- [ ] 查找用户 `greedy` 的所有文件并将其副本放入 `/root/findfiles` 目录
### answer
1. 创建目录：`mkdir /root/findfiles`
2. 查找greedy：`find / -xdev -user greedy`
3. 以归档模式复制：
```shell
cp -a /etc/samba/easy /root/findfiles
cp -a /usr/share/redhat-release/me.txt /root/findfiles
cp -a /var/games/readme /root/findfiles
```
4. （建议）管道符做法：`find / -xdev -user greedy | xargs -I{} cp -a {} /root/findfiles` `
**注**：必须有`-a`，否则复制的文件归属是root
## 11 查找字符串
- [ ] 查找文件 `/usr/share/doc/sudo/README` 中包含字符串 `ng` 的所有行。将所有这些行的副本按原始顺序放在文件`/root/list` 中。`/root/list` 不得包含空行，且所有行必须是 `/usr/share/doc/sudo/README` 中原始行的确切副本。
### answer
`grep ng /usr/share/doc/sudo/README >/root/list`

## 12 创建存档
- [ ] 创建一个名为 `/root/backup.tar.bz2` 的 tar 存档，其应包含 `/usr/local`的内容。该 tar 存档必须使用 bzip2 进行压缩。
- [ ] 创建一个名为` /root/etc.tar.gz` 的 tar 存档，其应包含 `/etc` 的内容。该 tar 存档必须使用 gzip 进行压缩。
### answer
以上2抽1
1. 安装tar软件包：`dnf install tar bzip2 -y`
2. 创建tar包：
```shell
tar -cjvf /root/backup.tar.bz2 /usr/local
tar -czvf /root/etc.tar.gz /etc
```
3. 验证
```shell
file /root/backup.tar.bz2
file /root/etc.tar.gz
tar -tvf /root/backup.tar.bz2
```

## 13 创建一个容器映像
- [ ] 使用 contsvc 用户，下载 http://content/Containerfile
- [ ] 不要修改这个文件内容，构建映像名为 pdf
### answer
1. 正常已配置免密登录`ssh contsvc@localhost`，若需要输入密码则为默认密码`flectrag`
2. 配置滞留服务：`loginctl enable-linger`
3. 下载container file：`wget http://content/Containerfile`
4. 确认容器环境
```shell
cd /home/contsvc/.config/containers

# 该路径下的registries.conf需要有以下内容
unqualified-search-registries = ['registry.lab.example.com']
 
[[registry]]
location = "registry.lab.example.com"
insecure = true
blocked = false
```
5. 若无容器环境，安装podman后，`man -k registry`→`man containers-registries.conf`→搜example
6. 登录仓库：`podman login registry.lab.example.com`→用户名密码看考试文档开头
7. 构建镜像：`podman build -t pdf .`
8. 验证：`podman images`

## 14 将容器配置为服务
- [ ] 以 `contsvc` 用户身份配置一个 `systemd` 服务
- [ ] 容器名称为 `ascii2pdf`
- [ ] 使用刚创建的映像 pdf
- [ ] 该服务命名为 `container-ascii2pdf`，并在系统重启时自动启动，无需干预
- [ ] 将服务配置为在启动时将` /opt/file` 挂载到容器中的 `/opt/incoming` 下，`/opt/progress` 挂载到容器中的`/opt/outcoming` 下
### answer
1. 创建宿主机目录，并配置目录权限
```shell
# root用户下
mkdir /opt/file
mkdir /opt/progress
chown contsvc:contsvc /opt/*
```
2. 启动容器（主要:Z，配置SELinux，contsvc用户下）
`docker run -d --name ascii2pdf -v /opt/file:/opt/incoming:Z -v /opt/progress:/opt/outcoming:Z pdf:latest`
3. 创建系统服务
```shell
mkdir -p ~/.config/systemd/user
cd $_
podman generate systemd -n ascii2pdf -f
systemctl --user daemon-reload
```
4. 验证
```shell
podman stop ascii2pdf
systemctl --user enable container-ascii2pdf.service --now
podman ps
```
## 15 配置 sudo 权限
- [ ] 将 admin 组所有成员配置为拥有 sudo 命令的执行权限，且执行 sudo 命令时无需输入密码验证
### answer
1. 本题无陷阱，按需求做题：
```shell
visudo
	# /wheel 搜索wheel
	%admin ALL=(ALL) NOPASSWORD:ALL ## yy p 对着抄把wheel改成admin
```
2. 验证（可不验证）
```shell
adduser test -G admin
su - test
sudo head -1 /etc/shadow
exit
userdel -r test
```

## 16 配置默认文件权限
- [ ] 设置 `natasha` 用户创建的目录权限默认为 733，文件默认为 622，要求永久生效
- [ ] 设置 `harry` 用户创建的目录和文件，组用户和任何人不具备任何权限，要求永久生效
### answer
*注*：目录默认权限 777 文件 666，因此题目所要求权限（umask）=777-???
```shell
vim ~natasha/.bashrc #最后一行加入如下内容
	umaks 044 #注: 777-044=733,666-044=622
vim ~harry/.bashrc 
	umask 077 #注: 777-077=700,666-077=600
```
验证（可不验证）
```shell
su - natasha
touch file
mkdir demo
ls -l
rm -rf file demo
```
## 17 配置新用户的密码策略
在系统创建新用户时，符合以下规则：
- [ ] 默认密码策略为：20天后，密码会过期
- [ ] 密码复杂度要求为：最低9位
### answer
内容在该文件的46%的位置（无需验证）
```shell
vim /etc/login.defs 
	PASS_MAX_DAYS 20 
	PASS_MIN_LEN 9
```
## 18 配置登录消息
- [ ] 配置 `natasha` 用户使其在登录系统时在终端上打印 `"Welcome to ex200 exam"`
- [ ] 配置 RHCSA 应用程序/环境变量使 `harry` 用户在登录系统时在终端上打印 `"Welcome to ex200 exam"`
### answer
该题为附加题，不一定考，在原来加umask的底下加
```shell
vim ~natasha/.bashrc #最后一行加入如下内容
	echo "Welcome to ex200 exam"
vim ~harry/.bashrc 
	RHCSA="Welcome to ex200 exam"
	export RHCSA
	echo $RHCSA
```

## 19 创建脚本文件
创建一个名为 `mysearch` 的脚本，要求如下：
- [ ] 脚本放置在 `/usr/local/bin`
- [ ] 查找 `/usr` 下所有大于 10K 并且小于 40K 且具有设置组ID权限的文件
- [ ] 将查找到的文件列表保存到 `/root/files` 中

创建一个名为 myfile 的脚本，要求如下：
- [ ] 脚本放置在` /usr/local/sbin`
- [ ] 查找` /usr/share` 下小于 1M 的文件
- [ ] 将查找到的文件保存到 `/root/dfiles` 中

### answer
以上也是二选一考
1. 切到对应路径，创建脚本并配置权限：
2. mysearch脚本
```bash
#!/bin/bash
find /usr -type f -size +10k -size -40k -perm -2000 >/root/files
```
3. myfile脚本
```bash
#!/bin/bash
find /usr/share -type f -size -1M | xargs -I{} cp -a {} /root/dfiles
```
4. 配置可执行权限：`chmod +x mysearch`;`chmod +x myfile`
5. 验证查看文件


## 重启 servera 虚拟机检查
重启 `reboot`
- 检查网络连通性，是否能够通过 ssh 连接到机器（IP地址检查）
- 容器是否能够自启动
- 其他题目只需要在做题时检查即可

# 在serverb.lab.example.com上执行以下任务

## 20 设置（破解）root密码
- [ ] 将 serverb 的 root 密码设置为 `flectrag`
您需要获得系统访问权限才能进行此操作。
### answer

#### 进入console做题
1. 打开控制台，发送`CTRL-ALT-DEL`，重启serverb
2. 进入到 bootloader 界面后，注意倒计时，选择带有 `rescue` 关键字的内核条目，即需要进入救援系统，正常原系统也会上锁无法进入
3. 在rescue上按`e`进入编辑
4. 光标移动到以`linux`开头的行，按`CTRL-E`将光标切换到行尾并输入`rd.break`
5. 按`CTRL-X`开始引导过程
6. 等待片刻后，会看到 "(or press Control-D to continue):"
7. 直接按下`Enter`无需输入密码，此时会看到命令提示符
8. 到救援系统根目录下查看，`sysroot`才是原系统真正的根目录
9. 执行`mount | grep root`命令查看根目录当前所在挂载点，可以看到位于`/sysroot `但以只读方式挂载
10. 执行`mount -o remount,rw /sysroot`重新以读写方式挂载该目录
11. 执行`chroot /sysroot`切换到根目录环境，注意提示符不会发生变化，但实际上已经切换，可以用 ls 命令进行验证
12. 执行`passwd`命令修改密码（如果给的密码太复杂，就先设置简单的登录后再改）
13. 执行`touch /.autorelabel`更新 SELinux
14. 输入两次`exit`继续引导过程
15. 等待后，重新以新密码 `flectrag` 登录系统
16. 同servera一样需要修改配置文件使root登录或者创建普通用户
## 21 配置您的系统以使用默认存储库
yum 存储库已可以从：
- [ ] http://content/rhel9.0/x86_64/dvd/BaseOS 和
- [ ] http://content/rhel9.0/x86_64/dvd/AppStream 使用
配置您的系统以使用这些位置用作默认存储库

### answer
本题与servera的答案一样，可以直接scp servera或者开个新终端复制
1. 创建存储库配置文件，注意后缀必须为` .repo`并写入相关配置
```shell
vi /etc/yum.repos.d/local.repo
	[BaseOS] 
	baseurl=http://content/rhel9.0/x86_64/dvd/BaseOS 
	enabled=1 
	gpgcheck=0 
	[AppStream]
	baseurl=http://content/rhel9.0/x86_64/dvd/AppStream
	enabled=1 
	gpgcheck=0
```
2. 验证存储库元数据并安装考试中可能需要的实用工具
```shell
### Verify ###
dnf repolist -v # 获取仓库元数据
dnf install -y vim bash-completion # 尝试安装软件

source /etc/profile.d/bash-completion.sh # 生效 Tab 键补齐
```
**注** ：如果常用软件在系统中已经预装，则无需安装，只需要看到仓库元数据即可

## 22 调整逻辑卷大小
- [ ] 将逻辑卷 `goodluckwithyourhcsaexam` 及其文件系统的大小调整为 230 MiB。确保文件系统内容保持不变。
*注*：分区大小很少与请求的大小完全相同，因此可以接收范围为 217 MiB 到 243 MiB 的大小。
### answer
1. `df -hT` 查看该分区大小
2. 验证该卷组内是否还有可用空间：`vgs`
3. 若有预留空间，则进行扩容，扩容命令卷组写在前，逻辑卷在后
```shell
# 卷组/逻辑卷
lvextend -rL 230M thisexamisreallyeasy/goodluckwithyourhcsaexam

# 验证
df -hT
```

## 23 添加交换分区
- [ ] 向您的系统添加一个额外的交换分区，大小 512 MiB
- [ ] 交换分区应在系统启动时自动挂载。不要删除或以任何方式改动系统上的任何现有交换分区
### answer
1. 检查磁盘是否还有剩余空间，并划分一个 512MiB 的分区
```shell
parted /dev/vdb unit MB print free
# GPT引导
fdisk /dev/vdb
# uefi引导，进入交互扩容命令
gdisk/dev/vdb
n

+512M

w
y

#
partprobe
lablk
```
2. 格式化文件系统并启用交换分区并写入 fstab
```shell
mkswap /dev/vdb2 ## 记录下 UUID
vim /etc/fstab
	UUID=xxxxxxxxxxx none swap defaults 0 0
swapon -a
```
3. 验证
```shell
free -h
swapon
```

## 24 创建逻辑卷
根据以下要求，创建新的逻辑卷：
- [ ] 逻辑卷名为 `uaRVlZIvkI`，属于 `VreOMHFjzJ` 卷组，大小为 60 个扩展块
- [ ] `VreOMHFjzJ` 卷组中逻辑卷的扩展快大小应为 16 MiB
- [ ] 使用 vfat 文件系统格式化新逻辑卷。该逻辑卷应在系统启动时自动挂载到 `/mnt/lv2` 下
### answer
```shell
gdisk /dev/vdb
n


+1G

w
y
#
partprobe
lablk

#加进物理卷
pvcreate /dev/vdb3
vgcreate -s 16m VreOMHFjzJ /dev/vdb3
#创建逻辑卷
lvcreate -n uaRVlZIvkI -l 60 VreOMHFjzJ

# 创建挂载点
mkdir /mnt/lv2
mkfs.vfar /dev/mapper/VreOMHFjzJ-uaRVlZIvkI
# 永久挂载
vim /etc/fstab
	/dev/mapper/VreOMHFjzJ-uaRVlZIvkI /mnt/lv2 vfat defaults 0 0
mount -a
```

## 25 配置系统调优
- [ ] 为您的系统选择建议的优化配置集并将它设为默认设置
### answer
1. 启动 `tuned` 服务并确保自启动：`systemctl enable --now tuned`
2. 使用`tuned-adm`命令配置最优配置集
```shell
tuned-adm recommend
tuned-adm profile virtual-guest
```
3. 验证
```shell
systemctl status tuned
tuned-adm active
```

## 重启 serverb 虚拟机检查
- 必须确保机器能够正常启动到命令行
- 登录系统检查与磁盘相关的所有题目


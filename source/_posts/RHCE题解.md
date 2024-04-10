---
title: RHCE题解
tags: Linux
description: RHCE做题的笔记
abbrlink: c030165f
date: 2024-04-10 00:09:26
password: RHCE exam
message: 仅供个人考试使用
---

# 前置
- 全部使用*ansible core*的方式做题，root登录bastion后，`su - devops`（devops用户不可直接连接的情况）,root下检查ansible core是否安装
- 每一题做完必须检查，练习时每一题都必须对
- 内容尽可能复制粘贴，手敲肯定会错
- playbook的开头三个横必须写：`---`
- 前3题是基础，没做出来后面都无法进行，整体还是需要按顺序做，前后均有一定关联性
- yaml文件注意空格
## 分屏技巧
1. 现场如果显示器能够排列下足够窗口则单独打开terminal
2. 现场如果显示器分辨率太低可以使用`vim -O file1 file2 `
## VIM
*这部分务必背下来*
编辑vim配置文件`vim ~/.vimrc`，配置tab缩进等
```shell
set nu
autocmd FileType yaml setlocal ai ts=2 sts=2 sw=2 et
set cuc #列光标定位
```
# 1 安装和配置 Ansible
按照下方所述，在控制节点上安装并配置 ansible：
- [ ] 安装所需的软件包
- [ ] 创建名为 `/home/devops/ansible/inventory` 的静态清单文件，以满足以下要求：
	- [ ] servera 是 dev 主机组的成员
	- [ ] serverb 是 test 主机组的成员
	- [ ] serverc 和 serverd 是 prod 主机组的成员
	- [ ] workstation 是 balancers 主机组的成员
	- [ ] prod 组是 webservers 主机组的成员
- [ ] 创建名为` /home/devops/ansible/ansible.cfg` 的配置文件，以满足以下要求：
	- [ ] 主机清单文件为`/home/devops/ansible/inventory`
	- [ ] 默认角色目录为 `/home/devops/ansible/roles`
	- [ ] 自定义的 collection 目录在 `/home/devops/ansible/mycollections`

## answer
1. 安装`dnf install ansible-core`
2. 切换为*devops*用户：`mkdir ansible && cd ansible`
3. 静态清单文件
```shell
vim invetory
	[dev]
	servera
	
	[test]
	serverb
	
	[prod]
	serverc
	serverd
	
	[balancers]
	workstation
	
	[webservers:children]
	prod
```
4. 生成配置文件
	1. 查看自带的配置文件项：`cat /etc/ansible/ansible.cfg`
	2. 可以看到中间的命令`ansible-config init --disabled -t all > ansible.cfg`，复制到ansible路径下，即可生成
	3. 可在vim视图中创建2个窗口`vim -O ansible.cfg demo/ansible.cfg`
	4. 排除`#`：`cat ansible.cfg | grep -vE "\#" >ansible.full`
	5. 根据生成的配置文件排除后进行删减修改为如下（在ansible.full的基础上修改）：
```shell
vim ansilbe.full
	[defaults]
	inventory=/home/devops/ansible/inventory
	remote_user=devops
	roles_path=/home/devops/ansible/roles
	collections_path=/home/devops/ansible/mycollections
	host_key_checking=False
	
	[privilege_escalation]
	become=True
	become_ask_pass=False
	become_method=sudo
	become_user=root

rm -rf ansible.cfg
mv ansible.full ansible.cfg
```
5. 创建清单目录
```shell
mkdir roles
mkdir mycollections
```
6. 验证
	1. 当前目录下运行`ansible-inventory --g`
	2. 验证主机数量：`ansible all --list`
	3. 验证连通性：`ansible all -m ping`，全部success才可
	4. 查看实际执行是否提权为root：`ansible all -a id`

## 如果需要配置密码的情况


# 2 配置您的系统以使用默认存储库
作为系统管理员，您需要在受管节点上安装软件：
- [ ] 创建一个名为` /home/devops/ansible/yum_repo.yml` 的剧本，在各个受管节点上安装以下存储库：
存储库1：
- [ ] 存储库的名称为 EX294_BASE
- [ ] 描述为 EX294 base software
- [ ] 基础 URL 为 http://content/rhel9.0/x86_64/dvd/BaseOS
- [ ] GPG 签名检查为启用状态
- [ ] GPG 密钥 URL 为 http://content/rhel9.0/x86_64/dvd/RPM-GPG-KEY-redhat-release
- [ ] 存储库为启用状态
存储库2：
- [ ] 存储库的名称为 EX294_STREAM
- [ ] 描述为 EX294 stream software
- [ ] 基础 URL 为 http://content/rhel9.0/x86_64/dvd/AppStream
- [ ] GPG 签名检查为启用状态
- [ ] GPG 密钥 URL 为 http://content/rhel9.0/x86_64/dvd/RPM-GPG-KEY-redhat-release
- [ ] 存储库为启用状态

## answer
1. 查找ansible doc中的项目：`ansible-doc -l | grep yum`
2. 查看yum对应的文档：`ansible-doc yum_repository`
3. 根据文档中EXAMPLE的例子补足即可；其中配置完第一个后，由于第二个基本上相同，`shift+v`全选后`y`粘贴修改即可
```yml
---
- name: configure dnf repository
  hosts: all
  tasks:
    - name: BaseOS
      ansible.builtin.yum_repository:
        name: EX294_BASE
        description: EX294 base software
        baseurl: http://content/rhel9.0/x86_64/dvd/BaseOS
        gpgcheck: yes
        gpgkey: http://content/rhel9.0/x86_64/dvd/RPM-GPG-KEY-redhat-release
        enabled: yes
    - name: AppStream
      ansible.builtin.yum_repository:
        name: EX294_STREAM
        description: EX294 stream software
        baseurl: http://content/rhel9.0/x86_64/dvd/AppStream
        gpgcheck: yes
        gpgkey: http://content/rhel9.0/x86_64/dvd/RPM-GPG-KEY-redhat-release
        enabled: yes
```
4. 执行：`ansible-playbook yum_repo.yml`
5. 验证：`ansible all -a "dnf repolist -v"`
# 3 安装 Collections
- [ ] 以 devops 用户身份，从以下路径中安装 collections
	- [ ] http://content/ansible-posix-1.5.4.tar.gz
	- [ ] http://content/community-general-8.2.0.tar.gz
	- [ ] http://content/redhat-rhel_system_roles-1.16.2.tar.gz
- [ ] 集合应安装到默认集合目录 `/home/devops/ansible/mycollections`
## answer
1. 安装
```shell
ansible-galaxy collection install http://materials/ansible-posix-1.5.4.tar.gz -p mycollections/

ansible-galaxy collection install http://content/community-general-8.2.0.tar.gz -p mycollections/

ansible-galaxy collection install http://content/redhat-rhel_system_roles-1.16.2.tar.gz -p mycollections/
```
2. 验证：`ansible-galaxy collection list`
## tips
本题一定要验证，和下面题目有极大关联，如果这里面的collection调用不了或者找不到安装，大概率是第一天的cfg中collections目录配置错误
# 4 安装软件包
- [ ] 创建一个名为 `/home/devops/ansible/packages.yml` 的 playbook 要求如下：
	- [ ] 将 php 和 mariadb 软件包安装到 dev、test 和 prod 组中的主机上
	- [ ] 将 RPM Development Tools 包组安装到 dev 组中的主机上
	- [ ] 将 dev 组中主机上的所有软件包更新为最新版本
## answer
1. 查找样例：`ansible-doc dnf`
2. 脚本如下
```yml
---
- name: install packages
  hosts: dev,test,prod
  tasks:
    - name: Install php and mariadb
      ansible.builtin.dnf:
        name:
          - php
          - mariadb
        state: present
- name: install packages group
  hosts: dev
  tasks:
    - name: Install package group
      ansible.builtin.dnf:
        name: '@RPM Development Tools'
        state: present
    - name: Upgrade all packages
      ansible.builtin.dnf:
        name: "*"
        state: latest
```
3. 运行：`ansible-playbook packages.yml`
4. 验证包安装（任意查看一个包是否安装成功即可）：`ansible dev,test,prod -a "rpm -q php"`
5. 验证软件包是否安装：（root下）`dnf search "RPM Development Tools"`→看到是`rpmdevtools.noarch`→（devops的ansible目录下）→`ansible dev -a "rpm -q repdevtools"`
6. 验证最新：`ansible dev -a "dnf update -y"`→最新的话应现实`nothing to do`
# 5 使用系统角色（A）
- [ ] 根据以下要求创建名为` /home/devops/ansible/selinux.yml` 的 playbook 要求如下：
	- [ ] 在所有受管节点上运行
	- [ ] 使用 selinux 角色
	- [ ] SELinux 策略为 targeted
	- [ ] SELinux 状态为 enforcing
## answer
1. （root下）搜索角色相关软件包`dnf search roles`→安装软件包`dnf install rhel-system-roles`
2. 切换回devops，进入到ansible的路径，查看软件包提供的系统角色`rpm -ql rhel-system-roles`→找到角色目录在`/usr/share/ansible/roles`下
3. 将角色目录复制到ansible的角色目录下（cp所需要的）`cp -r /usr/share/ansible/roles/rhel-system-roles.selinux/ roles`
4. 验证角色加入成功`ansible-galaxy list`
5. 查找seLinux提供的playbook的目录`rpm -ql rhel-system-roles | grep selinux | grep example` → 得到具体文件`/usr/share/doc/rhel-system-roles/selinux/example-selinux-playbook.yml`
6. 复制改文件修改`cp /usr/share/doc/rhel-system-roles/selinux/example-selinux-playbook.yml selinux.yml`
7. 编辑selinux.yml，其中become部分都删（cfg中已定义，selinux部分只留前两个，task中block前删除），编辑后的文件如下：
```shell
---
- hosts: all
  vars:
    selinux_policy: targeted
    selinux_state: enforcing
    selinux_reboot_required: true
    # Prepare the prerequisites required for this playbook
  tasks:
   - name: execute the role and catch errors
     block:
        - name: Include selinux role
          include_role:
            name: rhel-system-roles.selinux
     rescue:
        # Fail if failed for a different reason than selinux_reboot_required.
       - name: handle errors
         fail:
           msg: "role failed"
         when: not selinux_reboot_required
       - name: restart managed host
         reboot:
       - name: wait for managed host to come back
         wait_for_connection:
           delay: 10
           timeout: 300
       - name: reapply the role
         include_role:
           name: rhel-system-roles.selinux
```
8. 执行`ansible-playbook selinux.yml`，该执行过程较慢
9. 验证：`ansible all -a sestatus`，确认当前模式是enforcing
# 5 使用系统角色（B）
- [ ] 根据以下要求创建名为 `/home/devops/ansible/timesync.yml` 的 playbook 要求如下：
	- [ ] 在所有受管节点上运行
	- [ ] 使用 timesync 角色
	- [ ] 配置该角色，以使受管节点同步时间服务器 classroom.example.com
	- [ ] 配置该角色，以启用 iburst 参数
## answer
1. （root下）搜索角色相关软件包`dnf search roles`→安装软件包`dnf install rhel-system-roles`
2. 切换回devops，进入到ansible的路径，查看软件包提供的系统角色`rpm -ql rhel-system-roles`→找到角色目录在`/usr/share/ansible/roles`下
3. 将角色目录复制到ansible的角色目录下（cp所需要的）`cp -r /usr/share/ansible/roles/rhel-system-roles.timesync/ roles
4. 验证角色加入成功`ansible-galaxy list`
5. 查找所需的roles的playbook样例：`rpm -ql rhel-system-roles | grep timesync | grep example` → 选择`/usr/share/doc/rhel-system-roles/timesync/example-multiple-ntp-servers-playbook.yml`
6. `cp /usr/share/doc/rhel-system-roles/timesync/example-multiple-ntp-servers-playbook.yml timesync.yml`
7. 修改文件
```yaml
---
- hosts: all 
  vars:
    timesync_ntp_servers:
      - hostname: classroom.example.com
        iburst: yes
  roles:
    - rhel-system-roles.timesync
```
8. 验证`ansible all -a "chronyc sources"` → `*`同步成功

# 6 使用 Ansible Galaxy 安装角色
- [ ] 创建 `/home/devops/ansible/roles/requirements.yml` 文件，用以下载安装角色到 roles 目录中
- [ ] 角色信息如下：
	- [ ] 角色名：balancer 来自于 http://content/haproxy.tar
	- [ ] 角色名：phpinfo 来自于 http://content/phpinfo.tar
## answer
1. 编辑文件：`vim /home/devops/ansible/roles/requirements.yml`
```yaml
---
- name: balancer
  src: http://materials/haproxy.tar
  path: /home/devops/ansible/roles/

- name: phpinfo
  src: http://materials/phpinfo.tar 
  path: /home/devops/ansible/roles/

```
2. 安装：`ansible-galaxy install -r roles/requirements.yml`
3. 验证：`ansible-galaxy list`
# 7 创建和使用角色
- [ ] 创建一个 apache 角色，要求如下：
	- [ ] 安装 httpd 软件包，运行并设置为开机自启动
	- [ ] 配置防火墙允许访问 web 服务器的规则，运行并设置开机自启动
	- [ ] 创建模板文件 `index.html.j2` 该模板创建具有以下输出的文件：`/var/www/html/index.html`
	- [ ] `/var/www/html/index.html`文件输出如下：
		`Welcome to HOSTNAME on IPADDRESS`,其中HOSTNAME是受管节点的完全限定域名，IPADDRESS则是受管节点的IP地址
- [ ] 创建名为 newrole.yml 的 playbook 并使用该角色运行在 webservers 主机组上
## answer
1. 创建角色结构化文件:`ansible-galaxy init apache`
2. 编辑apache角色任务:`vim roles/apache/tasks/main.yml `，相关需要写的内容分布可以在`ansible-doc` 中的 `dnf` 、`service`、`firewalled`、`template` 查到
```yaml
---
- name: Install the latest version of Apache 
  ansible.builtin.dnf:
    name: httpd
    state: present
- name: Start service httpd and firewalld, if not started
  ansible.builtin.service:
    name: "{{ item }}"
    state: started
    enabled: yes
  loop:
    - httpd
    - firewalld
- name: permit traffic in default zone for https service
  ansible.posix.firewalld:
    service: http
    permanent: true
    state: enabled
    immediate: true
- name: Template a file to /etc/file.conf
  ansible.builtin.template:
    src: index.html.j2  
    dest: /var/www/html/index.html 
```
3. 收集事实变量`ansible dev -m setup >fact.txt` → 可以看到完全域名前的标识为`ansible_fqdn`（搜索servera）；IP地址前的标识为`ansible_default_ipv4.address`
4. 创建apache角色下的`template/index.html.j2`：`vim roles/apache/templates/index.html.j2`→根据事实变量得出：`Welcome to {{ ansible_fqdn }} on {{ ansible_default_ipv4.address }}`
5. 编写` newrole.yml `
```yaml
---
- name: use roles
  hosts: webservers
  roles: 
    - apache
```
6. 执行：`ansible-playbook newrole.yml`
7. 验证：在浏览器访问：`http://serverc`和`http://serverd`
# 8 使用 Ansible Galaxy 角色
- [ ] 创建一个名为 /home/devops/ansible/roles.yml 的剧本，并满足以下要求：
	- [ ] 在 balancers 主机组上，使用 balancer 角色，该角色为 webservers 主机组提供负载均衡服务，配置完成后，当访问 http://workstation.lab.example.com 将出现负载均衡现象
	- [ ] 剧本中另一个 Play 使用 phpinfo 角色，在 webservers 主机组上运行，执行完成后，访问主机组内的每台主机的 hello.php 应显示以下信息：`Hello PHP World from FQDN`
注：`FQDN`是主机的完全限定域名，来自于事实变量，页面同时显示每台受管主机的PHP配置创建
## answer
该题有陷阱，如果直接按照题目所述写的话会报错，需要调换顺序，webservers主机组需要先运行，这样balancers主机组能够收集到事实并记录，才可以进行调用。
负载均衡现象没有那么快出现，大概等个2-3秒才会看到，请勿紧张，只要有负载均衡现象出现即可
`vim /home/devops/ansible/roles.yml`
```yaml
---
- name: use apache
  hosts: webservers
  roles:
    - apache

- name: use roles balancers
  hosts: balancers
  roles:
    - balancer

- name: use roles phpinfo
  hosts: webservers
  roles:
    - phpinfo
```

## balance主机组陷阱
balancers 主机组的 firewalld 可能处于开启状态（陷阱），所以访问浏览器看不到任何页面，此时需要进入到 balancer 角色中手动添加 firewalld 模块来放行 http 流量。检查步骤如下：
1. 查看balancers下主机的名称：`ansible-inventory --g`
2. 检查balancers：`ansible balancers -a "systemctl status firewalld"`，没开则无问题
3. 检查webservers：`ansible webservers -a "systemctl status firewalld"`，若开启且没放行80，则需要进入`roles/balancer/task`更改剧本
4. 开启防火墙，可以在roles.yml中增加task，为balancers和webservers全部开启



# 9 创建和使用分区(A)
- [ ] 创建一个名为 `/home/devops/ansible/partition.yml `的 playbook ，它将在 `balancers` 主机组上运行：
	- [ ] 在磁盘 vdb 上创建主分区，id 为 1，大小为 1500MiB
	- [ ] 使用 ext4 文件系统格式化分区
	- [ ] 将文件系统永久挂载到`/newpart`
	- [ ] 如果无法创建请求的分区大小，应报错如下信息：Could not create partition of that size 并改为创建一个800MiB 的分区
	- [ ] 如果设备 vdb 不存在，应显示错误信息：Disk does not exist

## answer
### 收集环境
每个人环境可能不同，因此需要提前规划且知道在各个设备上可能出现的结果
1. `ansible all -a lsblk`，可以查看所有主机的磁盘情况
2. 根据显示结果可以得知，会在servera、serverb上创建800M，serverc上创建1.5G，workstation会报两个错
### 编写剧本
创建分区在parted找，找到带有`part_end`，打印报错在debug找，格式化在filesystem找，创建挂载点在file模块，挂载在mount模块
#### 思路
1. tasks中，写好block、rescue、always部分。block是尝试做的部分，rescue是在block出错后做的部分，always是必须执行的部分
2. 因此先在block中写好创建1.5G的内容，这部分在`parted`模块中找有`part_end`的部分
3. 若无法创建则会报错，因此在rescue中创建报错，`debug`模块；且1500创建不了需要创建800，因此也是写在rescue中，将block中的创建1500复制到下面改为800
4. 有盘后需要格式化为ext，因此放在always模块下，在`filesystem`模块下查找；格式化后则进行挂载，需要先创建挂载目录，在`file`模块中找，然后才能挂载，用`mount`模块
5. 最后如果不存在则报错打印信息，复用前面的debug；由于是不存在的时候报错，因此需要补充条件判断语句。
	1. 根据事实变量找到vdb → `ansible_devices.vdb` → ` when: ansible_devices.vdb is not defined`
	2. 补充其他的条件判断
#### 剧本
以下是在所有主机组上运行的剧本，如果只是balance主机组，则rescue中创建800M的部分不需要条件判断，建议都加上，平时练习以all来练习
```yaml
---
- name: create and use partition
  hosts: all
  tasks:
    - block:
        - name: create 1500M partition
          community.general.parted:
            device: /dev/vdb
            number: 1
            state: present
            part_end: 1500MiB
      rescue:
        - name: print error message
          ansible.builtin.debug:
            msg: "Could not create partition of that size"

        - name: create 800M partition 
          community.general.parted:
            device: /dev/vdb
            number: 1
            state: present
            part_end: 800MiB
          # 如果考试有盘小于800，则不用下面这条when
          when: ansible_devices.vdb is defined
          # 改为下条
          ignore_errors: yes
          
      always:
        - name: format fs
          community.general.filesystem:
            fstype: ext4
            dev: /dev/vdb1
          when: ansible_devices.vdb is defined

        - name: ensure directory exist
          ansible.builtin.file:
            path: /newpart
            state: directory
          when: ansible_devices.vdb is defined

        - name: mount fs
          ansible.posix.mount:
            path: /newpart
            fstype: ext4
            src: /dev/vdb1
            state: mounted
          when: ansible_devices.vdb is defined

    - name: print messages 
      ansible.builtin.debug:
        msg: "Disk does not exist"
      when: ansible_devices.vdb is not defined
```

### 验证
1. 验证分区：`ansible all -a lsblk`
2. 验证永久挂载：`ansible all -a "tail -1 /etc/fstab"`
# 9 创建和使用逻辑卷(B)
- [ ] 创建一个名为 `/home/devops/ansible/lvm.yml` 的 playbook，在所有节点运行要求如下：
	- [ ] 在 research 卷组中，创建 1500M 的逻辑卷 data
	- [ ] 使用 ext4 文件系统格式化逻辑卷
	- [ ] 如果无法创建请求的逻辑卷大小，则报错如下信息：Could not create logical volume of that size 并改为创建一个 800M 的逻辑卷代替
	- [ ] 如果卷组 research 不存在，则报错如下信息：`Volume group does not exist`
	- [ ] 不要以任何方式挂载逻辑卷
## answer
### 练习时
因为做了9A，因此执行：`bash init-vdb-status.sh`和`bash deploy-vg.sh`，清空上一题剧本运行后的结果，考试时，AB仅会抽一题，一般抽A

### 思路
1. 收集信息，因为是创建vg，因此使用`ansible all -a vgs`查看所有卷组信息，根据所得信息可以得出：servera、b、d会报错并创建800M的卷组，serverc会直接创建1500M，workstation会报两个错无法创建
2. 本题直接要求在所有主机组上运行，因此hosts要写all，tasks中，写好block、rescue、always部分。block是尝试做的部分，rescue是在block出错后做的部分，always是必须执行的部分
3. 首先是创建逻辑卷，模块是`lvol`，如果不记得可以`ansible-doc -l | grep -i lvm`，第一个示例就是对的，同理写在block中
4. 如果无法创建1500就报错，报错后需要再尝试创建800M的vg，这部分则需要在rescue中
5. 格式化逻辑卷，搜索`filesystem`模块，写在always中。此时的dev需要填写逻辑卷的路径→`/dev/research/data`即`/etc/vg/lv`
6. 最后卷组不存在报错，也是debug模块，需要在always中，同时要考虑条件判断，条件判断时，通过事实变量找到→`ansible_lvm.vgs.research`
7. 严谨些需要将前面的判断条件补充，如果在创建800M时也是空间不够，同样写`ignore_errors: yes`
### 剧本
```yaml
---
- name: create lv
  hosts: all
  tasks:
    - block:
        - name: Create 1500M lv
          community.general.lvol:
            vg: research
            lv: data
            size: 1500
      rescue:
        - name: print error messages
          ansible.builtin.debug:
            msg: "Could not create logical volume of that size"

        - name: Create 800M lv
          community.general.lvol:
            vg: research
            lv: data
            size: 800
          ignore_errors: yes
      always:
        - name: format ext4 fs
          community.general.filesystem:
            fstype: ext4
            dev: /dev/research/data
          when: ansible_lvm.vgs.research is defined

        - name: print messages
          ansible.builtin.debug:
            msg: "Volume group does not exist"
          when: ansible_lvm.vgs.research is not defined
```

### 验证


# 10 生成主机文件
- [ ] 将初始模板文件从 http://content/hosts.j2 下载到 `/home/devops/ansible` 目录中
- [ ] 完善该模板，用以生成受管节点的 `/etc/myhosts` 文件
- [ ] 创建名为`/home/devops/ansible/hosts.yml `的 playbook，对 dev 主机组使用此模板
- [ ] 该剧本运行后，dev 主机组中的 `/etc/myhosts` 内容最终如下：
```shell
127.0.0.1 localhost localhost.localdomain localhost4 localhost4.localdomain4
::1 localhost localhost.localdomain localhost6 localhost6.localdomain6
172.25.250.9 workstation.lab.example.com workstation
172.25.250.10 servera.lab.example.com servera
172.25.250.11 serverb.lab.example.com serverb
172.25.250.12 serverc.lab.example.com serverc
172.25.250.13 serverd.lab.example.com serverd
```

## answer
该题也有陷阱，需要所有主机先运行
1. 下载文件，wget
2. 下面内容的组成是由主机的事实变量组成，分别是`ansible_default_ipv4.address`、`ansible_fqdn` 、`ansible_hostname`
3. 涉及多个节点，因此需要编写for循环，循环中需要通过魔法变量`groups`来获取所有主机名字；下面需要调用各个主机组的事实变量则需要通过环境变量`hostvars`
### `hosts.j2`修改后
注意这个文件写的时候的空格，此处练习时常出错
```j2

{%  for i in groups.all  %}
{{  hostvars[i].ansible_default_ipv4.address  }} {{  hostvars[i].ansible_fqdn  }} {{  hostvars[i].ansible_hostname  }}
{%  endfor  %}
```
### playbook
查询template模块，且所有主机先运行才能收集到所有的事实变量
或者用when（如果要求一个剧集内完成）

#### 两个剧集的写法
```yaml
---
- name: get all facts
  hosts: all

- name: Generate /etc/myhosts
  hosts: dev
  tasks:
    - name: use template
      ansible.builtin.template:
        src: hosts.j2
        dest: /etc/myhosts
```

#### 一个剧集的写法
```yaml
---
- name: Generate /etc/myhosts
  hosts: all
  tasks:
    - name: use template
      ansible.builtin.template:
        src: hosts.j2
        dest: /etc/myhosts
      when: inventory_hostname in groups.dev
```

# 11 修改文件内容
- [ ] 创建一个名为 `/home/devops/ansible/issue.yml` 的 playbook，要求如下：
	- [ ] 剧本将在所有受管节点上运行
	- [ ] 将 `/etc/issue` 的内容替换为下方所显示的文本
		- [ ] 在 dev 主机组上，这行文本显示为：Development
		- [ ] 在 test 主机组上，这行文本显示为：Test
		- [ ] 在 prod 主机组上，这行文本显示为：Production
## answer
*tip*: 只能写一个剧集且hosts为all，否则没分
*验证*：`ansible all -a "cat /etc/issue"`
查询`copy`模块，找到`content`关键词，复制那一项，修改后如下：
```yaml
---
- name: modify file content
  hosts: all
  tasks:
    - name: dev
      copy:
        content: "Development"
        dest: /etc/issue
      when: inventory_hostname in groups.dev
    - name: test
      copy:
        content: "Test"
        dest: /etc/issue
      when: inventory_hostname in groups.test
    - name: prod
      copy:
        content: "Production"
        dest: /etc/issue
      when: inventory_hostname in groups.prod
```

# 12 创建并管理网站目录
- [ ] 创建一个名为 /home/devops/ansible/webcontent.yml 的 playbook，要求如下：
	- [ ] 剧本在 dev 主机组中运行
	- [ ] 创建 /webdev 目录，要求如下：
		- [ ] 所属组为 webdev 组
		- [ ] 权限为 owner=read+write+execute，group=read+write+execute，other=read+execute
		- [ ] 具有特殊组权限 set group ID
		- [ ] 用符号链接将 /var/www/html/webdev 链接到 /webdev
		- [ ] 创建文件 /webdev/index.html 内容为 Red Hat Ansible
		- [ ] 在 dev 主机组上浏览 http://servera.lab.example.com/webdev 将看到 Red Hat Ansible

## answer
本体陷阱：需要配置SELinux
1. 创建后使用先使用apache这一角色
2. ansible-doc file中找EXAMPLE→dire，使用该模板
```yaml
---
- name: create and manage website
  hosts: dev
  roles:
    - apache
  tasks:
    - name: Ensure group "webdev" exists
      ansible.builtin.group:
        name: webdev
        state: present

    - name: Create a directory if it does not exist
      ansible.builtin.file:
        path: /webdev 
        state: directory
        group: webdev
        mode: '2755'
    - name: Create a symbolic link
      ansible.builtin.file:
        src: /webdev 
        dest: /var/www/html/webdev 
        state: link
    - name: Copy using inline content
      ansible.builtin.copy:
        content: 'Red Hat Ansible'
        dest: /webdev/index.html
        setype: httpd_sys_content_t

```

# 13 生成硬件报告
- [ ] 创建一个名为 `/home/devops/ansible/hwreport.yml` 的 playbook，在所有受管节点上生成含有以下输出的文件`/root/hwreport.txt`
	- [ ] 清单主机名称
	- [ ] 总内存大小（单位MB）
	- [ ] BIOS 版本
	- [ ] 磁盘设备 vda 的大小
	- [ ] 磁盘设备 vdb 的大小
- [ ] 输出文件中的每一行含有一个 key=value
- [ ] 剧本应当能从 http://content/hwreport.empty 下载该文件并保存到 `/root/hwreport.txt`
- [ ] 如果硬件项不存在，相关的值设置为 NONE
## answer
下载是`get_url`模块，编辑文件内的行是`lineinfile`模块

# 14 创建密码库
- [ ] 创建一个 Ansible 密码库，用来存储用户密码
	- [ ] 库名称为 /home/devops/ansible/locker.yml 
	- [ ] 库中含有两个变量，名称如下：
		- [ ] pw_developer 值为 Imadev
		- [ ] pw_manager 值为 Imamgr
	- [ ] 用于加密和解密该库的密码为 8OUWb5ZOs93xw9nn
	- [ ] 密码存储在文件 /home/devops/ansible/secret.txt 中

## answer
送分大题
根据题目描述的locker.yml文件如下
```yaml
---
- pw_developer: Imadev
- pw_manager: Imamgr
```
编辑密码：`echo "8OUWb5ZOs93xw9nn" >/home/devops/ansible/secret.txt 
`
加密locker.yml：`ansible-vault encrypt locker.yml`
# 15 创建用户账户
- [ ] 从 http://content/user_list.yml 下载要创建的用户列表，并将它保存到 `/home/devops/ansible` 中
- [ ] 创建名为` /home/devops/ansible/users.yml `的 playbook，并按以下要求创建用户：
	- [ ] 职位描述为 developer 的用户应当：
		- [ ] 在 dev 和 test 主机组中创建
		- [ ] 从 pw_developer 变量分配密码
		- [ ] 是附属组 devops 的成员
		- [ ] 配置该用户的密码过期时间为 20 天
	- [ ] 职位描述为 manager 的用户应当：
		- [ ] 在 prod 主机组中创建
		- [ ] 从 pw_manager 变量分配密码
		- [ ] 是附属组 opsmgr 的成员
		- [ ] 配置该用户的密码过期时间为 30 天
- [ ] 用户密码采用 SHA512 哈希格式
- [ ] 剧本能在本次考试中使用在其他位置创建的库密码文件 `/home/devops/ansible/secret.txt`
## answer
1. 先下载文件，没说在playbook中下载就单独下载
2. playbook
```yaml

```
# 16 更新 Ansible 库的密钥
- [ ] 按照下方描述，更新现有库的密钥：
	- [ ] 从 http://content/salaries.yml 下载库到 /home/devops/ansible
	- [ ] 当前的库密码为: 4aBz036YEZtMvFF1
	- [ ] 新的库密码为: bxl64AcTKqXu1EQ2
## answer
送分大题
1. 下载文件`wget http://content/salaries.yml`
2. 修改密码：`ansible-vault rekey salaries.yml` → 先输入旧密码 → 输入新密码
3. 验证：`ansible-vault view salaries.yml`
# 17 配置计划任务
- [ ] 创建一个名为 `/home/devops/ansible/cron.yml` 的剧本，在所有受管节点上运行，配置一个计划任务：
	- [ ] 运行身份为：devops
	- [ ] 运行时间为：每间隔2分钟
	- [ ] 运行命令：logger "EX294 in progress"
## answer
模块为`cron`
陷阱：crontab不一定在运行，因此playbook需要先确保服务在运行且自启动
```yaml
---
- name: crontab
  hosts: all
  tasks:
	- name: 2 min
		ansible.builtin.cron:
		  name: "demo"
		  minute: "*/2"
		  user: devops
		  job: 'logger "EX294 in progress"'
	- name: Start service crond, if not started
	  ansible.builtin.service:
		name: crond
		state: started
		enabled: yes
```

验证计划任务配置：`ansible dev -a "crontab -u devops -l"`
验证计划任务运行：`ansible dev -a "systemctl status crond"
# 18 配置内核参数
- [ ] 创建一个名为` /home/devops/ansible/sysctl.yml` 的 playbook，在所有受管节点上运行并满足以下要求：
	- [ ] 在总内存大于 1.5G 的机器上配置虚拟内存参数 swappiness 的值为10
	- [ ] 在总内存小于 1G 的机器上打印报错：Server memory less than 1024MB

## answer
在`sysctl`模块中找
算1.5 * 1024：`echo 1.5*1024 | bc`
```yaml
---
- name: 配置内核参数
  hosts: all
  tasks:
    - name: 在内存大于 1.5G 的主机上设置 swappiness 参数为 10
      ansible.posix.sysctl:
        name: vm.swappiness
        value: "10"
        state: present
        reload: yes
      when: ansible_memtotal_mb > 1536

    - name: 打印报错信息
      ansible.builtin.debug:
        msg: Server memory less than 1024MB. RAM size = {{  ansible_memtotal_mb  }}
      when: ansible_memtotal_mb < 1024
```

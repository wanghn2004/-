## 一 . 安装前环境准备

### 1 关闭防火墙

1.修改/etc/selinux/config文件中的“SELINUX”值为“disabled”

```
vim /etc/selinux/config

SELINUX=disabled
```

2 .重新启动操作系统。

3 .检查防火墙是否关闭。

```
systemctl status firewalld
```

### 2 设置字符集参数

将各数据库节点的字符集设置为相同的字符集，可以在/etc/profile文件中添加“export LANG=XXX”（XXX为Unicode编码）。

```
vim /etc/profile
export LANG="zh_CN.UTF-8"
```

### 3 关闭swap交换空间

```
swapoff -a
```

### 4 关闭RemoveIPC

在各数据库节点上，关闭RemoveIPC。CentOS操作系统默认为关闭，可以跳过该步骤。

1.修改/etc/systemd/logind.conf文件中的“RemoveIPC”值为“no”。

```
vim /etc/systemd/logind.conf

RemoveIPC=no
```

2.修改/usr/lib/systemd/system/systemd-logind.service文件中的“RemoveIPC”值为“no”。

```
vim /usr/lib/systemd/system/systemd-logind.service

RemoveIPC=no
```

3.重新加载配置参数。

```
systemctl daemon-reload
systemctl restart systemd-logind
```

4.检查修改是否生效。

```
loginctl show-session | grep RemoveIPC
systemctl show systemd-logind | grep RemoveIPC
```

### 5.创建用户和组

```
# 创建组
groupadd dbgroup
# 创建omm用户
useradd -g dbgroup omm
# 设置数据库密码     Telewave@1234
passwd omm
```

### 6.创建安装路径并授权

```
# 创建数据库安装路径
mkdir -p /usr/local/openGauss
# 为安装路径及文件授权
chown 755 -R /usr/local/telewave
# 为omm用户授权安装路径权限
chown -R omm:dbgroup /usr/local/openGauss

```

### 7.上传至主机目录

```
# 上传至主机目录 /usr/local/并授权
chown -R omm:dbgroup /usr/local/openGauss-3.0.5-openEuler-64bit.tar.bz2

```

```
https://opengauss.org/zh/download/archive/

官网下载安装包 openGauss-3.0.5-openEuler-64bit.tar.bz2
```

## 二. 安装openGauss数据库

### 1. 数据库安装

（1）.切换至omm用户解压openGauss压缩包到安装目录。

```
su omm
tar -jxf openGauss-3.0.5-openEuler-64bit.tar.bz2 -C /usr/local/openGauss

```

（2）.进入解压后目录下的simpleInstall（/usr/local/openGauss/simpleInstall）。

```
cd /usr/local/openGauss/simpleInstall
```

（3）.执行install.sh脚本安装openGauss。

```
sh install.sh  -w "user@1234" &&source ~/.bashrc

no
```

- -w：初始化数据库密码（gs_initdb指定），安全需要必须设置。
- -p：指定的openGauss端口号，如不指定，默认为5432。

（4）.安装执行完成后，使用ps和gs_ctl查看进程是否正常。

```bash
# 切换用户至root，然后再切换至omm进行查看数据库状态
ps ux | grep gaussdb
gs_ctl query -D /usr/local/openGauss/data/single_node
```

### 2. 数据库启动、重启、停止命令

```
# 进入数据库安装路径的bin目录
cd /usr/local/openGauss/bin

# 查看状态
gs_ctl status -D /usr/local/openGauss/data/single_node/

# 启动
gs_ctl start -D /usr/local/openGauss/data/single_node/

# 重启
gs_ctl restart -D /usr/local/openGauss/data/single_node/

# 停止
gs_ctl stop -D /usr/local/openGauss/data/single_node/


```

### 3. 修改配置允许远程连接

```
# 1.文件 pg_hba.conf 修改
vim /usr/local/openGauss/data/single_node/pg_hba.conf
# 允许所有网段连接 在IPv4 local connections下添加
host  all    all    0.0.0.0/0    sha256
host  all    all    0.0.0.0/0    md5
 
# 2.重新加载 gs_ctl 策略
su omm
cd /usr/local/openGauss/bin
gs_ctl reload -D /usr/local/openGauss/data/single_node
 
# 3.文件 postgresql.conf 修改
vim /usr/local/openGauss/data/single_node/postgresql.conf
# 找到 listen_addresses 变量，将前面#去掉
listen_addresses = '*'
# 找到 password_encryption_type 变量，将前面#去掉
password_encryption_type  = 1
 
# 4. 重启数据库
su omm
cd /usr/local/openGauss/bin
gs_ctl restart -D  /usr/local/openGauss/data/single_node

```

### 4. 创建数据库远程连接用户

```
# 进入数据库安装路径的bin目录
cd /usr/local/openGauss/bin
# 进入数据库
gsql -d postgres -U omm -p 5432

# 创建远程连接用户 postgres
CREATE ROLE postgres LOGIN PASSWORD 'Run@1234';
# 设置postgres为管理员
GRANT ALL PRIVILEGES TO postgres;

ALTER USER postgres SET search_path = public, pg_catalog;

```

## Debian依赖

1. 从 CentOS 7 服务器拷贝以下文件：
   - /usr/lib64/libssl.so.10
   - /usr/lib64/libcrypto.so.10
2. 上传到麒麟服务器的/usr/lib/x86_64-linux-gnu/目录

```
su - root
mv /path/to/centos/libssl.so.10 /usr/lib/x86_64-linux-gnu/
mv /path/to/centos/libcrypto.so.10 /usr/lib/x86_64-linux-gnu/
# 更新库缓存
ldconfig
```

  3.安装依赖

```
cd /usr/local/gaosi_yl
dpkg -i *.deb 
```

  4.检查openGauss依赖

```
ldd /usr/local/openGauss/bin/gaussdb | grep "not found"
```


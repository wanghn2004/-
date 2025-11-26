## MySQL 5.7 二进制安装步骤



| **安装目录**         | `/home/software/mysql-5.7.44-linux-glibc2.12-x86_64`         | MySQL 程序文件的根目录                   |
| -------------------- | ------------------------------------------------------------ | ---------------------------------------- |
| **数据目录**         | `/data/mysql`                                                | 存储数据库表、索引等数据文件             |
| **日志文件**         | `/home/software/mysql-5.7.44-linux-glibc2.12-x86_64/mysqld.log` | 记录 MySQL 运行日志                      |
| **配置文件**         | `/etc/my.cnf`                                                | MySQL 的核心配置文件                     |
| **可执行文件目录**   | `/home/software/mysql-5.7.44-linux-glibc2.12-x86_64/bin`     | 包含`mysql`、`mysqld`等命令              |
| **systemd 服务文件** | `/etc/systemd/system/mysqld.service`                         | 用于系统服务管理（启动、停止、开机自启） |
| **环境变量**         | 已将`/home/software/mysql-5.7.44-linux-glibc2.12-x86_64/bin`添加到`/etc/profile` | 可直接在终端执行`mysql`等命令            |

- 启动服务：`systemctl start mysqld`
- 停止服务：`systemctl stop mysqld`
- 重启服务：`systemctl restart mysqld`
- 查看状态：`systemctl status mysqld`
- 远程连接：`mysql -uroot -p123456 -h 服务器IP -P 3306`

### 步骤 1：安装依赖 + 创建目录

```bash
# 安装libaio依赖
rpm -ivh libaio-0.3.109-13.el7.x86_64.rpm --force --nodeps

# 创建所需目录
mkdir -p /home/software /data/mysql /var/run/mysqld /home/mysql
# 配置目录权限
chmod 777 /var/run/mysqld /home/mysql
```

------

### 步骤 2：创建 mysql 用户组

```
# 新建mysql组（已存在则忽略报错）
groupadd -r mysql 2>/dev/null
# 新建mysql用户（已存在则忽略报错）
useradd -r -g mysql -s /sbin/nologin -d /home/mysql mysql 2>/dev/null
# 授权家目录
chown -R mysql:mysql /home/mysql
```



### 步骤 3：解压安装包

```bash
# 解压到/home/software
tar -zxvf mysql-5.7.44-linux-glibc2.12-x86_64.tar.gz -C /home/software

# 授权安装目录
chown -R mysql:mysql /home/software/mysql-5.7.44-linux-glibc2.12-x86_64
chown -R mysql:mysql /data/mysql /var/run/mysqld

# 给执行文件加权限
chmod 755 /home/software/mysql-5.7.44-linux-glibc2.12-x86_64/bin/*
```

### 步骤 4：生成配置文件

```
# 写入配置到/etc/my.cnf（覆盖原有文件）
cat > /etc/my.cnf << EOF
[mysqld]
bind-address=0.0.0.0
port=3306
basedir=/home/software/mysql-5.7.44-linux-glibc2.12-x86_64
datadir=/data/mysql
log-error=/home/software/mysql-5.7.44-linux-glibc2.12-x86_64/mysqld.log
pid-file=/var/run/mysqld/mysqld.pid
socket=/tmp/mysql.sock
symbolic-links=0
lower_case_table_names=1

[mysql]
default-character-set=utf8
socket=/tmp/mysql.sock
EOF
```

### 步骤 5：初始化数据库

```
# 清空数据目录（确保干净）
rm -rf /data/mysql/*
chown mysql:mysql /data/mysql

# 用mysql用户初始化（生成临时密码）
su - mysql -s /bin/bash -c "/home/software/mysql-5.7.44-linux-glibc2.12-x86_64/bin/mysqld --defaults-file=/etc/my.cnf --initialize"

# 查看临时密码（复制保存，下一步用）
grep "temporary password" /home/software/mysql-5.7.44-linux-glibc2.12-x86_64/mysqld.log
```

### 步骤6:启动mysql服务

`mysql.server` 是一个 Shell 脚本，它本质上也是调用 `mysqld_safe` 来启动 MySQL。

###### 1. 找到 `mysql.server` 路径

- 在 RHEL/CentOS 系统中，`mysql.server` 可能位于 `/usr/share/mysql/mysql.server`。
- 在源码安装的 MySQL 中，`mysql.server` 通常在 `support-files` 目录下，如 `/usr/local/mysql/support-files/mysql.server`。

###### 2. 启动命令

```bash
# 示例路径，请根据实际路径修改
/usr/share/mysql/mysql.server start
```

```
# 示例路径，请根据实际路径修改
/usr/sbin/mysqld --defaults-file=/etc/my.cnf &
使用 mysqld_safe 启动（推荐）
./mysqld_safe --defaults-file=/etc/my.cnf &

# 用mysql用户启动服务
su - mysql -s /bin/bash -c "/home/software/mysql-5.7.44-linux-glibc2.12-x86_64/bin/mysqld_safe --defaults-file=/etc/my.cnf &"

# 等待25秒（让服务启动稳定）
sleep 25

# 验证启动（有输出则成功）
ps -ef | grep -v grep | grep mysqld
netstat -tulpn | grep 3306
```

### 步骤7:配置systemctl

```
# 写入服务文件
cat > /etc/systemd/system/mysqld.service << EOF
[Unit]
Description=MySQL 5.7.44 Server
After=network.target

[Service]
Type=forking
User=mysql
Group=mysql
ExecStart=/home/software/mysql-5.7.44-linux-glibc2.12-x86_64/bin/mysqld_safe --defaults-file=/etc/my.cnf
ExecStop=/home/software/mysql-5.7.44-linux-glibc2.12-x86_64/bin/mysqladmin -uroot -p123456 --socket=/tmp/mysql.sock shutdown
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

# 重载配置+设置开机自启
systemctl daemon-reload
systemctl enable mysqld
```

### 步骤 8：修改密码 + 开启远程访问

```
#首先，使用临时密码登录 MySQL 服务器
/home/software/mysql-5.7.44-linux-glibc2.12-x86_64/bin/mysql -uroot -p --connect-expired-password


#修改 root 用户的密码
ALTER USER 'root'@'localhost' IDENTIFIED BY '123456';

#授权 root 用户允许远程连接
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY '123456' WITH GRANT OPTION;
#刷新权限，使更改生效
FLUSH PRIVILEGES;
```

### 步骤 9：配置环境变量（直接用 mysql 命令）

```
# 添加环境变量
echo "export PATH=\$PATH:/home/software/mysql-5.7.44-linux-glibc2.12-x86_64/bin" >> /etc/profile
echo "export PATH=\$PATH:/home/software/mysql-5.7.44-linux-glibc2.12-x86_64/bin" >> ~/.bashrc

# 生效环境变量
source /etc/profile
source ~/.bashrc
```

------

###  验证安装

```bash
mysql -u root -p\'你的新密码\' -e "SELECT VERSION();"
```

如果成功显示版本号，则安装完成。你也可以检查日志目录是否生成了日志文件：

```bash
ls -l /usr/local/mysql/log/
```

------

### **常见问题**

1. #### **启动失败**：

   - 检查日志：`tail -n 20 /usr/local/mysql/log/mysqld.err`
   - 检查权限：`ls -ld /usr/local/mysql/data /usr/local/mysql/log`，确保所有者是 `mysql:mysql`。

2. #### **忘记密码**：

   - 停止 MySQL：`systemctl stop mysqld`
   - 以跳过权限验证方式启动：`/usr/local/mysql/bin/mysqld_safe --skip-grant-tables --skip-networking &`
   - 登录并修改密码：

   ```bash
   mysql -u root
   USE mysql;
   UPDATE user SET authentication_string = PASSWORD(\'新密码\') WHERE user = \'root\';
   FLUSH PRIVILEGES;
   exit;
   ```

   - 重启 MySQL：`systemctl restart mysqld`

   3. **libncurses.so.5**

      ```
      libncurses.so.5
      
      error while loading shared libraries: libncurses.so.5: cannot open shared object file: No such file or directory
      ```

      - MySQL 5.7 基于旧版 `ncurses 5.x` 开发，依赖 `libncurses.so.5` 和 `libtinfo.so.5`

      - openEuler 22.03 预装 `ncurses 6.x`，仅提供 `libncurses.so.6` 和 `libtinfo.so.6`，但新版本库完全兼容旧版本接口，仅需通过软链接复用即可

        解决 `libncurses.so.5` 缺失

      ```bash
      # 1. 确认系统已有的新版本库位置（已验证存在 /usr/lib64/libncurses.so.6）
      find /usr/lib64 -name "libncurses.so.6*"
      # 2. 创建软链接（复用 libncurses.so.6）
      ln -s /usr/lib64/libncurses.so.6 /usr/lib64/libncurses.so.5
      # 3. 刷新系统库缓存，让链接生效
      ldconfig
      ```

      


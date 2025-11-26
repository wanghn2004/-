# MySQL 导入 SQL 文件 核心命令整理

 MySQL 5.7.44 离线版 + utf8mb4 字符集

## 一、前置准备命令（需先执行）

### 1. 创建目标数据库（若不存在）

```bash
# 登录 MySQL 并创建数据库（字符集统一 utf8mb4）
/home/software/mysql-5.7.44-linux-glibc2.12-x86_64/bin/mysql -u root -p
```

登录后执行 SQL

```sql
CREATE DATABASE IF NOT EXISTS dzp_exchange CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
exit;
```

## 二、核心导入命令（按场景选择）

### 场景 1：基础导入（SQL 明文文件，单表 / 全库通用）

```bash
# 替换：dzp_exchange=目标数据库名，/root/exports/xxx.sql=SQL文件绝对路径
/home/software/mysql-5.7.44-linux-glibc2.12-x86_64/bin/


mysql -u root -p --default-character-set=utf8mb4 dzp_exchange < /home/dzp.sql
```

### 场景 2：导入压缩文件（.sql.gz，无需解压）

```bash
# 替换：压缩文件路径、目标数据库名
gzip -d -c /root/exports/t_fsbillactive_full.sql.gz | /home/software/mysql-5.7.44-linux-glibc2.12-x86_64/bin/mysql -u root -p --default-character-set=utf8mb4 dzp_exchange
```

### 场景 3：内网远程传输 + 导入（从另一台机器拷贝并导入

```bash
# 步骤 1：先拷贝远程机器的 SQL 文件到本地
scp root@192.168.1.101:/root/exports/t_fsbillactive_full.sql /root/exports/

# 步骤 2：执行导入（同场景 1）
/home/software/mysql-5.7.44-linux-glibc2.12-x86_64/bin/mysql -u root -p --default-character-set=utf8mb4 dzp_exchange < /root/exports/t_fsbillactive_full.sql
```

### 场景 4：强制覆盖已有表（谨慎使用！先删旧表再导入）

```bash
# 替换：SQL文件路径、目标数据库名
echo "DROP TABLE IF EXISTS t_fsbillactive;" | /home/software/mysql-5.7.44-linux-glibc2.12-x86_64/bin/mysql -u root -p --default-character-set=utf8mb4 dzp_exchange && /home/software/mysql-5.7.44-linux-glibc2.12-x86_64/bin/mysql -u root -p --default-character-set=utf8mb4 dzp_exchange < /root/exports/t_fsbillactive_full.sql
```

## 三、关键说明

1. 所有命令执行后，会提示 `Enter password:`，输入 MySQL root 密码即可；
2. 替换占位符：`dzp_exchange`（目标数据库名）、`/root/exports/xxx.sql`（SQL 文件实际路径）、`192.168.1.101`（远程机器内网 IP）；
3. 无报错返回命令行，即导入成功；若报错，优先检查文件路径、数据库权限、字符集是否一致。

```
 mysqldump -u root -p --default-character-set=utf8mb4 --databases exchange --tables t_fsbillactive --where="ischecked=0" > /data/mysql/t_fsbillactive_full.sql
```


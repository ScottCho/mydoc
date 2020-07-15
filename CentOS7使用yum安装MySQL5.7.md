**CentOS7使用yum安装MySQL5.7**

参考： https://dev.mysql.com/doc/refman/5.7/en/linux-installation-yum-repo.html

1.  建立yum仓库

vim /etc/yum.repos.d/mysql-community.repo

```bash
[mysql57-community]
name=MySQL 5.7 Community Server
baseurl=http://repo.mysql.com/yum/mysql-5.7-community/el/7/$basearch/
enabled=1
gpgcheck=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-mysql
```



2.  安装MySQL

这将安装MySQL服务器的软件包（`mysql-community-server`）以及运行服务器所需的组件的软件包，包括客户端的软件包（`mysql-community-client`），客户端和服务器的常见错误消息和字符集（`mysql-community-common`）以及共享的客户端库（`mysql-community-libs`)

```bash
yum install mysql-community-server
```



3. 检查和修改配置文件

可以在该文件配置数据文件存放位置

```bash
/etc/my.cnf
```



4. 启动MySQL

```bash
systemctl start mysqld
```

- 服务器已初始化。
- SSL证书和密钥文件在数据目录中生成。
- [`validate_password`](https://dev.mysql.com/doc/refman/5.7/en/validate-password.html) 已安装并启用。
- `'root'@'localhost`创建 一个超级用户帐户。设置超级用户的密码并将其存储在错误日志文件中.



5. 初始化密码

   查看初始的密码

```bash
grep 'temporary password' /var/log/mysqld.log
```

通过使用生成的临时密码登录并尽快为超级用户帐户设置自定义密码，以更改root密码：

```bash
shell> mysql -uroot -p
mysql> ALTER USER 'root'@'localhost' IDENTIFIED BY 'MyNewPass4!';
```



6. 重置密码

   停止MySQL,然后以--skip-grant-tables 启动

   ```shell
   mysqld --skip-grant-tables --skip_networking  --user mysql&
   ```

   使用mysql客户端连接到MySQL服务器

   ```
   mysql
   ```

   在`mysql`客户端中，告诉服务器重新加载授权表，以便帐户管理语句起作用：

   ```bash
   mysql> FLUSH PRIVILEGES;
   ```

   然后更改`'root'@'localhost'` 帐户密码。用您要使用的密码替换密码。要更改`root`具有不同主机名部分的帐户的密码 ，请修改说明以使用该主机名。

   ```bash
   mysql> ALTER USER 'root'@'localhost' IDENTIFIED BY 'MyNewPass';
   ```

   语句未能重置密码，请尝试使用以下语句重复该过程以`user`直接修改该 表：

   ```bash
   UPDATE mysql.user SET authentication_string = PASSWORD('MyNewPass')
   WHERE User = 'root' AND Host = 'localhost';
   FLUSH PRIVILEGES;
   ```

   重启mysql。

   

   




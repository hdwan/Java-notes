## 准备工作

阿里云：实例安全组的入方向规则已放行**22、80、443、3306**端口。

MySQL相关**安装路径说明**如下：

- 配置文件：/etc/my.cnf
- 数据存储：/var/lib/mysql
- 命令文件：/usr/bin和/usr/sbin

## 安装

1. 更新YUM源。

    ```shell
    sudo rpm -Uvh https://dev.mysql.com/get/mysql80-community-release-el7-7.noarch.rpm
    ```

2. （可选）当操作系统为Alibaba Cloud Linux 3时，安装MySQl所需库文件。

    ```shell
    sudo rpm -Uvh https://mirrors.aliyun.com/alinux/3/updates/x86_64/Packages/compat-openssl10-1.0.2o-4.0.1.al8.x86_64.rpm
    ```

3. 运行以下命令，安装MySQL。

    ```shell
    sudo yum -y install mysql-community-server --enablerepo=mysql80-community --nogpgcheck
    ```

4. 查看MySQL版本号。

    ```shell
    mysql -V
    ```

    返回结果：

    ```shell
    mysql Ver 8.0.33 for Linux on x86_64 (MySQL Community Server - GPL)
    ```

## 配置MySQL

1. 启动并设置开机自启动MySQL服务。

    ```shell
    sudo systemctl start mysqld
    sudo systemctl enable mysqld
    ```

2. 获取并记录root用户的初始密码。

    ```shell
    sudo grep 'temporary password' /var/log/mysqld.log
    ```

    执行命令结果示例如下。

    ```shell
    2022-02-14T09:27:18.470008Z 6 [Note] [MY-010454] [Server] A temporary password is generated for root@localhost: r_V&f2wyu_vI
    ```

    **说明**

    示例末尾的`r_V&f2wyu_vI`为初始密码，后续在对MySQL进行安全性配置时，需要使用该初始密码。

3. 对MySQL进行安全性配置。

    ```shell
    sudo mysql_secure_installation
    ```

    1. 根据提示信息，重置MySQL数据库root用户的密码。

        ```shell
        Enter password for user root: #输入已获取的root用户初始密码
        
        The existing password for the user account root has expired. Please set a new password.
        
        New password: #输入新的MySQL密码
        
        Re-enter new password: #重复输入新的MySQL密码
        The 'validate_password' component is installed on the server.
        The subsequent steps will run with the existing configuration
        of the component.
        Using existing password for root.
        Change the password for root ? ((Press y|Y for Yes, any other key for No) :Y #输入Y选择更新MySQL密码。您也可以输入N不再更新MySQL密码。
        
        New password: #输入新的MySQL密码
        
        Re-enter new password: #重复输入新的MySQL密码
        
        Estimated strength of the password: 100
        Do you wish to continue with the password provided?(Press y|Y for Yes, any other key for No) :Y #输入Y确认使用已设置的密码。
        ```

    2. 根据提示信息，删除匿名用户。

        ```shell
        By default, a MySQL installation has an anonymous user,
        allowing anyone to log into MySQL without having to have
        a user account created for them. This is intended only for
        testing, and to make the installation go a bit smoother.
        You should remove them before moving into a production
        environment.
        
        Remove anonymous users? (Press y|Y for Yes, any other key for No) :Y #输入Y删除MySQL默认的匿名用户。
        Success.
        ```

    3. 禁止root账号远程登录。

        ```shell
        Normally, root should only be allowed to connect from
        'localhost'. This ensures that someone cannot guess at
        the root password from the network.
        
        Disallow root login remotely? (Press y|Y for Yes, any other key for No) :Y #输入Y禁止root远程登录。
        Success.
        ```

    4. 删除test库以及对test库的访问权限。

        ```shell
        By default, MySQL comes with a database named 'test' that
        anyone can access. This is also intended only for testing,
        and should be removed before moving into a production
        environment.
        
        
        Remove test database and access to it? (Press y|Y for Yes, any other key for No) :Y #输入Y删除test库以及对test库的访问权限。
         - Dropping test database...
        Success.
        
         - Removing privileges on test database...
        Success.
        ```

    5. 重新加载授权表。

        ```shell
        Reloading the privilege tables will ensure that all changes
        made so far will take effect immediately.
        
        Reload privilege tables now? (Press y|Y for Yes, any other key for No) :Y #输入Y重新加载授权表。
        Success.
        
        All done!
        ```

## 远程访问MySQL数据库

最好使用非root账号远程登录MySQL数据库。

1. 运行以下命令后，输入root用户的密码登录MySQL。

    ```shell
    sudo mysql -uroot -p
    ```

2. 创建远程登录MySQL的账号，并允许远程主机使用该账号访问MySQL。

    示例账号为`dmsTest`、密码为`Ecs@123****`。

    **密码要求**：长度为8至30个字符，必须同时包含大小写英文字母、数字和特殊符号。

    ```shell
    #创建数据库用户dmsTest,并授予远程连接权限。
    create user 'dmsTest'@'%' identified by 'Ecs@123****'; 
    #为dmsTest用户授权数据库所有权限。
    grant all privileges on *.* to 'dmsTest'@'%'; 
    #刷新权限。
    flush privileges; 
    ```

3. 退出数据库。

    ```shell
    exit
    ```

4. 使用`dmsTest`账号远程登录MySQL（用Navicat等）

## CentOS 8.x

### 安装MySQL

1. 切换CentOS 8源地址。

    CentOS 8操作系统版本结束了生命周期（EOL），按照社区规则，CentOS 8的源地址http://mirror.centos.org/centos/8/内容已移除，您在阿里云上继续使用默认配置的CentOS 8的源会发生报错。如果您需要使用CentOS 8系统中的一些安装包，则需要手动切换源地址。具体操作，请参见[CentOS 8 EOL如何切换源？](https://help.aliyun.com/zh/ecs/user-guide/change-centos-8-repository-addresses#task-2182261)。

2. 安装MySQL。

    ```shell
    sudo dnf -y install @mysql
    ```

3. 查看MySQL版本信息。

    ```shell
    mysql -V
    ```

### 配置MySQL

1. 启动并设置开机自启动MySQL服务。

    ```shell
    sudo systemctl start mysqld
    sudo systemctl enable mysqld
    ```

2. 对MySQL进行安全性配置。

    ```shell
    sudo mysql_secure_installation
    ```

    1. 根据提示信息，重置MySQL数据库root用户的密码。

        ```shell
        Press y|Y for Yes, any other key for No: y  #输入Y并回车开始相关配置。
        
        There are three levels of password validation policy:
        
        LOW Length >= 8
        MEDIUM Length >= 8, numeric, mixed case, and special characters
        STRONG Length >= 8, numeric, mixed case, special characters and dictionary file
        
        Please enter 0 = LOW, 1 = MEDIUM and 2 = STRONG: 2   #选择密码验证策略强度，策略0表示低，1表示中，2表示高。建议您选择高强度的密码验证策略。
        Please set the password for root here.
        
        New password:     #设置MySQL的新密码
        
        Re-enter new password:    #重复输入新的MySQL密码
        
        Estimated strength of the password: 100 
        Do you wish to continue with the password provided?(Press y|Y for Yes, any other key for No) : y   ##输入Y确认使用已设置的密码。
        ```

    2. 根据提示信息，删除匿名用户。

        ```shell
        By default, a MySQL installation has an anonymous user,
        allowing anyone to log into MySQL without having to have
        a user account created for them. This is intended only for
        testing, and to make the installation go a bit smoother.
        You should remove them before moving into a production
        environment.
        
        Remove anonymous users? (Press y|Y for Yes, any other key for No) :Y #输入Y删除MySQL默认的匿名用户。
        Success.
        ```

    3. 禁止root账号远程登录。

        ```shell
        Normally, root should only be allowed to connect from
        'localhost'. This ensures that someone cannot guess at
        the root password from the network.
        
        Disallow root login remotely? (Press y|Y for Yes, any other key for No) :Y #输入Y禁止root远程登录。
        Success.
        ```

    4. 删除test库以及对test库的访问权限。

        ```shell
        By default, MySQL comes with a database named 'test' that
        anyone can access. This is also intended only for testing,
        and should be removed before moving into a production
        environment.
        
        
        Remove test database and access to it? (Press y|Y for Yes, any other key for No) :Y #输入Y删除test库以及对test库的访问权限。
         - Dropping test database...
        Success.
        
         - Removing privileges on test database...
        Success.
        ```

    5. 重新加载授权表。

        ```shell
        Reloading the privilege tables will ensure that all changes
        made so far will take effect immediately.
        
        Reload privilege tables now? (Press y|Y for Yes, any other key for No) :Y #输入Y重新加载授权表。
        Success.
        
        All done!
        ```

### 远程访问MySQL数据库

建议使用非root账号远程登录MySQL数据库。

1. 运行以下命令后，输入root用户的密码登录MySQL。

    ```shell
    sudo mysql -uroot -p
    ```

2. 依次运行以下命令，创建远程登录MySQL的账号，并允许远程主机使用该账号访问MySQL。

    本示例账号为`dmsTest`、密码为`Ecs@123****`。

    密码要求：长度为8至30个字符，必须同时包含大小写英文字母、数字和特殊符号。

    ```shell
    #创建数据库用户dmsTest,并授予远程连接权限。
    create user 'dmsTest'@'%' identified by 'Ecs@123****'; 
    #为dmsTest用户授权数据库所有权限。
    grant all privileges on *.* to 'dmsTest'@'%'; 
    #刷新权限。
    flush privileges; 
    ```

3. 执行以下命令，退出数据库。

    ```shell
    exit
    ```

4. 使用`dmsTest`账号远程登录MySQL（Navicat）。
## 环境

1. Tomcat 版本：Tomcat 8.5.88；
2. JDK 版本：JDK 1.8；

## 准备工作

### 1、放行端口

SSH协议的22端口和HTTP协议的8080端口。

### 2、关闭防火墙

1、**临时**关闭防火墙：

```shell
systemctl stop firewalld
```

该操作只是暂时关闭防火墙，下次重启Linux后，防火墙还会开启。

2、**永久**关闭防火墙：

1. 关闭当前运行中的防火墙。

    ```shell
    systemctl stop firewalld
    ```

2. 关闭防火墙服务。

    ```shell
    systemctl disable firewalld
    ```



**查看防火墙状态：**

```shell
systemctl status firewalld
```

`Unit firewalld.service could not be found.`表示没有防火墙服务，如果要用得到的话需要自己下载。

**启动**防火墙：

```shell
service firewalld start
```

### 3、关闭SELinux

安全增强型 Linux（Security-Enhanced Linux）简称 SELinux，它是一个 Linux 内核模块，也是 Linux 的一个安全子系统。用不到，直接关掉。

1. 查看SELinux的当前状态：

```shell
getenforce
```

结果示例如图所示：![查看SELinux状态](https://help-static-aliyun-doc.aliyuncs.com/assets/img/zh-CN/7712359951/p21065.png)

- 如果SELinux状态参数是Disabled， 则SELinux为关闭状态。
- 如果SELinux状态参数是Enforcing，则SELinux为开启状态。

2、**临时**关闭SELinux：

```shell
setenforce 0
```

3、**永久**关闭：

- 打开SELinux配置文件：

    ```shell
    vi /etc/selinux/config
    ```

- 将光标移动到`SELINUX=enforcing`一行，按 `i`键进入编辑模式，修改为`SELINUX=disabled`，然后按 `Esc` 键，再输入`:wq`并回车，保存关闭SELinux配置文件。

重启系统使设置生效。

## 安装jdk

1. [官网](https://www.oracle.com/java/technologies/javase/javase8u211-later-archive-downloads.html)下载服务器系统版本对应的jdk压缩包，然后上传到服务器

2. 在jdk安装路径进行解压：

    ```shell
    tar -xvzf 压缩包名字
    // 移动压缩包或者解压后的文件夹
    mv 压缩包名字或者文件夹路径 目标路径
    ```

3. 配置环境变量并更新

    ```shell
    #在profile文件末尾添加
    export JAVA_HOME=/usr/java/jdk1.8.0_65 // 注意这里改为自己的安装路径
    export PATH=$PATH:$JAVA_HOME/bin
    export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
    ```

    重新生效profile

    ```shell
    source /etc/profile
    ```

4. 查看是否安装成功。

    ```shell
    java -version
    ```

    如果显示如下所示，则表示JDK安装成功。

    ```shell
    # java -version
    java version "1.8.0_381"
    Java(TM) SE Runtime Environment (build 1.8.0_381-b09)
    Java HotSpot(TM) 64-Bit Server VM (build 25.381-b09, mixed mode)
    ```

    **注意**：

    修改环境变量后，可能会导致执行其他命令时，出现类似`-bash: chmod: command not found`这样的问题，执行**export PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin:/root/bin**即可。

## 安装Tomcat

1. 下载并解压Tomcat 8安装包：

    ```shell
    wget https://archive.apache.org/dist/tomcat/tomcat-8/v8.5.88/bin/apache-tomcat-8.5.88.tar.gz  --no-check-certificate 
    tar -zxvf apache-tomcat-8.5.88.tar.gz
    ```

    **说明**

    Tomcat下载地址官网会持续更新。如果上述下载地址失效，直接访问[Tomcat官网](https://tomcat.apache.org/)获取。

2. 移动Tomcat所在目录：

    ```shell
    sudo mv apache-tomcat-8.5.88 /usr/local/tomcat/
    ```

3. 为保证系统安全性，建议创建一般用户来运行Tomcat。

    例如，本示例中创建一般用户www。

    ```bash
    useradd www
    ```

4. 运行以下命令，将文件的所属用户设置为www。

    ```bash
    chown -R www.www /usr/local/tomcat/
    ```

    在/usr/local/tomcat/目录下：

    - bin：存放Tomcat的一些脚本文件，包含启动和关闭Tomcat服务脚本。
    - conf：存放Tomcat服务器的各种全局配置文件，其中最重要的是server.xml和web.xml。
    - webapps：Tomcat的主要Web发布目录，默认情况下把Web应用文件放于此目录。
    - logs：存放Tomcat执行时的日志文件。

5. 配置server.xml文件。

    1. 运行以下命令，切换到/usr/local/tomcat/conf/目录。

        ```shell
        cd /usr/local/tomcat/conf/
        ```

    2. 运行以下命令，重命名server.xml文件。

        ```shell
        mv server.xml server.xml_bk
        ```

    3. 新建一个server.xml文件。

        1. 运行以下命令，创建并打开server.xml文件。

            ```shell
            vi server.xml
            ```

        2. 按下i键，添加以下内容。

            ```shell
            <?xml version="1.0" encoding="UTF-8"?>
            <Server port="8006" shutdown="SHUTDOWN">
            <Listener className="org.apache.catalina.core.JreMemoryLeakPreventionListener"/>
            <Listener className="org.apache.catalina.mbeans.GlobalResourcesLifecycleListener"/>
            <Listener className="org.apache.catalina.core.ThreadLocalLeakPreventionListener"/>
            <Listener className="org.apache.catalina.core.AprLifecycleListener"/>
            <GlobalNamingResources>
            <Resource name="UserDatabase" auth="Container"
             type="org.apache.catalina.UserDatabase"
             description="User database that can be updated and saved"
             factory="org.apache.catalina.users.MemoryUserDatabaseFactory"
             pathname="conf/tomcat-users.xml"/>
            </GlobalNamingResources>
            <Service name="Catalina">
            <Connector port="8080"
             protocol="HTTP/1.1"
             connectionTimeout="20000"
             redirectPort="8443"
             maxThreads="1000"
             minSpareThreads="20"
             acceptCount="1000"
             maxHttpHeaderSize="65536"
             debug="0"
             disableUploadTimeout="true"
             useBodyEncodingForURI="true"
             enableLookups="false"
             URIEncoding="UTF-8"/>
            <Engine name="Catalina" defaultHost="localhost">
            <Realm className="org.apache.catalina.realm.LockOutRealm">
            <Realm className="org.apache.catalina.realm.UserDatabaseRealm"
              resourceName="UserDatabase"/>
            </Realm>
            <Host name="localhost" appBase="/data/wwwroot/default" unpackWARs="true" autoDeploy="true">
            <Context path="" docBase="/data/wwwroot/default" debug="0" reloadable="false" crossContext="true"/>
            <Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"
            prefix="localhost_access_log." suffix=".txt" pattern="%h %l %u %t &quot;%r&quot; %s %b" />
            </Host>
            </Engine>
            </Service>
            </Server>
            ```

        3. 按Esc键，输入`:wq`并回车以保存并关闭文件。

6. 设置JVM内存参数。

    1. 运行以下命令，创建并打开/usr/local/tomcat/bin/setenv.sh文件。

        ```shell
        vi /usr/local/tomcat/bin/setenv.sh
        ```

    2. 按下i键，添加以下内容。

        指定`JAVA_OPTS`参数，用于设置JVM的内存信息以及编码格式。

        ```bash
        JAVA_OPTS='-Djava.security.egd=file:/dev/./urandom -server -Xms256m -Xmx496m -Dfile.encoding=UTF-8'
        ```

    3. 按下Esc键，输入`:wq`并回车以保存并关闭文件。

7. 设置Tomcat自启动脚本。

    1. 运行以下命令，下载Tomcat自启动脚本。

        如果下载失败，浏览器访问`https://raw.githubusercontent.com/oneinstack/oneinstack/master/init.d/Tomcat-init`直接获取脚本内容。

        ```shell
        wget https://raw.githubusercontent.com/oneinstack/oneinstack/master/init.d/Tomcat-init
        ```

    2. 运行以下命令，移动并重命名Tomcat-init。

        ```shell
        mv Tomcat-init /etc/init.d/tomcat
        ```

    3. 运行以下命令，为/etc/init.d/tomcat添加可执行权限。

        ```shell
        chmod +x /etc/init.d/tomcat
        ```

    4. 运行以下命令，设置启动脚本JAVA_HOME。

        **重要**

        **脚本中JDK的版本信息必须与安装的JDK版本信息一致**，否则Tomcat会启动失败。

        ```shell
        sed -i 's@^export JAVA_HOME=.*@export JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.342.b07-1.el7_9.x86_64@' /etc/init.d/tomcat                  
        ```

8. 设置Tomcat开机自启动：

    ```shell
    chkconfig --add tomcat
    chkconfig tomcat on
    ```

9. 启动Tomcat：

    ```bash
    service tomcat start             
    ```

    回显信息类似如下所示，表示启动Tomcat成功。

    ```bash
    [root@test000**** conf]# service tomcat start
    Starting tomcat
    Using CATALINA_BASE:   /usr/local/tomcat
    Using CATALINA_HOME:   /usr/local/tomcat
    Using CATALINA_TMPDIR: /usr/local/tomcat/temp
    Using JRE_HOME:        /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.342.b07-1.el7_9.x86_64
    Using CLASSPATH:       /usr/local/tomcat/bin/bootstrap.jar:/usr/local/tomcat/bin/tomcat-juli.jar
    Using CATALINA_OPTS:
    Tomcat started.
    Tomcat is running with pid: 11837            
    ```

## 安装MySQL

1. 查看是否已安装，如果曾经安装了，卸载：

    ```shell
    // 查看当前系统中的mysql
    rpm -qa | grep -i mysql 
    // 卸载
    rpm -e mysql-libs-5.1.71-1.e16.x68_64（根据上面查看的内容更改） --nodeps
    ```

2. 安装：

    ```
    sudo apt install mysql-server
    ```

3. 查看mysql的状态：

    ```
    sudo service mysql status
    ```

4. 设置mysql密码：

    ```
    sudo mysql // 进入mysql
    ALTER USER '用户名' IDENTIFIED WITH mysql_native_password BY '密码';
    ```

5. 登录mysql：

    说明：设置了mysql密码后，就无法通过` sudo mysql`命令进入了，需要使用下面的输入密码的方式进入。

    ```
    mysql -u root -p
    ```

    如果想回到用`sudo mysql`的方式登录，可以使用如下命令：

    ```shell
    ALTER USER '用户名' IDENTIFIED WITH auth_socket;
    ```

6. 

## 配置MySQL

### 配置安全信息

1. 启动并设置**开机自启动**MySQL服务：

    ```shell
    sudo systemctl start mysqld
    sudo systemctl enable mysqld
    ```

2. 对MySQL进行安全性配置：

    ```shell
    sudo mysql_secure_installation
    ```

    1. 是否修改密码：

        ```shell
        # 是否需要修改密码
        Change the password for root ? ((Press y|Y for Yes, any other key for No) :N
        ```

    2. 是否设置密码验证组件：

        ```shell
        would you like to setup validate password component？
        Press y|Y for Yes, any other key for No) :Y
        ```

    3. 选择密码安全等级：

        ```
        There are three levels of password validation policy:
        
        LOW    Length >= 8
        MEDIUM Length >= 8, numeric, mixed case, and special characters
        STRONG Length >= 8, numeric, mixed case, special characters and dictionary                  file
        
        Please enter 0 = LOW, 1 = MEDIUM and 2 = STRONG: 1
        ```

        分别是低中高。第二个是：数字、大小写、特殊字符

    4. 根据提示信息，删除匿名用户。

        ```shell
        Remove anonymous users? (Press y|Y for Yes, any other key for No) :Y #输入Y删除MySQL默认的匿名用户。
        Success.
        ```

    5. 禁止root账号远程登录。

        ```shell
        # 输入Y禁止root远程登录。
        Disallow root login remotely? (Press y|Y for Yes, any other key for No) :Y 
        Success.
        ```

        禁止root远程，后续创建新用户进行远程登录。

    6. 删除test库以及对test库的访问权限。

        ```shell
        Remove test database and access to it? (Press y|Y for Yes, any other key for No) :Y #输入Y删除test库以及对test库的访问权限。
         - Dropping test database...
        Success.
        
         - Removing privileges on test database...
        Success.
        ```

    7. 重新加载授权表。

        ```shell
        Reload privilege tables now? (Press y|Y for Yes, any other key for No) :Y #输入Y重新加载授权表。
        Success.
        All done!
        ```

### 远程访问MySQL

1. 输入root用户的密码登录MySQL。

    ```shell
    sudo mysql -uroot -p
    ```

2. 创建远程登录MySQL的账号，并允许远程主机使用该账号访问MySQL：

    注意：密码需要符合上面设置的密码安全等级的规范。

    ```shell
    create user '用户名' identified by '密码'; 
    #为用户授权数据库所有权限。
    grant all privileges on *.* to '用户名'; 
    #刷新权限。
    flush privileges; 
    ```

3. 执行以下命令，退出数据库。

    ```shell
    exit
    ```

4. 修改mysql配置文件，允许远程连接：

    ```shell
    vi /etc/mysql/mysql.conf.d/mysqld.conf
    bind-address = 0.0.0.0  # 0.0.0.0表示允许所有ip访问  默认是127.0.0.1 只允许本地访问
    port = 3306
    ```

    注意：这里的路径是使用apt安装的MySQL的路径，如果使用其他方法，需要更换对应的配置文件的路径。

5. 使用设置的账号远程登录MySQL。（如navicat）

## 项目部署

通过`maven`生成jar包，然后将jar包上传到服务器并启动。

### jar包生成

1. 将项目的父工程设置为 `spring-boot`。

```xml
 <parent>
     <groupId>org.springframework.boot</groupId>
     <artifactId>spring-boot-starter-parent</artifactId>
     <version>2.7.9</version>
     <relativePath/>
</parent>
```

注意：这里规定了父工程的版本，子项目也需要使用相同版本的spring-boot（2）。

2. 添加maven插件

    如果项目的pom配置文件中没有的话，需要添加maven插件。

    ```
        <build>
            <plugins>
                <plugin>
                    <groupId>org.springframework.boot</groupId>
                    <artifactId>spring-boot-maven-plugin</artifactId>
                    <version>2.7.9</version>
                </plugin>
            </plugins>
        </build>
    ```

    有了maven插件后，spring boot可以通过spring-boot-maven-plugin将所有应用启动运行所需要的jar都包含进来，从逻辑上将具备了独立运行的条件。

2. 打包生成jar包并上传到服务器。

### jar启动

1. 查看当前`Java`进程：

    ```shell
    ps -ef | grep java 
    # 示例
    root        6123    5705 15 16:54 pts/0    00:00:08 java -jar test-0.0.1-SNAPSHOT.jar
    ```

2. 根据进程号关闭对应进程：

    ```shell
    kill pid # pid是进程号，但是无法关闭后台进程、守护进程等
    kill - 9 pid # 强制关闭进程
    ```

3. 启动对应的jar包

    ```shell
    nohup  java -jar xxx.jar >> daily.log & 
    ```

    说明：`nohup`表示进程可以后台运行，程序的输出信息会输出到daily.log日志文件中，默认会自动创建该文件。

## 注意事项

这里的安装是基于centos系统，`yum`是centos中的软件包管理器，Ubuntu中的是`apt`。




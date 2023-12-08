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

1. 查看yum源中JDK版本：

    ```shell
    yum list java*
    ```

2. 使用yum安装JDK1.8.0。

    ```shell
    yum -y install java-1.8.0-openjdk*
    ```

3. 查看是否安装成功。

    ```shell
    java -version
    ```

    如果显示如下所示，则表示JDK安装成功。

    ```shell
    [root@ ~]# java -version
    openjdk version "1.8.0_342"
    OpenJDK Runtime Environment (build 1.8.0_342-b07)
    OpenJDK 64-Bit Server VM (build 25.342-b07, mixed mode)
    ```

4. 配置环境变量。

    1. 查看JDK安装的路径：

        ```shell
        find /usr/lib/jvm -name 'java-1.8.0-openjdk-1.8.0*'
        ```

        回显信息如下所示。

        ```shell
        [root@test~]# find /usr/lib/jvm -name 'java-1.8.0-openjdk-1.8.0*'
        /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.342.b07-1.el7_9.x86_64
        ```

    2. 打开配置文件。

        ```shell
        vim /etc/profile
        ```

    3. 在配置文件末尾，按 `i` 进入编辑模式。

    4. 添加以下信息。

        **说明**

        `JAVA_HOME`值为**当前JDK安装**的路径。

        ```shell
        JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.342.b07-1.el7_9.x86_64
        PATH=$PATH:$JAVA_HOME/bin
        CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
        export JAVA_HOME CLASSPATH PATH
        ```

    5. 按下`Esc`键，输入`:wq`并回车以保存并关闭文件。

    6. 运行以下命令，立即生效环境变量。

        ```shell
        source /etc/profile
        ```

    

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

## 注意事项

这里的安装是基于centos系统，`yum`是centos中的软件包管理器，Ubuntu中的是`apt`。
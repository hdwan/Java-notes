前提：

阿里云：实例安全组的入方向规则已放行22端口。

## 使用二进制文件安装

Node.js官网：https://nodejs.org/en

下载服务器系统对应的node.js的二进制安装包。

## 环境安装

1. 下载Node.js安装包：

    注意：根据服务器系统更改安装包的网址。

    ```shell
    wget https://nodejs.org/dist/v16.0.0/node-v16.0.0-linux-x64.tar.xz
    ```

2. 解压Node.js安装包：

    ```shell
    tar xvf node-v16.0.0-linux-x64.tar.xz
    ```

3. 创建node和npm的软链接：

    说明：创建软链接后，可以在任意目录下直接使用node和npm命令。

    ```shell
    ln -s /root/node-v16.0.0-linux-x64/bin/node /usr/local/bin/node
    ln -s /root/node-v16.0.0-linux-x64/bin/npm /usr/local/bin/npm
    ```

4. 依次查看node、npm版本信息。

    ```shell
    node -v
    npm -v
    ```

    

5. 安装完毕。软件默认安装在/root/node-v16.0.0-linux-x64/目录下。

## 更换安装文件

将该软件安装到其他目录下，例如：/opt/node/，可以依次运行以下命令：

1. 创建/opt/node/路径。

    ```shell
    mkdir -p /opt/node/
    ```

2. 将Node.js的所有文件移动至/opt/node/。

    ```shell
    mv /root/node-v16.0.0-linux-x64/* /opt/node/
    ```

3. 依次运行以下命令，移除源路径中node和npm的软链接。

    ```shell
    rm -f /usr/local/bin/node
    rm -f /usr/local/bin/npm
    ```

4. 依次运行以下命令，在/opt/node/中新建node和npm的软链接。

    ```shell
    ln -s /opt/node/bin/node /usr/local/bin/node
    ln -s /opt/node/bin/npm /usr/local/bin/npm
    ```

## 部署测试项目

1. 创建测试项目文件`example.js`：

    1. 返回`/root`路径。

        ```shell
        cd
        ```

    2. 创建测试项目文件example.js。

        ```shell
        touch example.js
        ```

2. 修改项目文件`example.js`：

    1. 运行以下命令打开`example.js`。

        ```shell
        vim example.js
        ```

    2. 按 `i` 键进入编辑模式，并将以下内容添加至 `example.js` 文件中。

        说明：项目占用的端口号为3000、输出的内容为Hello World。

        ```shell
        const http = require('http');
        const hostname = '0.0.0.0';
        const port = 3000;
        const server = http.createServer((req, res) => { 
            res.statusCode = 200;
            res.setHeader('Content-Type', 'text/plain');
            res.end('Hello World\n');
        }); 
        
        server.listen(port, hostname, () => { 
            console.log(`Server running at http://${hostname}:${port}/`);
        });
        ```

    3. 添加完成后，按Esc键退出编辑模式，并输入`:wq`后按Enter键，保存退出文件。

3. 运行项目并得到项目的端口号。

    ```shell
    node ~/example.js &
    ```

4. 运行以下命令，列入系统已在监听的端口信息。

    ```shell
    netstat -tpln
    ```

    返回的结果列表中包含端口3000，表明项目正常运行。

5. 在ECS实例的安全组中，添加入方向规则，放行项目中配置的端口号：

    即上面的项目端口号为3000。

6. 打开浏览器并访问`http://<ECS实例公网IP地址>:<项目端口号>`，访问成功说明安装成功。
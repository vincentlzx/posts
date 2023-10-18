---
title: Jenkins自动化部署Java项目
categories:
- [CI/CD]
tags:
- CI/CD
- Jenkins
---



## Maven项目自动构建

下面演示 jenkins 部署 RuoYi-Cloud 项目的 ruoyi-auth 子模块。

新建 Item ，选择构建一个 maven 项目。

一般有一下几个步骤：

![image-20230717160445600-9581087](https://file.liuzx.com.cn/docsify-pic/202307241123005.png)

1. General（通用配置）：可以指定项目的名称、描述等通用配置信息。

2. 源码管理：

   - 填写仓库地址，并添加仓库访问凭证，指定分支。（本次示例使用账号密码方式）

   ![image-20230717150648336](https://file.liuzx.com.cn/docsify-pic/202307241126532.png)

3. 构建触发器：可以配置触发构建的条件。例如，定时触发、代码变更触发、远程触发等。

4. 构建环境：可以配置构建过程中的环境变量、构建参数等。还可以选择是否清理工作区、指定构建 JDK 等。
   - 选中 Add timestamps to the Console Output ，在构建日志控制台输出中打印时间。

5. Pre Steps：选择 maven 版本，可选择默认或者自定义安装的 maven 版本；在目标文本框中添加 mvn 执行参数。（跳过这一步骤，直接在 Build 步骤配置 maven 好像也行）

   ![image-20230717161517033](https://file.liuzx.com.cn/docsify-pic/202307241129095.png)

6. Build：配置 maven 构建参数。

   > -pl ruoyi-auth 表示打包指定模块；
   >
   > -am 表示同时打包处理选定模块所依赖的模块；
   >
   > -Dmaven.test.skip=true 表示跳过单元测试。

   ![image-20230717162212630](https://file.liuzx.com.cn/docsify-pic/202307241134753.png)

   点击“高级”，可以自定义 maven 路径。

   ![image-20230724113417493](https://file.liuzx.com.cn/docsify-pic/202307241134518.png)

7. Post Steps：以下是装有插件 [Publish Over SSH版本1.24](https://plugins.jenkins.io/publish-over-ssh) 的配置方式。

   7.1 SSH Server：选择远程服务器，选中 Verbose output in console 表示在 jenkins 控制台输出自定义 shell 执行的日志。

   > 参考：https://blog.csdn.net/qq_32077121/article/details/121861523

   ![image-20230717162658530](https://file.liuzx.com.cn/docsify-pic/202307241135419.png)

   7.2 Transfers：传输文件。1. Source files 指定 RuoYi-Cloud 项目中的 jar 包相对路径；2. Remove prefix 表示传输文件过程中不会在远程服务器上创建的文件路径；3. Remote directory 指定远程服务器的相对路径（基础路径我在 publish over ssh 插件配置中指定为 /root）；4. Exec command 为远程服务器上执行的 shell 内容。

   ![image-20230717165305298](https://file.liuzx.com.cn/docsify-pic/202307241135337.png)

   shell 内容：

   ```shell
   port=19200
   project=ruoyi-auth
   jar=ruoyi-auth.jar
   
   # 查找占用指定端口号的Java进程PID
   pid=$(lsof -i :$port -t)
   
   if [ -z "$pid" ]; then
     echo "没有占用端口号 $port 的Java进程"
   else
     echo "正在终止占用端口号 $port 的Java进程，PID: $pid"
     kill -9 $pid
     echo "Java进程已终止"
   fi
   
   # 拷贝jar包在远程服务器的路径
   cd /root/projects/$project
   nohup java -Xms128m -Xmx256m -Xmn90m -jar $jar --server.port=$port > nohup.out 2>&1 & echo "进程ID：$!"
   ```

   7.3 Exec timeout (ms)：Exec command 超时时间，0 表示禁用；在 jenkins 构建后执行 nohup 脚本有不能退出的问题，选中 Exec in pty 试试。

   ![image-20230717171125383](https://file.liuzx.com.cn/docsify-pic/202307241135931.png)

   > nohup java -Xms128m -Xmx256m -Xmn90m -jar $jar --server.port=$port > nohup.out 2>&1 &
   >
   > 执行该命令没有出现过不能自动退出构建的情况，而直接执行 nohup ... & 好像可能会发生不能退出构建的情况。

执行到这一步可以直接保存，构建项目了。
---
title: systemctl解决进程自动挂掉的问题
categories:
- [CentOS]
tags:
- systemctl
---



## 场景

docsify 部署的文档服务进程运行一段时间后总是自动挂掉。

## 解决方案

使用 systemctl 管理 docsify 服务，设置服务在崩溃或退出后自动重启。并且后续可以查看相关的系统日志和服务状态信息。

## systemctl运行服务的步骤

1. 创建 Systemd 服务单元文件：在 `/etc/systemd/system/` 目录下创建一个以 `.service` 结尾的文件，比如 `docsify.service`。

   ```shell
   sudo vim /etc/systemd/system/docsify.service
   ```

2. 添加以下内容：

   ```shell
   [Unit]
   # 服务的描述。
   Description=Your Java Application
   After=network.target
   
   [Service]
   # 服务运行的用户名。
   User=root
   # 启动命令
   ExecStart=/usr/local/bin/docsify serve /root/docs
   # 设置服务在崩溃或退出后自动重启。
   Restart=always
   
   [Install]
   WantedBy=multi-user.target
   ```

3. 重新加载 Systemd 配置：

   ```
   sudo systemctl daemon-reload
   ```

4. 启动服务：

   ```
   sudo systemctl start docsify
   ```

    `docsify` 就是刚创建的服务单元文件 docsify.service 中定义的服务名称。

5. 验证服务是否成功启动：

   ```
   sudo systemctl status docsify
   ```

   如果应用正在运行，你应该会看到输出中包含 "active (running)" 的标识。这将显示服务的当前状态，包括是否正在运行、最后一次的活动时间以及是否存在任何错误或警告信息。

6. 如果希望服务在系统启动时自动启动，可以执行以下命令将其添加到启动项中：

   ```
   sudo systemctl enable docsify
   ```

> 1. 服务名称由文件名决定
>
>    在 Systemd 中，服务的名称是由服务单元文件的文件名决定的，而不是由其中的 `Description` 字段确定的。文件名应该以 `.service` 结尾，并且通常会反映服务的名称。
>
>    例如，如果你有一个名为 `your-service.service` 的服务单元文件，那么服务的名称就是 `your-service`。这是因为 Systemd 会根据服务单元文件的文件名来识别和操作服务。
>
>    服务单元文件中的 `Description` 字段是对服务的描述，它用于提供关于服务的简要说明，但它不会直接影响服务的名称或标识。
>
>    所以，服务的名称是由服务单元文件的文件名决定的，而不是由其中的 `Description` 字段确定的。确保在创建服务单元文件时，为文件名选择一个合适的名称，以便能够清楚地标识和操作服务。
>
> 2. 当在 Systemd 服务单元文件中指定 `Executable` 路径时，需要确保该路径都是绝对路径，而不是相对路径。如果你遇到 "Executable path is not absolute" 的错误消息，说明指定的路径不是绝对路径。
>
>    如上代码，ExecStart 需要指定正确的可执行文件绝对路径 /usr/local/bin/docsify 和应用的文件绝对路径 /root/docs 。

## 查看服务日志

要查看 `systemctl` 服务的日志，可以使用 `journalctl` 命令。`journalctl` 是 CentOS 系统中用于查看系统日志的工具，它可以显示各个服务的日志记录。

1. 查看指定服务的日志：

   ```
   journalctl -u docsify.service
   ```

2. 查看最近的服务日志：

   ```
   journalctl -u docsify.service -n 100
   ```

   这将显示最近的 100 条关于指定服务的日志记录。

3. 实时查看服务日志：

   ```
   journalctl -u docsify.service -f
   ```

   使用 `-f` 参数，`journalctl` 将实时显示指定服务的日志记录，并不断更新。

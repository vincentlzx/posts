---
title: Shell脚本的解释器
categories:
- [CentOS, Shell]
tags:
- Shell
---



## sh script.sh和./script.sh的区别

- `sh script.sh` 是通过指定 `sh` 解释器来运行脚本，忽略脚本中的 shebang 行。（例如 `#!/usr/bin/expect -f`）
- `./script.sh` 是通过脚本中的 shebang 行指定的解释器来运行脚本，需要脚本具有可执行权限。

## 指定解释器执行shell脚本的方式

假设希望指定 expect 解释器执行脚本，方式如下：

1. 在 shell 脚本中的 shebang 行（第一行）指定解释器。

   ```shell
   #!/usr/bin/expect -f
   ```

   > 必须使用 ./script.sh 的方式运行脚本，避免忽略指定的解释器。

2. 使用 expect 命令来执行脚本，这种方式是直接指定 expect 解释器来执行脚本，忽略脚本中指定的解释器。

   ```shell
   expect script.sh
   ```

   
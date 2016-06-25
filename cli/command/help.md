## Help 命令

### 作用

用来显示所有可用的命令或某个命令的帮助说明。

### 用法

```
$ {YII_PATH}/yiic help [command name]
```
如果不给命令名称，则显示所有可用命令。

### 类别名

```
system.console.CHelpCommand
```

### run

1. 不给命令名称或无效命令名称，则显示所有可用命令；
2. 提供有效命令名称则显示命令的帮助说明。

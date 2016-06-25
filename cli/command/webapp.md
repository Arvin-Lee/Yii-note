## Webapp 命令

### 作用

在指定目录下快速生成一套常规Yii应用的目录结构。

### 用法

```
$ {YII_PATH}/yiic webapp webroot/App [git|hg]
```
执行命令后，将会在文档根目录下的App目录生成一套常规Yii应用的目录结构。支持指定vcs工具，不建议使用，建议使用特定vcs客户端。

更多使用细节，请查看帮助方法：

```
$ {YII_PATH}/yiic help webapp
```

### 类别名

```
system.cli.commands.WebAppCommand
```

### run

1. 遍历```system.cli.views.webapp```目录，获取源/目标Mapping表；
2. 为特定文件，如：index.php，加上callback，用于改写文件中yii.php等文件相对路径；
3. 在指定目录生成应用的目录结构；
4. 调整特定目录权限。

### 说明

如果想要定制应用目录结构，维护```system.cli.views.webapp```下的目录即可。

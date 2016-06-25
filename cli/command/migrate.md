## Migrate 命令

### 作用

像用Git来控制源码版本一样，Migrate用来追踪数据库结构的变化（migration），如：创建新表，为表字段加索引等等。
目的是为了保证源代码和数据库的变更同步，便于协同开发。

### 用法

```
$ {YII_PATH}/yiic migrate [action] [parameter]
```

可用```action```列表：

* **up** - 提交migration
* **down** - 回滚migration
* **to** - 提交或回滚migration
* **mark** - 提交或回滚migration至特定版本
* **history** - 查看已提交migration
* **new** - 查看未提交migration
* **redo** - 撤销提交或回滚操作

可用```parameter```列表：

* **migrationPath** - migration目录，必须为别名，默认为application.migrations
* **migrationTable** - migration表名，默认为tbl_migration
* **connectionID** - 使用的数据库组件ID
* **templateFile** - migration模板文件路径，必须为别名
* **defaultAction** - 默认的action，默认为up
* **interactive** - 是否已命令行交互模式执行，默认为true。当以后台进程执行时，设为false

**注意：** 除了上述带名称的可选项，还可以传入一些不带名称的参数。

更多使用细节，请查看帮助方法：

```
$ {YII_PATH}/yiic help migrate
```

### 类别名

```
system.cli.commands.MigrateCommand
```

### run

1. 处理入参，将动作（方法名），可选项（类属性）和不带名字的参数（方法实参）分类；
2. 当传入了可选项参数时，给类属性赋值；
3. 通过反射调用不同的action方法。

### 说明

1. 当组内成员共用同一台开发库时，migrate的作用会大打折扣，但还是可以在本地使用。
2. 不同于其它命令，migrate的run方法在抽象父类CConsoleCommand中实现。其作为一个路由方法，调用不同的action方法完成不同的指令。

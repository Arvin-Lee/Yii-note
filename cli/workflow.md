## YIIC工作流

### 脚本入口

yiic有两个执行脚本入口，一个**应用级别**，一个**框架级别**。区别在于应用级别的入口可以定制一些应用级别的配置数据。

1. 应用入口

应用目录下的```yiic```或```yiic.php```。其中， ```yiic```为Unix/Linux环境脚本，```yiic.php```为Windows环境脚本。可以在```application.config.console```文件中定制配置数据。

2，框架入口

框架目录下的```yiic```或```yiic.php```。其中， ```yiic```为Unix/Linux环境脚本，```yiic.php```为Windows环境脚本。

### 工作流

> 以应用入口举例

1. 加载框架yiic脚本并定义配置文件路径；
2. 加载框架引导文件yii.php，并以配置数据创建一个console应用实例；
3. 创建CommandRunner并将```application.commands```目录下的command类脚本路径加载进内存；
4. 将框架内建的command类脚本(如：webapp，shell等)路径加载进内存；
5. 如果本地设置了环境变量```YII_CONSOLE_COMMANDS```，将环境变量指向的目录下的command类脚本路径加载进内存；
6. 终端运行yiic命令，执行CommandRunner，创建对应command实例，带入参数执行命令。

### 说明

Yii提供了环境变量```YII_CONSOLE_COMMANDS```用来加载独立于应用的command脚本。当一台机器上部署了多个Yii应用，每个应用都需要某个yiic命令脚本时，就可以将该脚本放置在该环境变量指向的目录下统一维护。

## Yii中调试SQL的方法

在程序开发中，为了保证代码的可读性和维护性，往往采用一种抽象的方式去操作数据库，而不是直接写SQL。但很多时候，我们想要知道最终执行的SQL到底长怎样。事实上，通过简单的配置，Yii就能帮我们达到目标。下面介绍配置方法：

### 步骤

#### 1. 开启DEBUG

因为需要使用到```Yii::trace```方法记录的Log，我们需要开启DEBUG。开启方法非常简单，在应用入口加上如下代码即可：

```
defined('YII_DEBUG') or define('YII_DEBUG',true);
```

#### 2. 开启Web日志功能

因为我们需要非常方便的就能查看到调试信息，最好在页面上就能看到，所以我们开启Web日志功能。开启的方法同样非常简单，在应用的配置文件main.php中的log组件里加上如下路由规则即可：

```
array(
	'class'=>'CWebLogRoute',
    'showInFireBug'=>true,
),
```

事实上，```showInFireBug```不是必须的。当不设置此属性或设为false时，调试信息将打印在页面底部；当设置为true时，调试信息将打印在浏览器控制台中。为了保证页面的完整性，建议将其设置为true。

#### 3. 开启参数记录功能

当完成前面两步时，我们就已经可以调试SQL。但当存在查询绑定时，具体绑定在参数上的值并不知道。怎样才能知道绑定的值呢？只需在数据库组件上加一行配置即可：

```
'enableParamLogging' => true,
```

这样就可以知道完整的SQL了。

### 示例

#### 1. DAO

**代码**

```php
<?php
// ......

public function actionDao()
{
    $sql = 'update user set email=:email where id = 11';
    $command = Yii::app()->db->createCommand($sql);
    $command->bindValue(':email', 'zhangsan2@tuniu.com', PDO::PARAM_STR);

    $command->execute();
}
```

** 调试信息 **

![DAO](http://m.tuniucdn.com/fb2/t1/G2/M00/16/A3/Cii-TleY1VOIHXoJAAH1DvrZy2AAAAhQgLFBWgAAfUm004.png)

#### 2. Query Builder

** 代码 **

```php
<?php
// ......

public function actionQb()
{
	Yii::app()->db->createCommand()
    	->select()
        ->from('user')
        ->where(array('and', 'sex=0', array('like', 'telphone', '131%'), array('like', 'email', '%@php.net')))
        ->queryAll();
}
```
** 调试信息 **

![QB](http://m.tuniucdn.com/fb2/t1/G2/M00/16/A3/Cii-TFeY1YKIDrRAAAIm0e9WJ_8AAAhQgMBXlUAAibp061.png)

#### 3. Active Record

** 代码 **

```php
<?php
// ......

public function actionAr()
{
    User::model()->exists('id > :id', array(':id'=>10));
}
```

** 调试信息 **

![AR](http://m.tuniucdn.com/fb2/t1/G2/M00/16/A3/Cii-T1eY1amIVtVfAAK6qMwdcIgAAAhQgMHynAAArrA245.png)

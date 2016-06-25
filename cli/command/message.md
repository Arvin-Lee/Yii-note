## Message 命令

### 作用

从指定源文件中抽取消息，把它们按照指定语言翻译组织在PHP数组中，并存储在指定目录下。

### 用法

```
$ {YII_PATH}/yiic message <config-file>
```

```config-file``` 必选项，消息配置文件，格式可参照示例：```framework/messages/config.php```。

```php
<?php
/**
 * This is the configuration for generating message translations
 * for the Yii framework. It is used by the 'yiic message' command.
 */
return array(
	'sourcePath'=>dirname(__FILE__).DIRECTORY_SEPARATOR.'..',
	'messagePath'=>dirname(__FILE__).DIRECTORY_SEPARATOR.'..'.DIRECTORY_SEPARATOR.'messages',
	'languages'=>array('fi','zh_cn','zh_tw','ca','de','el','es','sv','he','nl','pt','pt_br','ru','it','fr','ja','pl','hu','ro','id','vi','bg','lv','sk','uk','ko_kr','kk','cs','da'),
	'fileTypes'=>array('php'),
	'overwrite'=>true,
	'exclude'=>array(
		'.svn',
		'.gitignore',
		'yiilite.php',
		'yiit.php',
		'/i18n/data',
		'/messages',
		'/vendors',
		'/web/js',
	),
);
```

可用的配置项列表：

* **sourcePath**: 源文件目录，在此目录下搜索抽取消息。
* **messagePath**: 消息目录，存放各语言目录。
* **languages**: 语言列表，生成相应语言目录。
* **fileTypes**: 文件类型列表，规定搜索哪些文件类型。为空则搜索所有文件类型。
* **exclude**: 需排除的文件列表。
* **translator**: 翻译消息的方法名，默认为```Yii::t```。作为查找消息的标记。
* **overwrite**: 如果文件存在，是否覆盖。默认不覆盖，创建.merged文件。
* **removeOld**: 为true时，移除已翻译消息文件中被删除的消息（物理删除）；为false时，不移除消息，而是在消息两侧加上@@，表示不再维护的消息（逻辑删除）。默认为false。
* **sort**: 合并消息文件时是否对消息数组按照键名排序。
* **fileHeader**: 是否需要文件头注释内容，默认需要。

更多使用细节，请查看帮助方法：

```
$ {YII_PATH}/yiic help message
```


### 类别名

```
system.cli.commands.MessageCommand
```

### run

1. 入参检测和初始化可选参数值；
2. 递归查找所有有效源文件列表```CFileHelper::findFiles```；
3. 根据translator从源文件中抽取消息，消息以category(```Yii::t```方法第一个参数)归档，然后递归合并；
4. 根据languages生成不同语言的消息文件。

### 说明

使用该命令生成的消息文件都没有翻译，需要人工去翻译消息文件中的消息。然后在使用```Yii::t```方法时，就能通过当前设定的语言到对应的语言目录里找到相应的已翻译过的消息，实现多语言版本支持。

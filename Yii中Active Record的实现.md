## Yii中Active Record的实现

![AR](http://m.tuniucdn.com/fb2/t1/G2/M00/13/1D/Cii-TleUdT6IXDeWAACmlrDZUv4AAAbUgLGcI4AAKau126.jpg)

### 引言

Active Record（AR）是一种流行的对象关系映射（ORM）技术。 通过使用AR，程序员可以不用再去编写Web开发中常见的CRUD操作，而是通过一种更加面向对象的方式去操作数据库。这大大的提高了程序的可读性与可维护性。

### 实现

结合具体的应用场景来聊实现，或许能让我们理解的更深入一些。假设有如下的一个简单场景：

> 通过一个文章ID，查询文章详情，如：文章名称，作者，摘要，发布时间等等。

文章表tbl_post结构如下：

![Schema](http://m.tuniucdn.com/fb2/t1/G2/M00/13/1A/Cii-TFeUce2IbK8iAAJhO3ll524AAAbRAFPFGsAAmFT839.png)

按照常规的Yii DAO的写法，大概是这样：

```php
<?php
// ......

$query = "select * from tbl_post where id=:id limit 1";

$command = Yii::app()->db->createCommand($query);
$command->bindParam(':id', 1);
$post = $command->queryRow();

```

当存在```id=1```的记录时，返回一维数组；否则返回false。类似这样的代码在应用中随处可见，程序的维护性不是太好。而按照Yii AR的写法就简洁了许多，也更加的清晰：

```php
<?php
// ......

$post = Post::model()->findByPk(1);
```

当存在```id=1```的记录时，返回```Post```模型对象，即一个record对象；否则返回NULL。从两者写法对比上，我们就能看出AR是一种更加OO的方式。

要使用AR，需为每张表创建一个模型类，该模型类继承抽象的CActiveRecord类，并且需创建静态model方法用以获取该模型类对象。

```php
<?php

class Post extends CActiveRecord
{
	/**
	 * Returns the static model of the specified AR class.
	 * @return static the static model class
	 */
	public static function model($className=__CLASS__)
	{
		return parent::model($className);
	}

	/**
	 * @return string the associated database table name
	 */
	public function tableName()
	{
		return 'tbl_post';
	}

```

下面按照代码执行的时序，逐一讲解这其中的逻辑。

#### 1. 初始化

当执行```Post::model()```时，会实例化Post模型类。然后，会将定义的模型行为列表载入Post对象。需要定义模型的行为，只需在模型类中定义```behaviors```方法即可，具体定义方式请参照官方文档声明。

接着，访问find系列方法```findByPk```：

```php
<?php
// ......

public function findByPk($pk,$condition='',$params=array())
{
	$prefix=$this->getTableAlias(true).'.';
	$criteria=$this->getCommandBuilder()->createPkCriteria(
		$this->getTableSchema(),
		$pk,
		$condition,
		$params,
		$prefix
	);
	return $this->query($criteria);
}
```

#### 2. 获取表别名

Yii通过```getTableAlias```方法获取表别名。模型类可以定义一个默认范围，它将应用于所有 (包括相关的那些) 关于此模型的查询。例如：SQL中的select, condition, limit, order, alias等部分都可以通过命名范围去定义。这些属性将在```CDbCriteria```类被实例化时载入。想要定义默认范围，只需在模型类在定义```defaultScope```方法即可，如：

```php
<?php
// ......

public function defaultScope()
{
	return array(
		'alias'=>'p',
	);
}
```

此时，表别名就是```p```。否则，就是默认的表别名```t```。

#### 3. 获取表模式

Yii通过```getTableSchema```方法获取表模式。每一种DBMS都对应一种模式，Yii实现了统一的方法用来获取不同DBMS的模式，具体使用哪种模式完全由配置决定。获取表模式是通过```CActiveRecordMetaData```类实现的，该类和抽象的```CActiveRecord```类同在```system.db.ar.CActiveRecord```对应文件中。我们看下```CActiveRecordMetaData```类的构造方法，
其中省略了部分代码：

```php
<?php
// ......

public function __construct($model)
{
	$this->_modelClassName=get_class($model);

	$tableName=$model->tableName();
    // 根据表名，获取表模式
	if(($table=$model->getDbConnection()->getSchema()->getTable($tableName))===null)
		// 省略部分代码。抛出异常。

    // 省略部分代码。当自定义主键时，优先使用自定义主键。在模型类中定义primaryKey方法即可自定义主键。

    // 将表模式信息存赋值给tableSchema属性
	$this->tableSchema=$table;
	$this->columns=$table->columns;

    // 将表字段名和默认值，以k/v方式赋值给attributeDefaults属性
	foreach($table->columns as $name=>$column)
	{
		if(!$column->isPrimaryKey && $column->defaultValue!==null)
			$this->attributeDefaults[$name]=$column->defaultValue;
	}

    // 载入模型类与其他模型类的关联关系，在模型类中定义relations方法即可定义关联关系。
    // 目前支持四种关联关系的定义，具体定义方式请参照官方文档声明。
	// 通过定义此方法，实现关联AR。
	foreach($model->relations() as $name=>$config)
	{
		$this->addRelation($name,$config);
	}
}
```

重点是 **$table=$model->getDbConnection()->getSchema()->getTable($tableName)** 这行代码。
其中，```getSchema```方法会通过配置文件中DB组件配置的 **connectionString** 内容，获取数据库模式。
如：**'connectionString' => 'mysql:host=localhost;dbname=blog'** 这样一段配置，其对应的
DB驱动就是mysql，对应的数据库模式类就是```system.db.schema.mysql.CMysqlSchema```。

获取到数据库模式之后，接着获取指定表的元数据（metadata），也就是表的模式。

```php
<?php
// ......

public function getTable($name,$refresh=false)
{
	if($refresh===false && isset($this->_tables[$name]))
		return $this->_tables[$name];
	else
	{
		// 省略部分代码。针对忽略表名前缀的处理

		// 省略部分代码。临时禁掉查询缓存

		if(!isset($this->_cacheExclude[$name])
            && ($duration=$this->_connection->schemaCachingDuration)>0
            && $this->_connection->schemaCacheID!==false
            && ($cache=Yii::app()->getComponent($this->_connection->schemaCacheID))!==null)
		{
			$key='yii:dbschema'.$this->_connection->connectionString.':'.$this->_connection->username.':'.$name;
			$table=$cache->get($key);
			if($refresh===true || $table===false)
			{
				$table=$this->loadTable($realName);
				if($table!==null)
					$cache->set($key,$table,$duration);
			}
			$this->_tables[$name]=$table;
		}
		else
			$this->_tables[$name]=$table=$this->loadTable($realName);

		// 省略部分代码。重新开启查询缓存

		return $table;
	}
}
```

当查询的表名不在缓存排除列表，且设置了有效```schemaCachingDuration```属性，且```schemaCacheID```属性
不为false，且能够加载```schemaCacheID```属性指定缓存组件时，将首先从缓存中获取表的元数据。如果缓存中没找到，则访问loadTable方法获取表的元数据，然后写入缓存。否则，直接访问loadTable方法获取表的元数据。

loadTable方法在```system.db.schema.CDbSchema```抽象类中被声明，该抽象类是所有数据库模式类公共的父类，在所有数据库
模式类中必需要实现loadTable方法。因此，我们在```system.db.schema.mysql.CMysqlSchema```下找到loadTable方法。

```php
<?php
// ......

protected function loadTable($name)
{
	$table=new CMysqlTableSchema;
    // 处理各种各样的表名。比如有的表名会带上库名，有的则不会。
	$this->resolveTableNames($table,$name);

    // 收集表字段元数据
	if($this->findColumns($table))
	{
        // 收集指定表的外键信息
		$this->findConstraints($table);
		return $table;
	}
	else
		return null;
}
```

其中```findColumns```方法收集表字段元数据，执行如下SQL：
```
mysql> SHOW FULL COLUMNS FROM tbl_post;
```
返回的结果集将以k/v的形式存储在```CDbTableSchema```类的```columns```属性中。k为字段名，v为```CMysqlColumnSchema```类的实例，其储存了字段的各种元数据，如：是否允许为空，是否主键，是否自增，注释等。

另一个```findConstraints```方法收集指定表的外键信息，执行如下SQL：

```
mysql> SHOW CREATE TABLE tbl_post;
```
通过正则表达式匹配结果中的关键词，如果设置了外键，则将相应字段指向的```CMysqlColumnSchema```类的```isForeignKey```属性设置true。

最后返回包含有表元数据的变量。程序回到```CActiveRecordMetaData```类的构造函数中，将包含元数据的变量赋值给```tableSchema```属性和将字段元数据赋值给```columns```属性。最后向上返回表元数据，回到```findByPk```方法。

#### 4. 创建CDbCriteria实例

在Yii中，可以通过创建```CDbCriteria```类实例将SQL语句的各个部分储存在其中，每一个实例最终将生成一条SQL语句。这样我们维护的就不是一条条SQL语句，而是SQL语句中各个结构化的部分。这些部分，有的支持配置，有的使用特定的AR方法配合传参维护。最终，将这些部分拼接就是一条完整的SQL语句。

#### 5. 查询

```php
<?php
// ......

protected function query($criteria,$all=false)
{
	$this->beforeFind();
	$this->applyScopes($criteria);

	if(empty($criteria->with))
	{
		if(!$all)
			$criteria->limit=1;
		$command=$this->getCommandBuilder()->createFindCommand($this->getTableSchema(),$criteria);
		return $all ? $this->populateRecords($command->queryAll(), true, $criteria->index) : $this->populateRecord($command->queryRow());
	}
	else
	{
		$finder=$this->getActiveFinder($criteria->with);
		return $finder->query($criteria,$all);
	}
}
```

对```findByPk```场景来说，重点是 **$this->populateRecord($command->queryRow())** 这行代码。
其中```$command->queryRow()```我们很熟悉，就是获取结果集的第一行记录。我们重点看看```populateRecord```方法。

```php
<?php
// ......

public function populateRecord($attributes,$callAfterFind=true)
{
	if($attributes!==false)
	{
		$record=$this->instantiate($attributes);
		$record->setScenario('update');
		$record->init();
		$md=$record->getMetaData();
		foreach($attributes as $name=>$value)
		{
			if(property_exists($record,$name))
				$record->$name=$value;
			elseif(isset($md->columns[$name]))
				$record->_attributes[$name]=$value;
		}
		$record->_pk=$record->getPrimaryKey();
		$record->attachBehaviors($record->behaviors());
		if($callAfterFind)
			$record->afterFind();
		return $record;
	}
	else
		return null;
}
```

该方法通过queryRow返回的记录创造一个活动记录。如果模型类中定义了和字段对应的属性，则直接赋值；否则，将记录以k/v的形式
赋值给私有属性```_attributes```。当我们访问这些未定义的属性时，```CActiveRecord```类中定义的魔术getter方法会尝试从属性```_attributes```中获取数据。

```php
<?php
// ......

public function __get($name)
{
	if(isset($this->_attributes[$name]))
		return $this->_attributes[$name];
	elseif(isset($this->getMetaData()->columns[$name]))
		return null;
	elseif(isset($this->_related[$name]))
		return $this->_related[$name];
	elseif(isset($this->getMetaData()->relations[$name]))
		return $this->getRelated($name);
	else
		return parent::__get($name);
}
```

至此，就完成了关系数据到对象数据映射的全部工作，即一次ORM。

### 说明

本文旨在通过```findByPk```这种最简单的AR应用场景，向大家介绍Yii实现AR的主流程。其中有很多的知识点都只是一句带过或压根没提，
感兴趣的同学可以自己去阅读源码。比如：使用AR处理事务等。此外，除了通过静态方法```model```实例化模型类外，也可以直接通过```new```关键字实例化，这两者存在一些区别。还有各种各样的AR方法，想要了解细节，都需自己去阅读源码。总之，ORM是一种比较复杂但值得花时间去学习的一种技术。

### 总结

对有追求的程序员来说，通过AR去操作MySQL确实是一件很爽的事情，但AR也并不是银弹。当业务逻辑较简单（CRUD），或表中的每行记录比较容易抽象成对象时，使用AR就非常好。但是当业务逻辑较复杂，需多表联合查询时，可能就需要使用Relational AR处理了，当然前提是表结构具有设计良好的主键-外键约束及正确的关联关系定义，程序会更加复杂一些。所以，当你真正的清楚了它的实现细节时，你就能更加恰如其分的使用它，这是最好的。

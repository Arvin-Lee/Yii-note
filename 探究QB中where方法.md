## 探究Query Builder中where方法

### 引言

Yii的Query Builder提供了一种面向对象的方式去写SQL语句。它允许开发者使用方法和属性去指定SQL中各个不同部分，然后将其拼接成一条有效的SQL语句供数据访问对象（DAO）执行。在所有这些结构化的部分中，where方法是难点也是重点。

### 源码分析

先贴出源码：

```php
<?php
// ......
public function where($conditions, $params = array())
{
    $this->_query['where'] = $this->processConditions($conditions);
    foreach ($params as $name => $value)
        $this->params[$name] = $value;
    return $this;
}
```

该方法所在类别名：```system.db.CDbCommand```。方法接受两个参数，第一个为```$conditions```，mixed数据类型，通过该入参，生成SQL的where部分；第二个为```$params```，array数据类型，作为查询绑定的参数。

代码较简单，依次做下面三件事儿：

1. 处理```$conditions```，将结果按键值对的方式，键为where，存储在私有属性_query中；
2. 遍历```$params```，将查询绑定参数按键值对的方式存储在公有属性params中；
3. 返回```CDbCommand```实体对象。

下面我们重点看看处理条件方法```processConditions```的实现：

```php
<?php
// ......
private function processConditions($conditions)
{
    if (!is_array($conditions))
        return $conditions;
    else if ($conditions === array())
        return '';
    $n = count($conditions);
    $operator = strtoupper($conditions[0]);
    if ($operator === 'OR' || $operator === 'AND') {
        $parts = array();
        for ($i = 1; $i < $n; ++$i) {
            $condition = $this->processConditions($conditions[$i]);
            if ($condition !== '')
                $parts[] = '(' . $condition . ')';
        }
        return $parts === array() ? '' : implode(' ' . $operator . ' ', $parts);
    }

    if (!isset($conditions[1], $conditions[2]))
        return '';

    $column = $conditions[1];
    if (strpos($column, '(') === false)
        $column = $this->_connection->quoteColumnName($column);

    $values = $conditions[2];
    if (!is_array($values))
        $values = array($values);

    if ($operator === 'IN' || $operator === 'NOT IN') {
        if ($values === array())
            return $operator === 'IN' ? '0=1' : '';
        foreach ($values as $i => $value) {
            if (is_string($value))
                $values[$i] = $this->_connection->quoteValue($value);
            else
                $values[$i] = (string) $value;
        }
        return $column . ' ' . $operator . ' (' . implode(', ', $values) . ')';
    }

    if ($operator === 'LIKE' || $operator === 'NOT LIKE' || $operator === 'OR LIKE' || $operator === 'OR NOT LIKE') {
        if ($values === array())
            return $operator === 'LIKE' || $operator === 'OR LIKE' ? '0=1' : '';

        if ($operator === 'LIKE' || $operator === 'NOT LIKE')
            $andor = ' AND ';
        else {
            $andor = ' OR ';
            $operator = $operator === 'OR LIKE' ? 'LIKE' : 'NOT LIKE';
        }
        $expressions = array();
        foreach ($values as $value)
            $expressions[] = $column . ' ' . $operator . ' ' .$this->_connection->quoteValue($value);
        return implode($andor, $expressions);
    }

    throw new CDbException(Yii::t('yii', 'Unknown operator "{operator}".', array('{operator}' => $operator)));
}
```

该方法会以字符串的形式返回SQL的where部分，否则抛出未知operator的异常。我们走读下代码：

1. 如果```$conditions```为字符串，则直接返回；如果```$conditions```为空数组，则返回空字符串；
2. 当操作符（operator）为```and```或```or```且设置了操作数（operand）时，以递归的方式构建where语句；
3. 没有设置操作数时，返回空字符串；
4. 第一个操作数，此时为字段名，没有用括号包围时，将其引用化（即：``` `column` ```）；第二个操作数不为数组时，将其转换为数组；
5. 当操作符为```in```或```not in```时，如果第二个操作数```$values```为空数组且操作符为```in```时，返回```0=1```；为```not in```时，返回空字符串。遍历```$values```，将字符类型用引号包围，将非字符类型字符化；最后返回由操作符，字段名和查询数值拼接的字符串；
6. 当操作符为```like```，```not like```，```or like```或```or not like```时，如果第二个操作数```$values```为空数组且操作符为```like```或```or like```时，返回```0=1```；否则返回空字符串。操作符为```like```，```not like```时，级联符为```and```，否则为```or```。遍历```$values```，将字段名，操作符和查询表达式拼接而成的字符串存储在表达式数组中；最后返回由级联符拼接表达式数组而成的字符串。

### 安全性

Yii本身安全性不成问题，但如果使用不规范，还是可能会出现安全漏洞。比如这样一个Action：

```php
<?php
// ......
public function actionQb()
{
    $title = Yii::app()->request->getQuery('title');

    Yii::app()->db->createCommand()
        ->select()
        ->from('tbl_post')
        ->where(array('and', "title='$title'"))
        ->queryRow();
}
```

将外部参数title和字段直接拼接传给了where，形成了一个SQL注入漏洞。

![SQLI](http://m.tuniucdn.com/fb2/t1/G1/M00/3B/EA/Cii9EFd13c2ILgOUAAOfXoS81XIAAG2ZgGhQIcAA592928.png)

正确的做法应该是先把SQL的结构传给MySQL，再赋值。也就是先预编译，再绑定参数。

```php
<?php
// ......
public function actionQb()
{
    $title = Yii::app()->request->getQuery('title');

    Yii::app()->db->createCommand()
        ->select()
        ->from('tbl_post')
        ->where(array('and', "title=:title"), array(':title'=>$title))
        ->queryRow();
}
```

### Demo

以下面这张用户表举例：

```
+----+--------+-----+-------------+---------------------+
| id | name   | sex | telphone    | email               |
+----+--------+-----+-------------+---------------------+
|  1 | 张三   |   1 | 13951770000 | zhangsan@tuniu.com  |
|  2 | 李四   |   1 | 13851770000 | lisi@gmail.com      |
|  3 | 王五   |   1 | 13751770000 | wangwu@gmail.com    |
|  4 | 赵六   |   0 | 13651770000 | zhaoliu@126.com     |
|  5 | 孙七   |   1 | 13551770000 | sunqi@163.com       |
|  6 | 周八   |   0 | 13451770000 | zhouba@tuniu.com    |
|  7 | 吴九   |   1 | 13351770000 | wujiu@tuniu.com     |
|  8 | 郑十   |   0 | 13251770000 | zhengshi@tuniu.com  |
|  9 | 刘一   |   0 | 13151770000 | liuyi@php.net       |
| 10 | 陈二   |   0 | 13051770000 | chener@php.net      |
| 11 | 张三   |   0 | 13951770001 | zhangsan2@tuniu.com |
+----+--------+-----+-------------+---------------------+

```

**场景一：** 找出所有姓名为张三且邮箱域为tuniu.com的用户。

```shell
mysql> select * from user where name='张三' and email like '%@tuniu.com';
```

```php
Yii::app()->db->createCommand()
    ->select()
    ->from('user')
    ->where(array('and', 'name="张三"', array('like', 'email', '%@tuniu.com')))
    ->queryAll();
```

**场景二：** 找出所有女性且手机号为13751770000，13651770000或13451770000的用户。

```shell
mysql> select * from user where sex = 1 and telphone in (13751770000, 13651770000, 13451770000);
```

```php
Yii::app()->db->createCommand()
    ->select()
    ->from('user')
    ->where(array('and', 'sex=1', array('in', 'telphone', array(13751770000, 13651770000, 13451770000))))
    ->queryAll();
```

**场景三：** 找出所有手机号前3位为131且邮箱域为php.net的男性用户。

```shell
mysql> select * from user where sex = 0 and telphone like '131%' and email like '%@php.net';
```

```php
Yii::app()->db->createCommand()
    ->select()
    ->from('user')
    ->where(array('and', 'sex=0', array('like', 'telphone', '131%'), array('like', 'email', '%@php.net')))
    ->queryAll();
```

### 总结

使用QB有很多好处，比如：可以使用OO的编程方式构建SQL语句；提供了某种程度的DB抽象，方便迁移到不同的DB平台等等。熟练掌握where方法的实现细节，就能让我们写出各种复杂业务场景的SQL语句。

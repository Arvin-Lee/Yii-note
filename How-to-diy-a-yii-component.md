## 如何自定义一个Yii组件

### 概览

1. 组件包括：

    **Yii自身核心组件** 和 **应用自定义组件**

2. 组件的生命周期(lifecycle)：

    注册核心组件(registerCoreComponents)
-> 注册应用组件(configure) -> 预加载组件(preloadComponents) -> 使用时实例化组件(getComponent)

### 何时需要组件

* 可复用的功能模块
* 标准化

如：城市组件，获取所有有效城市，根据code获取name，根据name获取code等

### 如何与Yii无缝衔接

继承抽象类CApplicationComponent或CComponent

### 好的实践
> 以CHttpRequest为例

* 区分公共属性和私有属性，并提供方法支持自定义私有属性值
```php
<?php
// 设置私有属性
private $_baseUrl;
// getter
public function getBaseUrl($absolute=false)
{
	if($this->_baseUrl===null)
		$this->_baseUrl=rtrim(dirname($this->getScriptUrl()),'\\/');
	return $absolute ? $this->getHostInfo() . $this->_baseUrl : $this->_baseUrl;
}
// setter
public function setBaseUrl($value)
{
	$this->_baseUrl=$value;
}
```

* 定义init方法
```php
<?php
public function init()
{
	parent::init();
    // 初始化工作
	$this->normalizeRequest();
}
```

* KISS - 每个方法只做一件事，代码尽量简洁
```php
<?php
// 获取入参
public function getParam($name,$defaultValue=null)
{
	return isset($_GET[$name]) ? $_GET[$name] : (isset($_POST[$name]) ? $_POST[$name] : $defaultValue);
}
// 是否Ajax请求
public function getIsAjaxRequest()
{
	return isset($_SERVER['HTTP_X_REQUESTED_WITH']) && $_SERVER['HTTP_X_REQUESTED_WITH']==='XMLHttpRequest';
}
```

* 健壮性 - 发生异常时抛出异常
```php
<?php
public function getScriptUrl()
{
	if($this->_scriptUrl===null)
	{
		$scriptName=basename($_SERVER['SCRIPT_FILENAME']);
		if(basename($_SERVER['SCRIPT_NAME'])===$scriptName)
			$this->_scriptUrl=$_SERVER['SCRIPT_NAME'];
		else if(basename($_SERVER['PHP_SELF'])===$scriptName)
			$this->_scriptUrl=$_SERVER['PHP_SELF'];
		else if(isset($_SERVER['ORIG_SCRIPT_NAME']) && basename($_SERVER['ORIG_SCRIPT_NAME'])===$scriptName)
			$this->_scriptUrl=$_SERVER['ORIG_SCRIPT_NAME'];
		else if(($pos=strpos($_SERVER['PHP_SELF'],'/'.$scriptName))!==false)
			$this->_scriptUrl=substr($_SERVER['SCRIPT_NAME'],0,$pos).'/'.$scriptName;
		else if(isset($_SERVER['DOCUMENT_ROOT']) && strpos($_SERVER['SCRIPT_FILENAME'],$_SERVER['DOCUMENT_ROOT'])===0)
			$this->_scriptUrl=str_replace('\\','/',str_replace($_SERVER['DOCUMENT_ROOT'],'',$_SERVER['SCRIPT_FILENAME']));
		else // 抛出异常 丢给框架去处理
			throw new CException(Yii::t('yii','CHttpRequest is unable to determine the entry script URL.'));
	}
	return $this->_scriptUrl;
}
```
* 组件/属性/方法都有详细注释
* 尽可能不依赖其它应用组件，保证可移植性

### 单元测试

> 没有实践 :(

### 部署

```php
<?php
...
return array(
    'components' => array(
        'reverter' => array(
            'class' => 'DataReverter', // 必需
            'property1' => 'value1', // 可选
            'property2' => 'value2', // 可选
        )
    ),
);
```

### 使用

* 当部署过组件时：`$reverter = Yii::app()->reverter;`
* 未部署过组件时：`$reverter = new DataReverter; $reverter->init();`

__EOF__

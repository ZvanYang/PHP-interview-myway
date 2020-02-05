## 1. 以下会输出什么？
```
<?php
$a = [];
$a[2] = 'baidu';
$a[1] = 'hello';
$a[3] = 'world';

var_dump($a);

array(3) {
  [2]=>
  string(5) "baidu"
  [1]=>
  string(5) "hello"
  [3]=>
  string(5) "world"
}
```


## 2. 以下会输出什么？

```
<?php
$a = array(1,2,3,4);
foreach($a as &$v){
	echo $v;  // 1,1,2,4,
     $v *= $v;
	echo $v;  // 3,9,4,16
} 
foreach($a as $v){
     echo $v; // 1,4,9,9
}

```
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
1. 缓存的更新模式


[!https://user-gold-cdn.xitu.io/2018/5/11/1634fc3319beda97?imageView2/0/w/1280/h/960/format/webp/ignore-error/1]


## 2. 这段代码有什么问题
```
$ok = $redis->setNX($key, $value);
if ($ok) {
    $cache->update();
    $redis->del($key);
}
```
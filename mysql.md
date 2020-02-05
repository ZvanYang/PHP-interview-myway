## 1. 查询一个人数超过20人的班级

```
CREATE TABLE `class` (
  `id` int(10) NOT NULL AUTO_INCREMENT,
  `name` varchar(20) COLLATE utf8mb4_general_ci NOT NULL DEFAULT '',
  `class_name` varchar(20) COLLATE utf8mb4_general_ci NOT NULL DEFAULT '',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=9 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci
```


```
select class_name, count(id) as nums from class group by class_name having nums > 20;
```
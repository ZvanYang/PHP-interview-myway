## 查询一个日志文件中访问次数最多前10个IP？
第一步：按照IP进行将记录排序。
第二步：按照IP去重，并且显示重复次数
第三步：按照次数升序排列
第四步：显示前10行
cat log.txt|awk -F" " '{print $1}' |sort|uniq -c|sort -nrt " "|awk -F" " 'print &2' |head -10

cat url.log | sort | uniq -c |sort -n -r -k 1 -t   ' ' | awk -F  '//'  '{print $2}' | head -10

## 常用的 linux 命令有哪些
````
touch 
mkdir 
rm 
ls 
cd 
chmod 
chgrp 
chown 
systemctl 
ps 
top 
df 
kill
ping
iostat
scp
lftp
screen。
uniq
wc
head
tail
find
whereis

````

# 借鉴一下别人的例子：比较 file1的1-4字符 和 file2的2-5 字符，如果相同，将file2 的第二列 与 file1 合并 file3（http://bbs.chinaunix.net/thread-577044-1-1.html） awk 处理多个文件的情况
````
$ cat file1
0011AAA 200.00 20050321 
0012BBB 300.00 20050621 
0013DDD 400.00 20050622 
0014FFF 500.00 20050401 

$ cat file2
I0011  11111 
I0012  22222 
I0014  55555 
I0013  66666 

$ awk 'NR==FNR{a[substr($1,1,4)]=$0}NR!=FNR&&a[b=substr($1,2,5)]{print a[b] $2}' file1 file2
0011AAA 200.00 20050321 11111
0012BBB 300.00 20050621 22222
0014FFF 500.00 20050401 55555
0013DDD 400.00 20050622 66666
````
当第一个文件的1-4个字段  与 第二个文件里面的2-5个里面的字段相同时，将文件进行合并。

# 查看日志一个ip 在一个小时内，请求了多少次？
````
查看某一时间段的IP访问量(4-5点)
grep "07/Apr/2017:0[4-5]" nginx.log | awk '{print $1}' | sort | uniq -c| sort -nr | wc -l
````

````
查看访问100次以上的IP
awk '{print $1}' nginx.log | sort -n |uniq -c |awk '{if($1 >100) print $0}'|sort -rn
````

很棒的一个介绍
https://codante.org/blog/post/service-log-common-commands/#%E6%9F%A5%E7%9C%8B%E6%9F%90%E4%B8%80%E6%97%B6%E9%97%B4%E6%AE%B5%E7%9A%84IP%E8%AE%BF%E9%97%AE%E9%87%8F-4-5%E7%82%B9

# 查询一个时间段内的ip请求量，取前100个值。
````
grep "03/Mar/2020:1[7-8]" access.log | awk '{print $1}' | sort -n | uniq -c | sort -nr | head -n 100

   3 127.0.0.1
   ````
# grep
搜索多个文件并查找匹配文本在哪些文件中：

grep -l "text" file1 file2 file3...
````
# 如何排查问题？
1. cpu是否沾满，一般会是mysql的索引有问题
2. NGINX的报错啊，502 504 等
3. php-fpm 的慢日志查询
4. MySQL的下游是否有问题。
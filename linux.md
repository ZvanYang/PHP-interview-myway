## 查询一个日志文件中访问次数最多前10个IP？
第一步：按照IP进行将记录排序。
第二步：按照IP去重，并且显示重复次数
第三步：按照次数升序排列
第四步：显示前10行
cat log.txt|awk -F" " '{print $1}' |sort|uniq -c|sort -nrt " "|awk -F" " 'print &2' |head -10

cat url.log | sort | uniq -c |sort -n -r -k 1 -t   ' ' | awk -F  '//'  '{print $2}' | head -10
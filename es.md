
## 项目的日志收集

通过filebeat读取日志，传送到kafka队列里，kafka再传给logstash过滤切割日志，logstash传到es里，最后kibana展示
https://www.jianshu.com/p/0a5acf831409
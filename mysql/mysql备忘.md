# 清除mysql缓存
```sql
RESET QUERY CACHE;
```
[clear-mysql-query-cache-without-restarting-server](http://stackoverflow.com/questions/5231678/clear-mysql-query-cache-without-restarting-server)

# 查询时指定不使用缓存
```sql
select SQL_NO_CACHE ...
```
[MYSQL SQL_NO_CACHE的真正含义](http://blog.sina.com.cn/s/blog_6e27a45f01015oyk.html)

# 设置binlog保存时间, 立刻释放磁盘空间
```sql
set global expire_logs_days = 1;
flush logs;
```



计算机领域最经常考虑到的性能优化手段就是缓存了。
能不能把结果缓存在内存中，下次只查内存就好了呢？所以做后端服务的时候，我们不会只用 mysql，一般会结合内存数据库来做缓存，最常用的是 redis。

data types

字符串命令
```
set // store a string value
setnx // stores a string value only if the key does not already exist.Useful 
for implementing locks.
get // retrieves a string value
mget // retrieves multiple string values in a single operation
```

incr 是自动增长

list类型
lrange list1 0 -1 现实所有数据
```
lpush // 新增一个新的元素到list的头部；
rpush // 增加到尾部
lpop // 删掉并且返回在list头部的元素；
rpop // 删掉并且返回list尾部的元素
llen // 返回list的长度
lmove // 将元素从一个list到另一个list
ltrim // 将列表指定到一定的范围
```

set类型无序并且元素不重复。
```
sadd 增加
srem 删除指定的数据
sismember 是否是属于这个集合
sinter 返回两个或者多个成员合集
scard 返回集合的基数
```

zset 可以排序 set不能排序.zset 每一个数据
```
zadd 将新成员和关联分数添加到排序中，如果已经存在，更新分数
zrange 返回在给定范围内排序的有序成员
zrank 返回所提供成员的排名，假设按升序排序。
zrevrank返回所提供成员的排名，假设排序集按降序排列。
```

hash 相当于map
```
hset 设置哈希上一个或多个字段的值
hget 返回给定指定字段的值
hmget 返回一个或多个字段的值
mincrby 将给定字段的值增加所提供的整数。
```


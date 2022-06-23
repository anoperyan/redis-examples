## 常见问题

**Redis的基本数据类型有哪些？**

共有八种类型，String，List，Hash，Set，SortSet，HyperLogLog，Geo，Pub/Sub。

**如果有大量的Key要设置在同一时间失效，一般要注意什么？**

可以为每个Key的失效时间加一个随机值，使得他们不在同一时间失效，从而防止Redis可能出现的短时间卡顿。
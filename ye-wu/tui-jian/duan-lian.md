# 短链

nginx：拦截ip攻击，负载均衡

主服务：

* bloom 查询链接是否存在
* redis  缓存热门短链和长链
* 请求分号器
  * 分号器用snowflake算法，不用hash怕冲突
  * 62进制压缩
  * 考虑用百度的uid generator，通过把时间改为long类型自增id解决时间回拨问题
* mysql
  * key短链val长链，两个字段都加索引，读写分离分库分表


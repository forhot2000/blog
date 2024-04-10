---
category: api
author: forhot2000@qq.com
date: 2024/01/17
---

# 时序数据查询接口几种设计方法

最近做了一个数据收集的项目，需要对外提供数据查询接口，将收集到所有的数据提供给数据分析程序调用，收集到的数据量很大，需要注意查询接口的性能问题。

## 方法一

提到 API，最先想到的当然就是提供 RESTful 的查询接口，接受时间查询条件、分页大小、当前页码，返回 json 数据。

```
GET /api/message?start_time={start_time}&end_time={end_time}&size={page_size}&page={page_index}
```

对应的查询 sql 如下

```sql
select *
from message
where createtime >= @start_time -- 如果有 start_time
and createtime < @end_time -- 如果有 end_time
order by createtime, id -- 排序中添加id解决createtime重复的问题
limit @page_size
offset @offset -- offset = page_size * (page_index - 1)
```

**_优点_**

- 简单通用，适合普通前端页面调用

**_缺点_**

- 数据量很大的情况下，查询后面的分页会越来越慢，直接会拖垮服务器
- 查询下一页的时候，如果前面的数据发生了变化，将会出现与上一页重复的数据，或者遗漏部分数据

## 方法二

始终根据时间查询，能够高效的利用时间字段的索引，能够保证下一页的数据紧跟上一页，不会出现重复数据或遗漏数据，适合连续获取分页数据的场景，比如数据分析服务拉取数据。

查询第一页，同方法一

```
GET /api/message?start_time={start_time}&end_time={end_time}&size={page_size}
```

对应的查询 sql 如下

```sql
select * 
from message 
where createtime >= @start_time -- 如果有 start_time
and createtime < @end_time -- 如果有 end_time
order by createtime, id -- 排序中添加id解决createtime重复的问题
limit @page_size
```

从第二页开始，根据上一页的最后一个结果的 createtime 和 id 来查询，在原来的 URL 后增加 `&last_time={last_time}&last_id={last_id}`

```
GET /api/message?start_time={start_time}&end_time={end_time}&size={page_size}&last_time={last_time}&last_id={last_id}
```

为了保护 last_time 和 last_id，可以在服务器端缓存 last_time 和 last_id，仅告诉客户端一个 next_token，客户端请求下一页的时候直接传入服务器端发下的 next_token，在原来的 URL 后增加 `&next_token={next_token}`

```
/api/message?start_time={start_time}&end_time={end_time}&size={page_size}&next_token={next_token}
```

对应的查询 sql 如下

```sql
select *
from message
where createtime < {end_time} -- 如果有 end_time
and createtime > {last_time} -- 覆盖了 createtime >= @start_time
and id > {last_id}
order by createtime, id -- 排序中添加id解决createtime重复的问题
limit {page_size}
```

**_优点_**

- 不会因为页码太大导致查询速度变慢
- 下一页始终紧跟上一页结果，不会出现重复数据或遗漏数据

**_缺点_**

- 没有页码，前端无法显示当前页页码，无法任意跳转分页

## 方法三

在方法二的基础上，提供 websocket 的接口，利用 websocket 的长链接特性，将 next_token 存在服务器端，客户端每次只要发送 next 命令即可接收下一页的数据。

客户端 websocket 的消息队列如下

```
↑ {"cmd":"query","data":{"page_size":$page_size,"where":{"start_time":"$start_time","end_time":"$end_time"}}}
↓ {"data":[...]}
↑ {"cmd":"next"}
↓ {"data":[...]}
↑ {"cmd":"next"}
↓ {"data":[...]}
```

> 注意：始终应该由客户端发起 next 命令，防止服务器端发送数据过快，导致客户端来不及消费数据的情况。

**_优点_**

- 减少了 HTTP 的握手次数，提高了性能
- 对用户屏蔽了 next_token，简化了调用方的逻辑

**_缺点_**

- 需要额外处理网络不稳定导致 websocket 断开的情况

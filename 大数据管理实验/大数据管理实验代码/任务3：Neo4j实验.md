### 1.
```cypher
MATCH (city:CityNode) RETURN city LIMIT 10
```
### 2.
```cypher
MATCH(business:BusinessNode) 
WHERE business.city='Ambridge' 
RETURN business
```
### 3.
```cypher
MATCH(review:ReviewNode{reviewid:'rEITo90tpyKmEfNDp3Ou3A'})
-[:Reviewed]->(business:BusinessNode)
RETURN properties(business)
```
### 4.
```cypher
MATCH(r:ReviewNode)-[:Reviewed]->(business:BusinessNode{businessid:'fyJAqmweGm8VXnpU4CWGNw'})
MATCH(user:UserNode)-[:Review]->(r)
RETURN user.name,user.fans
```
### 5.
```cypher
MATCH(u:UserNode{userid:'TEtzbpgA2BFBrC0y0sCbfw'})
-[:Review]->(r:ReviewNode)
MATCH(r)-[:Reviewed]->(business:BusinessNode)
WHERE r.stars='5.0'
RETURN business.name,business.address
```
### 6.
```cypher
MATCH(business:BusinessNode)
RETURN business.name,business.stars,business.address
ORDER BY business.stars DESC
LIMIT 15
```
### 7.
```cypher
MATCH(user:UserNode)
WHERE toInteger(user.fans)>200
RETURN user.name,user.fans
LIMIT 10
```
### 8.
```cypher
PROFILE
MATCH( b:BusinessNode)-[:IN_CATEGORY]->(c:CategoryNode)
WHERE b.businessid='tyjquHslrAuF5EUejbPfrw'
RETURN count(c)
```
### 9.
```cypher
MATCH( b:BusinessNode)-[:IN_CATEGORY]->(c:CategoryNode)
WHERE b.businessid='tyjquHslrAuF5EUejbPfrw'
RETURN collect(c.category) AS category
```
### 10.
```cypher
MATCH (a:UserNode {name: 'Allison'})-[:HasFriend]->(b:UserNode)
MATCH (b)-[:HasFriend]->(c:UserNode)
with b ,count(c) AS numberOfFoFs
RETURN b.name AS friendsList,numberOfFoFs
```
### 11.
```cypher
MATCH (a:BusinessNode)-[:IN_CATEGORY]->(b:CategoryNode{category:'Salad'})
MATCH (a)-[:IN_CITY]->(c:CityNode)
WITH c,count(a) AS `count(*)`
RETURN c.city AS `business.city`,`count(*)`
ORDER BY `count(*)` DESC
LIMIT 5													
```
### 12.
```cypher
MATCH (a:BusinessNode)
RETURN a.name AS name,count(a) AS cnt
ORDER BY cnt DESC
LIMIT 10
```
### 13.
```CYPHER
MATCH (total_business:BusinessNode)
WITH count(DISTINCT total_business.name) AS cnt

MATCH (business:BusinessNode)
WHERE toInteger(business.reviewcount) > 5000
WITH business,cnt

MATCH(b:BusinessNode)
WHERE b.name=business.name
WITH business.name AS name,count(*) AS frequency,cnt,business.reviewcount AS reviewcount

RETURN frequency*1.0/cnt AS heat,name,reviewcount
ORDER BY reviewcount DESC
```
### 14.
```cypher
MATCH(business:BusinessNode{stars:'5.0'})-[:IN_CATEGORY]->(c:CategoryNode{category:'Zoos'})
WITH business
MATCH(business)-[:IN_CITY]->(city:CityNode)
RETURN city;
```
### 15.
```CYPHER
MATCH(u:UserNode)-[:Review]->(r:ReviewNode)-[:Reviewed]->(business:BusinessNode)
WITH business,count(*) AS user_count
RETURN business.businessid,business.name,user_count
ORDER BY user_count DESC
LIMIT 10
```
### 16.
- 新建属性
```CYPHER
MATCH(u:UserNode)
WITH u
LIMIT 100000
SET u.flag=456
```
- 查询 用时1109ms->7ms
```CYPHER
MATCH(u:UserNode)
WHERE u.flag>8000
RETURN u
```
- 创建 用时689ms->525ms
```CYPHER
MATCH(u:UserNode)
WITH u
SKIP 100000
LIMIT 100000
SET u.flag=10000
```
- 更新 用时1287ms->710ms
```CYPHER
MATCH(u:UserNode)
WHERE u.flag=456
SET u.flag=3456
```
- 删除 用时1266ms->828ms
```CYPHER
MATCH(u:UserNode)
WHERE u.flag>4000
REMOVE u.flag
```
- 建立索引
```CYPHER
CREATE INDEX ON :UserNode(flag)
```
### 17.
```CYPHER
PROFILE
MATCH (user1:UserNode{userid:"tvZKPah2u9G9dFBg5GT0eg"})-[:Review]->(r1:ReviewNode)-[:Reviewed]->(business:BusinessNode)<-[:Reviewed]-(r2:ReviewNode)<-[:Review]-(user2:UserNode)
WHERE NOT (user1)-[:HasFriend]->(user2)
WITH user1,user2,count(DISTINCT business) AS num	
RETURN user1.name,user2.name,num			
ORDER BY num DESC			 
```
### 18.
- neo4j  14685ms
```cypher
MATCH(r:ReviewNode)-[:Reviewed]->(b:BusinessNode)
WHERE r.reviewid='TIYgnDzezfeEnVeu9jHeEw'
RETURN b
```
- mongodb 266ms
``` javascript
db.business.find({
  'business_id': db.review.findOne({'review_id':'TIYgnDzezfeEnVeu9jHeEw'}).business_id
}).explain('executionStats')

```

- neo4j适合中小规模数据集和复杂的图分析任务
- mongo适合大规模数据并且能处理高并发的请求
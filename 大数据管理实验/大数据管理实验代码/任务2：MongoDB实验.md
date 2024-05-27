- 启动mongo
```JAVASCRIPT
mongod --dbpath /var/lib/mongodb/ --logpath /var/log/mongodb/mongodb.log --logappend &
```
### 1.
```
db.review.find().limit(2).skip(6).pretty()
```
### 2.
```
db.business.find({'city':'Las Vegas'}).limit(5).pretty()
```
### 3.
```
db.user.find({'name':'Steve'},{useful:1,cool:1}).limit(10).pretty()
```
### 4.
```
db.user.find({'funny':{$in:[66,67,68]}},{name:1,funny:1}).limit(20).pretty()
```
### 5.
```
db.user.find({'cool':{$gte:15,$lt:20},'useful':{$gte:50}}).limit(10).pretty()
```
### 6.
- 统计
```
db.business.count()
```
- 查询执行计划
```
db.business.explain('executionStats').count()
```
### 7.
```
db.business.find({$or:[{'city':'Westlake'},{'city':'Calgary'}]}).pretty()
```
### 8.
```
db.business.find({'categories':{$size:6}},{categories:1}).limit(10).pretty()
```
### 9.
- 查询执行计划
```
db.business.find({business_id: "5JucpCfHZltJh5r1JabjDg"}).explain('executionStats')
```
![[Snipaste_2023-10-25_11-02-52.png]]
- 建立索引
```
db.business.createIndex({business_id:1})
```
![[Snipaste_2023-10-25_11-07-31.png]]
### 10.
```
db.business.aggregate([
{$group:{cnt:{$sum:1},_id:'$stars'}},
{$sort:{_id:-1}},
{$project:{_id:0,cnt:1,stars:'$_id'}}
])
```
### 11.
- 创建子集合
```
db.review.aggregate([
    { $limit: 500000 },
    { $out: "Subreview" }
])
```
- 建立索引
```
db.Subreview.createIndex({text:'text'})
db.Subreview.createIndex({useful:1})
```
- 查询
```
db.Subreview.find({text:{$regex:/delicious/},useful:{$gt:9}}).pretty()
```
### 12.
```
db.Subreview.aggregate([
{$match:{useful:{$gt:6},funny:{$gt:6},cool:{$gt:6}}},
{$group:{_id:'$business_id',stars_avg:{$avg:'$stars'}}},
{$sort:{_id:-1}},
{$project:{_id:0,stars_avg:1,business_id:'$_id'}}
])
```
### 13.
- 建立索引
```
db.business.createIndex({loc:'2dsphere'})
```
- 查询 
```
db.business.find({
loc:{
$near:{
$geometry:db.business.findOne({business_id:'xvX2CttrVhyG2z1dFg_0xw'}).loc,
$maxDistance:100}}},{_id:0,name:1,address:1,stars:1}).pretty()
```
### 14.
- 建立索引
```
db.Subreview.createIndex({user_id:1})
```
- 查询
```
db.Subreview.aggregate([
{
	$addFields: {
            dateISO: {
                $dateFromString: {
                    dateString: '$date',
                    format: '%Y-%m-%d %H:%M:%S'
                }
            }
        }
},
{$match:{ dateISO: { $gte:ISODate("2017-01-01T00:00:00Z")  }}},
{$group:{_id:'$user_id',total:{$sum:1}}},
{$sort:{total:-1}},
{$limit:20},
{$project:{_id:0,total:1,user_id:'$_id'}}
])
```
### 15.
- map-reduce
```
db.Subreview.mapReduce(
function() {emit(this.business_id,{count:1,stars:this.stars});},
function(key,values) {
	var result ={stars:0,count:0};
	values.forEach(function(value){
		result.stars+=value.stars;
		result.count+=value.count;
	});
	return result;
},
{
	out:'test_map_reduce',
	finalize:function(key,reducedValue)
	{
		reducedValue.avg=reducedValue.stars/reducedValue.count;
		return reducedValue;
	}
}
)
```
- 查看结果
```
db.test_map_reduce.find().pretty()
```
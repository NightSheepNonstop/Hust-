### 1.
```MYSQL
SELECT business_id,business_info
FROM business
WHERE JSON_EXTRACT(business_info,'$.city')='Tampa'
ORDER BY JSON_EXTRACT(business_info,'$.review_count') desc
limit 10;
```
### 2.
```MYSQL
SELECT JSON_KEYS(business_info) AS keys_info,JSON_LENGTH(JSON_KEYS(business_info)) AS key_num_info,JSON_KEYS(JSON_EXTRACT(business_info,'$.attributes')) AS keys_attr, JSON_LENGTH(JSON_KEYS(JSON_EXTRACT(business_info,'$.attributes')) )AS key_num_attr
FROM business
limit 5;
```
### 3.
```mysql
SELECT 
JSON_EXTRACT(business_info,'$.name') AS name,
JSON_TYPE(JSON_EXTRACT(business_info,'$.name')) AS name_type,
JSON_EXTRACT(business_info,'$.stars') AS stars,
JSON_TYPE(JSON_EXTRACT(business_info,'$.stars')) AS stars_type,
JSON_EXTRACT(business_info,'$.attributes') AS attributes,
JSON_TYPE(JSON_EXTRACT(business_info,'$.attributes')) AS attributes_type
FROM business
limit 5;
```
### 4.
```mysql
SELECT 
JSON_EXTRACT(business_info,'$.name') AS name,
JSON_EXTRACT(business_info,'$.attributes') AS attributes,
JSON_EXTRACT(business_info,'$.hours') AS open_time
FROM business
WHERE
JSON_EXTRACT(JSON_EXTRACT(business_info,'$.attributes'),'$.HasTV') = 'True' AND ISNULL(JSON_EXTRACT(JSON_EXTRACT(business_info,'$.hours'),'$.Sunday'))
ORDER BY name
LIMIT 10;
```
### 5.
```mysql
EXPLAIN FORMAT=JSON select * from user where user_info->'$.name'='Wanda';
SHOW PROFILES;
```
```mongo
db.user.find({"name":"Wanda"}).pretty().explain("executionStats")
```
mysql执行时间约为34s，mongodb耗时约为10s，查询效率比为1：3.4
### 6.
- 查询语句：
```MYSQL
SELECT JSON_PRETTY(business_info)
FROM business
WHERE business_id='4r3Ck65DCG1T6gpWodPyrg';
```
修改前：
![[Snipaste_2023-10-22_18-41-16.png]]
- 修改语句：
```MYSQL
UPDATE business SET business_info=JSON_SET(business_info,'$.hours.Tuesday','16:0-23:0','$.stars',4.5,'$.attributes.WiFi','Free')
WHERE business_id='4r3Ck65DCG1T6gpWodPyrg';
```
修改后：
![[Snipaste_2023-10-22_19-10-41.png]]
### 7.
- 插入
```mysql
INSERT INTO business (business_id, business_info)
SELECT 'aaaaaabbbbbbcccccc2023', business_info
FROM business
WHERE business_id = '5d-fkQteaqO6CSCqS5q4rw';
```
- 删除
```MYSQL
UPDATE business
SET business_info=JSON_REMOVE(business_info,'$.name')
WHERE business_id='aaaaaabbbbbbcccccc2023';
```
- 查询
```MYSQL
SELECT *
FROM business
WHERE business_id='aaaaaabbbbbbcccccc2023';
```
### 8.
```mysql
SELECT state, 
       JSON_OBJECTAGG(city, city_count) AS city_occ_num
FROM (
    SELECT business_info ->> '$.state' AS state,
           business_info ->> '$.city' AS city,
           COUNT(*) AS city_count
    FROM business
    GROUP BY state, city
) AS subquery
GROUP BY state
ORDER BY state ;
```
### 9.
```mysql
WITH FRIENDS AS(
	SELECT *
	FROM user
	WHERE REGEXP_LIKE(user_info ->> '$.friends','__1cb6cwl3uAbMTK3xaGbg')
)
SELECT tip.user_id,user_info -> '$.name' AS name,JSON_ARRAYAGG(tip_info ->>'$.text') AS text_array
FROM tip
JOIN FRIENDS ON tip.user_id=FRIENDS.user_id
GROUP BY tip.user_id
ORDER BY name;
```
### 10.
```mysql
WITH EDMONTON_SHOP AS (
	SELECT business_info->>'$.name' AS name,business_info->>'$.hours' AS hours
	FROM business
	WHERE business_info ->> '$.city'='EdMonton'
),
ELSMERE_SHOP AS (
	SELECT business_info->>'$.name' AS name,business_info->>'$.hours' AS hours
	FROM business
	WHERE business_info ->> '$.city'='Elsmere'
)
	SELECT EDMONTON_SHOP.name AS name1,'EdMonton' AS city1,ELSMERE_SHOP.name AS name2,'Elsmere' AS city2,EDMONTON_SHOP.hours AS hours1,ELSMERE_SHOP.hours AS hours2,JSON_OVERLAPS(EDMONTON_SHOP.hours,ELSMERE_SHOP.hours)AS has_same_opentime
	FROM EDMONTON_SHOP,ELSMERE_SHOP;
```
### 11.
```mysql
EXPLAIN FORMAT=JSON
SELECT 
    user_info ->'$.name' AS name,
    user_info -> '$.average_stars' AS avg_stars,
    JSON_ARRAYAGG(CONCAT(
	    user_info ->>'$.funny', ',',
        user_info ->>'$.useful', ',',
        user_info ->>'$.cool', ',',
        user_info ->>'$.funny' + user_info ->>'$.useful' +user_info ->>'$.cool'
    ))
    AS `[funny,useful,cool,sum]`
FROM 
    user
WHERE 
    user_info ->>'$.funny' > 2000 AND  user_info ->> '$.average_stars'> 4.0
GROUP BY name,avg_stars
ORDER BY avg_stars DESC
LIMIT 10;

```
### 12.
```MYSQL
SELECT JSON_PRETTY(JSON_MERGE_PRESERVE(b.business_info, u.user_info))
FROM (
    SELECT business_id
    FROM tip
    GROUP BY business_id
    HAVING COUNT(*) >= ALL(
        SELECT COUNT(*)
        FROM tip
        GROUP BY business_id
    )
) AS max_business
JOIN (
    SELECT user_id
    FROM tip
    GROUP BY user_id
    HAVING COUNT(*) >= ALL(
        SELECT COUNT(*)
        FROM tip
        GROUP BY user_id
    )
) AS max_user
JOIN business AS b ON max_business.business_id = b.business_id
JOIN user AS u ON max_user.user_id = u.user_id;

```

### 13.  
```mysql
WITH TOP_BUSINESSES AS (
	SELECT business_info
	FROM business
	ORDER BY JSON_EXTRACT(business_info, '$.review_count') DESC
	LIMIT 3
)
SELECT t1.*,t2.num, t2.hours_in_a_week
FROM  
    TOP_BUSINESSES 
JOIN 
    JSON_TABLE(business_info, '$' COLUMNS (
		business_name VARCHAR(100) PATH '$.name',
		business_review_count INT PATH '$.review_count',
		business_open_on_Tuesday VARCHAR(100) EXISTS PATH '$.hours.Tuesday'
	)
) AS t1 
RIGHT JOIN 
	(
		SELECT ROW_NUMBER() OVER(PARTITION BY JSON_EXTRACT(business_info ,'$.name')  ORDER BY id) AS num,JSON_EXTRACT(business_info ,'$.name') AS business_name,t21.hours_in_a_week
		FROM TOP_BUSINESSES 
		JOIN
		JSON_TABLE(
			JSON_ARRAY(
			JSON_EXTRACT(business_info,'$.hours.Monday'),
		    JSON_EXTRACT(business_info,'$.hours.Tuesday'),
		    JSON_EXTRACT(business_info,'$.hours.Wednesday'),
		    JSON_EXTRACT(business_info,'$.hours.Thursday'),
		    JSON_EXTRACT(business_info,'$.hours.Friday'),
		    JSON_EXTRACT(business_info,'$.hours.Saturday'),
		    JSON_EXTRACT(business_info,'$.hours.Sunday')
			)
	    , 
	    '$[*]' COLUMNS (
	    id FOR ORDINALITY,
	    hours_in_a_week VARCHAR(100) PATH '$')
		)AS t21
		WHERE hours_in_a_week IS NOT NULL
	)AS t2 ON t1.business_name=t2.business_name
ORDER BY business_name,num;

```

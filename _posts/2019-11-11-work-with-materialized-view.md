---
title: "Re-build Materialized View with Relationship"
date: 2019-11-11
tags: [Data Science, PostgreSQL]
---

If you have every working with PostgreSQL, you would have known that there are numerous wonderful features that other SQL servers would be jealous. However, due to its level of advanceness, there are also a different set of issues on its own, and one of them is the relationship between Materialized View (MV), which is a well-known feature of PostgreSQL. 

Most average SQL users may never know about MV, but if they are using PostgreSQL for a while, high chance they already use it without knowing. Common Table Expression (CTE) or usually known as the "WITH ... AS" method is a well-known method that creates temporary table that can be re-used multiple times and help improve the cleanliness of the script that usually get super fuzzy due to the sub-query. The reason why I mentions CTE here is because CTE actually create a hidden MV that is removed right after the query is finalized. MV allows CTE to be faster when the temporary table gets larger and more complicated. 

So now we know MT is important, lets look into the issue created due to the uniqueness of MV, which is Relationship. Let's setup a simple sitation, we have a table `player_detail` that stores football players' name and their unique ID that are crawled from www.futbin.com. 

player_id | player_name
--- | ---
1 | Cristiano Ronaldo
2 | Thierry Henry
3 | Carles Puyol Saforcada
4 | Carlos Alberto Torres
5 | Johan Cruyff


Now we want to create a MV having only players with name starting with "C": 

```sql
CREATE MATERIALIZED VIEW data_sch.player_named_c AS 
SELECT 
        player_name 
FROM    data_sch.player_detail 
WHERE   1=1
AND     player_name LIKE 'C%' 
```


With this, the MV `player_named_c`sets a relationship with table `player_detail` so that even if we change its name into `player_detail_v2`, the MV will still remembers the relationship and change the query accordingly. Quite convenient, right? It is surely a simple way to right your temporary tables, especially if you are using it in different procedure. Changing the normal table is easy, but new problem arises, if you want to change the logic or add a new column or conditions into the MV, you will need to drop it and rebuild it, here is how: 

```sql
DROP MATERIALIZED VIEW IF EXIST data_sch.player_named_c;
CREATE MATERIALIZED VIEW data_sch.player_named_c AS 
SELECT 
        player_id
        , player_name 
FROM    data_sch.player_detail 
WHERE   1=1
AND     player_name LIKE 'C%' 
```


That is just the beginning, let's say you want to use that MV to create another MV of players' name start with "C" and names' length is more than 20 characters. 

```sql
CREATE MATERIALIZED VIEW data_sch.player_named_c_v2 AS 
SELECT 
        player_id
        , player_name  
FROM    data_sch.player_named_c 
WHERE   1=1
AND     player_name LIKE 'C%' 
AND     LENGTH(player_name) > 20
```

Now, the trouble will start to arises, so if you start to realize that you also want players with not just first name but also last name starting with "C", and you want to change the `player_named_c` table, you do the DROP and CREATE, but you encounter an issue as follow: 

`ERROR: cannot drop materialized view player_named_c because other objects depend on it`

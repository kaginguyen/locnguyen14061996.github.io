---
title: "Re-build Materialized View having Relationship"
date: 2019-11-11
tags: [Data Science][PosgreSQL]
---

If you have every working with PostgreSQL, you would have known that there are numerous wonderful features that other SQL servers would be jealous. However, due to its level of advanceness, there are also a different set of issues on its own, and one of them is the relationship between Materialized View (MV), which is a well-known feature of PostgreSQL. 

Most average SQL users may never know about MV, but if they are using PostgreSQL for a while, high chance they already use it without knowing. Common Table Expression (CTE) or usually known as the "WITH ... AS" method is a well-known method that creates temporary table that can be re-used multiple times and help improve the cleanliness of the script that usually get super fuzzy due to the sub-query. The reason why I mentions CTE here is because CTE actually create a hidden MV that is removed right after the query is finalized. MV allows CTE to be faster when the temporary table gets larger and more complicated. 

So now we know MT is important, lets look into the issue created due to the uniqueness of MV, which is Relationship. Let's setup a simple sitation, we have a table that stores football players' name and their unique ID that are crawled from www.futbin.com. Now we want to create a MV having only players with name starting with "C": 

```sql
CREATE MATERIALIZED VIEW AS 
SELECT 
        player_name 
FROM    data_sch.player_detail 
WHERE   1=1
AND     player_name LIKE 'C%' 
```


---
title: "Re-build Materialized View with Relationship"
date: 2019-11-11
tags: [Data Science, PostgreSQL]
---

If you have every working with PostgreSQL, you would have known that there are numerous wonderful features that other SQL servers would be jealous. However, due to its level of advanceness, there are also a different set of issues on its own, and one of them is the relationship between Materialized View (MV), which is a well-known feature of PostgreSQL. 
<br>

Most average SQL users may never know about MV, but if they are using PostgreSQL for a while, high chance they already use it without knowing. Common Table Expression (CTE) or usually known as the "WITH ... AS" method is a well-known method that creates temporary table that can be re-used multiple times and help improve the cleanliness of the script that usually get super fuzzy due to the sub-query. The reason why I mentions CTE here is because CTE actually create a hidden MV that is removed right after the query is finalized. MV allows CTE to be faster when the temporary table gets larger and more complicated. 
<br>

So now we know MV is important, lets look into the issue created due to the uniqueness of MV, which is Relationship. Let's setup a simple sitation, we have a table `player_detail` that stores football players' name and their unique ID that are crawled from www.futbin.com. 

player_id | player_name
--- | ---
1 | Cristiano Ronaldo
2 | Thierry Henry
3 | Carles Puyol Saforcada
4 | Carlos Alberto Torres
5 | Johan Cruyff

<br>

Now we want to create a MV having only players with name starting with "C": 

```sql
CREATE MATERIALIZED VIEW data_sch.player_named_c AS 
SELECT 
        player_name 
FROM    data_sch.player_detail 
WHERE   1=1
AND     player_name LIKE 'C%' 
```
<br>

With this, the MV `player_named_c`sets a relationship with table `player_detail` so that even if we change its name into `player_detail_v2`, the MV will still remembers the relationship and change the query accordingly. Quite convenient, right? It is surely a simple way to write your temporary tables, especially if you are using it in different procedure but you dont want to re-run it every time. 
<br>

So changing the normal table is easy, but new problem arises, if you want to change the logic or add a new column or conditions into the MV, you will need to drop it and rebuild it, here is how: 

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
<br>

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
<br>

Now, the trouble will start to arises, so if you start to realize that you also want players with not just first name but also last name starting with "C", and you want to change the logic of `player_named_c` , you do the DROP and CREATE, but you encounter an issue as follow: 

`ERROR: cannot drop materialized view player_named_c because other objects depend on it`
<br>

Due to the relationship that `player_named_c_v2` set with `player_named_c`, it is now *impossible* to change the logic of `player_named_c` without re-building everything. When I say everything, I actually mean it, you will now have to DROP both MVs, rebuild `player_named_c` first then `player_named_c_v2`. And that is just 2 levels of relationship. Now, imagine we have more levels, with more complicated relationship, such as a MV at level 3, requiring data of a MV level 1 and a MV of level 2 and a normal table. 
<br>

Well that is my situation when I took over managing a forecast engine from my manager when he left the company. Not talking if I can understand how the forecast engine works, if I can't change the logic inside the MVs, there is no way I can go any further. And it took me quite some time to figure it out, let's me share about my solution with an explation on logic first which can be applied to any programming language (and of course SQL is not enough), and a demonstration using Python. 
<br>

Basically, any MV would be built from tables, normal or maybe other MVs. Those other MVs can be built from other MVs, and it can go on forever, but at the end, there will surely be a MV that is built from normal tables. So first, lets build a list of base MVs that are built purely using normal tables. To do that, lets look at how we get out all MVs and their relationships using the following query: 

```sql 
SELECT 
        DISTINCT view_cs.nspname
        , view_c.relname
        , tab_cs.nspname
        , tab_c.relname
FROM    pg_depend view_d
JOIN    pg_class view_c ON view_c.oid = view_d.refobjid AND view_c.relkind = 'm'
JOIN    pg_type view_ct ON view_ct.oid = view_c.reltype
JOIN    pg_namespace view_cs ON view_cs.oid = view_ct.typnamespace
JOIN    pg_depend tab_d ON tab_d.objid = view_d.objid
JOIN    pg_class tab_c ON tab_c.oid = tab_d.refobjid AND tab_c.relkind = 'r'
JOIN    pg_type tab_ct ON tab_ct.oid = tab_c.reltype
JOIN    pg_namespace tab_cs ON tab_cs.oid = tab_ct.typnamespace
WHERE   1=1
AND     view_d.deptype = 'n'
```
<br>

Using the query above, we will get all MVs and their relationships, make adjust so it can get MVs from other schema if you want. In this query, dependent MVs will have the depending tables separate into normal or MV, so as to do the first step, get MVs that does not have any depending MV. 
<br>

Now we have our list of base MVs. From this, the idea is to run a loop (of course, using another programming language) that will check the others dependant MVs if their depending MVs have already in the list of base MVs, if so, proceed to add them into the list and continue the loop. Be aware that we need to have an indicator to know which round an MV belongs to, normally I name the list of base MVs as Level 0, then continue to increase to Level 1, Level 2, etc. with each loop until finish. With that indicator, you can now know the order that MVs need to be re-built. 
<br>

Now we know the order of re-building MVs, we need to aware that we need to DROP all MVs first, so we need to have the logic of the MVs stored somewhere, luckily in my query, the query used to build the MV are also available, so when writing your script, take advantage of that. And if you want to change the logic of a MV, do it during the process of re-building, and that is it. 
<br> 

For some people, it can be difficult to understand explanation like that. Therefore, let's talk in programming language, here is my Python scripts that I use to do the re-building:
<br>

```python 
from pyool import PostgreSQLConnector 

db = PostgreSQLConnector()
db.connect()
mv_list_query = """
                SELECT 
                        DISTINCT view_cs.nspname
                        , view_c.relname
                        , tab_cs.nspname
                        , tab_c.relname
                FROM    pg_depend view_d
                JOIN    pg_class view_c ON view_c.oid = view_d.refobjid AND view_c.relkind = 'm'
                JOIN    pg_type view_ct ON view_ct.oid = view_c.reltype
                JOIN    pg_namespace view_cs ON view_cs.oid = view_ct.typnamespace
                JOIN    pg_depend tab_d ON tab_d.objid = view_d.objid
                JOIN    pg_class tab_c ON tab_c.oid = tab_d.refobjid AND tab_c.relkind = 'r'
                JOIN    pg_type tab_ct ON tab_ct.oid = tab_c.reltype
                JOIN    pg_namespace tab_cs ON tab_cs.oid = tab_ct.typnamespace
                WHERE   1=1
                AND     view_d.deptype = 'n'
                """
df = db.run_query(mv_list_query, return_data = True)

```
<br>

Minor note, there is a strange package pyool which is built by me in order to simplify the task of running PostgreSQL jobs, you can use psycopg2 for better control. 

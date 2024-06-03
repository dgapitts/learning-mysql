## Mysql pk versus secondary index efficiency

### Generating randomo datasets the mysql way

Apparently this isn't easy i.e. based on this [how-to-generate-1000000-rows-with-random-data stackoverflow post](https://stackoverflow.com/questions/25098747/how-to-generate-1000000-rows-with-random-data):

```
create database test;
use test;

CREATE TABLE `data` 
(
  `id`         bigint(20) NOT NULL      AUTO_INCREMENT,
  `datetime`   timestamp  NULL          DEFAULT CURRENT_TIMESTAMP,
  `channel`    int(11)                  DEFAULT NULL,
  `value`      float                    DEFAULT NULL,

  PRIMARY KEY (`id`)
);


DELIMITER $$
CREATE PROCEDURE generate_data()
BEGIN
  DECLARE i INT DEFAULT 0;
  WHILE i < 1000 DO
    INSERT INTO `data` (`datetime`,`value`,`channel`) VALUES (
      FROM_UNIXTIME(UNIX_TIMESTAMP('2014-01-01 01:00:00')+FLOOR(RAND()*31536000)),
      ROUND(RAND()*100,2),
      1
    );
    SET i = i + 1;
  END WHILE;
END$$
DELIMITER ;

CALL generate_data();
```

next to demo the secondary index affect add a secondary index on column id2 


```
update data set id2 = id where 1=1;
create index data_idx on data(id2);
create unique index data_idx_uniq on data(id2);
```

and then run `show index data;`

```
mysql> show index from data;
+-------+------------+---------------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+
| Table | Non_unique | Key_name      | Seq_in_index | Column_name | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment | Index_comment | Visible | Expression |
+-------+------------+---------------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+
| data  |          0 | PRIMARY       |            1 | id          | A         |        1000 |     NULL |   NULL |      | BTREE      |         |               | YES     | NULL       |
| data  |          0 | data_idx_uniq |            1 | id2         | A         |        1000 |     NULL |   NULL | YES  | BTREE      |         |               | YES     | NULL       |
| data  |          1 | data_idx      |            1 | id2         | A         |        1000 |     NULL |   NULL | YES  | BTREE      |         |               | YES     | NULL       |
+-------+------------+---------------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+
3 rows in set (0.01 sec)
```


now for individual short operations (single value lookups) it is hard to see the diference - at least at 1/100 second level


```
mysql> explain analyze select * from data where id2 = 345;
+-------------------------------------------------------------------------------------------------------+
| EXPLAIN                                                                                               |
+-------------------------------------------------------------------------------------------------------+
| -> Rows fetched before execution  (cost=0.00..0.00 rows=1) (actual time=0.000..0.001 rows=1 loops=1)
 |
+-------------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)

mysql> explain analyze select * from data where id = 345;
+-------------------------------------------------------------------------------------------------------+
| EXPLAIN                                                                                               |
+-------------------------------------------------------------------------------------------------------+
| -> Rows fetched before execution  (cost=0.00..0.00 rows=1) (actual time=0.000..0.000 rows=1 loops=1)
 |
+-------------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)
```


but this is more telling

```
mysql> explain analyze select * from data where id2 > 800;
+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| EXPLAIN                                                                                                                                                                        |
+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| -> Index range scan on data using data_idx_uniq over (800 < id2), with index condition: (`data`.id2 > 800)  (cost=90.26 rows=200) (actual time=0.065..0.645 rows=200 loops=1)
 |
+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.01 sec)

mysql> explain analyze select * from data where id > 800;
+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| EXPLAIN                                                                                                                                                                                                                           |
+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| -> Filter: (`data`.id > 800)  (cost=40.35 rows=200) (actual time=0.081..0.358 rows=200 loops=1)
    -> Index range scan on data using PRIMARY over (800 < id)  (cost=40.35 rows=200) (actual time=0.077..0.315 rows=200 loops=1)
 |
+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)
```



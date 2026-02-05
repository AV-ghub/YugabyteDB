##### [linux](https://download.yugabyte.com/#linux)
```
wget https://software.yugabyte.com/releases/2025.2.0.1/yugabyte-2025.2.0.1-b1-linux-x86_64.tar.gz
tar xvfz yugabyte-2025.2.0.1-b1-linux-x86_64.tar.gz && cd yugabyte-2025.2.0.1/
./bin/post_install.sh
./bin/yugabyted start

./bin/ysqlsh -h 10.19.57.138  -U yugabyte -d yugabyte
```
```
+-----------------------------------------------------------------------------------------------------------+
|                                                 yugabyted                                                 |
+-----------------------------------------------------------------------------------------------------------+
| Status              : Running.                                                                            |
| YSQL Status         : Ready                                                                               |
| Replication Factor  : 1                                                                                   |
| YugabyteDB UI       : http://xx.xx.xx.xx:15433                                                           |
| JDBC                : jdbc:postgresql://xx.xx.xx.xx:5433/yugabyte?user=yugabyte&password=yugabyte        |
| YSQL                : bin/ysqlsh -h xx.xx.xx.xx  -U yugabyte -d yugabyte                                 |
| YCQL                : bin/ycqlsh xx.xx.xx.xx 9042 -u cassandra                                           |
| Data Dir            : /root/var/data                                                                      |
| Log Dir             : /root/var/logs                                                                      |
| Universe UUID       : 48cc99d2-4bdd-42c0-a977-1d45665c0b95                                                |
+-----------------------------------------------------------------------------------------------------------+
```

```sql
CREATE DATABASE yb_demo;
\c yb_demo;

\i share/schema.sql
\i share/products.sql
\i share/users.sql
\i share/orders.sql
\i share/reviews.sql

yb_demo=# SELECT users.id, users.name, users.email, orders.id, orders.total FROM orders INNER JOIN users ON orders.user_id=users.id LIMIT 10;
  id  |        name         |             email             |  id   |       total
------+---------------------+-------------------------------+-------+--------------------
  616 | Rex Thiel           | rex-thiel@gmail.com           |  4443 | 101.41460206027735
 2289 | Alanis Kovacek      | alanis.kovacek@yahoo.com      | 17195 |  71.84993665642064
   37 | Jaleel Collins      | jaleel.collins@gmail.com      |   212 |  38.88214510228093
 2164 | Cordia Farrell      | cordia.farrell@gmail.com      | 16223 | 37.748943028753146
 1528 | Donny Murazik       | murazik-donny@hotmail.com     | 11546 | 52.308227375158594
 1389 | Henriette O'Connell | connell-o-henriette@yahoo.com | 10551 |  69.31176446876957
 2408 | Blake Jast          | jast.blake@hotmail.com        | 18149 | 150.78892588707666
 1201 | Kaycee Keebler      | kaycee-keebler@gmail.com      |  8937 | 48.344095586670846
 1421 | Cornell Cartwright  | cornell-cartwright@gmail.com  | 10772 |   191.867670306882
  523 | Deonte Hoeger       | hoeger.deonte@hotmail.com     |  3710 |  71.40107541698262
(10 rows)

```

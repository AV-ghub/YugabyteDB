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
| YugabyteDB UI       : http://10.19.57.138:15433                                                           |
| JDBC                : jdbc:postgresql://10.19.57.138:5433/yugabyte?user=yugabyte&password=yugabyte        |
| YSQL                : bin/ysqlsh -h 10.19.57.138  -U yugabyte -d yugabyte                                 |
| YCQL                : bin/ycqlsh 10.19.57.138 9042 -u cassandra                                           |
| Data Dir            : /root/var/data                                                                      |
| Log Dir             : /root/var/logs                                                                      |
| Universe UUID       : 48cc99d2-4bdd-42c0-a977-1d45665c0b95                                                |
+-----------------------------------------------------------------------------------------------------------+
```

# sysbench 體驗

sysbench 是一個壓力測試工具，可以對資料庫進行壓測，也可以對作業系統進行壓測，是我在前公司用來對mysql，以及tiDB(號稱是進化過的mysql)用來測試資料庫性能的工具，所以這次的體驗算是一個回味。

## sysbench 安裝

- 因為我電腦是windows，所以這邊使用docker的方式來使用sysbench

```cmd
docker pull severalnines/sysbench
```

## 準備 mysql

- 這邊也是透過docker建一個mysql

```cmd
docker run --name mysql-test -e MYSQL_ROOT_PASSWORD=123456 -e MYSQL_DATABASE=test -p 3306:3306 -d mysql:8.0 --default-authentication-plugin=mysql_native_password
```

## 使用 sysbench

- 這邊就透過docker run的方式來執行sysbench，注意host要用wsl網卡的ip位址，因為mysql是由docker建起來的

1. 準備測試資料 

```powershell
docker run --rm --name sysbench --network host severalnines/sysbench `
  sysbench /usr/share/sysbench/oltp_read_write.lua `
  --mysql-host=172.29.16.1 `
  --mysql-port=3306 `
  --mysql-user=root `
  --mysql-password=123456 `
  --mysql-db=test `
  --tables=10 `
  --table-size=10000 `
  prepare
```

2. 壓力測試

```powershell
docker run --rm --name sysbench --network host severalnines/sysbench `
  sysbench /usr/share/sysbench/oltp_read_write.lua `
  --mysql-host=172.29.16.1 `
  --mysql-port=3306 `
  --mysql-user=root `
  --mysql-password=123456 `
  --mysql-db=test `
  --tables=10 `
  --table-size=10000 `
  --threads=4 --time=30 run
```

3. 清理

```powershell
docker run --rm --name sysbench --network host severalnines/sysbench `
  sysbench /usr/share/sysbench/oltp_read_write.lua `
  --mysql-host=172.29.16.1 --mysql-port=3306 `
  --mysql-user=root --mysql-password=123456 `
  --mysql-db=test cleanup
```

## 測試 postgresql

- 最近發現postgresql很火熱，所以就想說拿來跟mysql做比較才有意思

```cmd
docker run -d --name pg-test `
  -e POSTGRES_PASSWORD=pgpass `
  -e POSTGRES_DB=test `
  -e POSTGRES_INITDB_ARGS="--auth-host=md5 --auth-local=md5" `
  -p 5432:5432 `
  postgres:16 -c password_encryption=md5

# 建 sbuser/sbpass，並授權能在 test DB 操作
docker exec -it pg-test psql -U postgres -d test -c "CREATE USER sbuser WITH PASSWORD 'sbpass'; GRANT ALL PRIVILEGES ON DATABASE test TO sbuser; GRANT ALL ON SCHEMA public TO sbuser;"

```

1. 準備測試資料 

```powershell
docker run --rm --name sysbench --network host severalnines/sysbench `
  sysbench /usr/share/sysbench/oltp_read_write.lua `
  --db-driver=pgsql `
  --pgsql-host=172.29.16.1 --pgsql-port=5432 `
  --pgsql-user=postgres --pgsql-password=pgpass `
  --pgsql-db=test `
  --tables=10 --table-size=10000 `
  prepare
```

2. 壓力測試

```powershell
docker run --rm --name sysbench --network host severalnines/sysbench `
  sysbench /usr/share/sysbench/oltp_read_write.lua `
  --db-driver=pgsql `
  --pgsql-host=172.29.16.1 --pgsql-port=5432 `
  --pgsql-user=postgres --pgsql-password=pgpass `
  --pgsql-db=test `
  --tables=10 --table-size=10000 `
  --threads=4 --time=30 run
```

3. 清理

```powershell
docker run --rm --name sysbench --network host severalnines/sysbench `
  sysbench /usr/share/sysbench/oltp_read_write.lua `
  --db-driver=pgsql `
  --pgsql-host=172.29.16.1 --pgsql-port=5432 `
  --pgsql-user=postgres --pgsql-password=pgpass `
  --pgsql-db=test cleanup
```

## 比較結果

- mysql的結果

```txt
SQL statistics:
    queries performed:
        read:                            24682
        write:                           7052
        other:                           3526
        total:                           35260
    transactions:                        1763   (58.65 per sec.)
    queries:                             35260  (1173.04 per sec.)
    ignored errors:                      0      (0.00 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          30.0572s
    total number of events:              1763

Latency (ms):
         min:                                   57.83
         avg:                                   68.14
         max:                                  111.57
         95th percentile:                       78.60
         sum:                               120122.44

Threads fairness:
    events (avg/stddev):           440.7500/0.43
    execution time (avg/stddev):   30.0306/0.02
```

- postgresql的結果

```txt
SQL statistics:
    queries performed:
        read:                            29218
        write:                           8347
        other:                           4175
        total:                           41740
    transactions:                        2087   (69.46 per sec.)
    queries:                             41740  (1389.20 per sec.)
    ignored errors:                      0      (0.00 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          30.0453s
    total number of events:              2087

Latency (ms):
         min:                                   50.33
         avg:                                   57.54
         max:                                   93.65
         95th percentile:                       68.05
         sum:                               120095.98

Threads fairness:
    events (avg/stddev):           521.7500/0.43
    execution time (avg/stddev):   30.0240/0.02
```

## 結論

從測試結果來看，postgresql 16似乎比mysql 8.0還優秀一些，但這邊就做一個很基本的比較，有興趣的可以再自行衍生，另外整個測試過程透過sysbench的簡化就能快速又流暢，體驗還挺不錯的，美中不足的是sysbench再連postgresql或是mysql的時候，都有遇到一些版本議題，會需要另外處理一下，不過還算瑕不掩瑜。

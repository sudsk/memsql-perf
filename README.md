# MemSQL performance evaluation

## Infrastructure
2 VMs provisioned on Google Cloud Platform in europe-west2-a zone. 

- VM#1 
  - Machine type - custom (8 vCPUs, 20 GB memory)
  - CPU platform - Intel Broadwell
  - 50GB SSD persistent disk 
  - source image - debian-9-stretch-v20180307
  - MemSQL - Master aggregator and a leaf node

- VM#2 
  - Machine type - custom (8 vCPUs, 20 GB memory)
  - CPU platform - Intel Broadwell
  - 90GB SSD persistent disk 
  - source image - debian-9-stretch-v20180307
  - MemSQL - Two leaf nodes

## Install MemSQL on both VMs
Download the software
```
cd /tmp
wget http://download.memsql.com/memsql-ops-6.0.10/memsql-ops-6.0.10.tar.gz
tar zxvf memsql-ops-6.0.10.tar.gz
```
Run the installer
```
cd memsql-ops-6.0.10
sudo ./install.sh
```

Make the second VM follow the first one. At this point MemSQL Ops Web UI should be able to see both VMs as part of cluster.
```
memsql-ops follow -h <master-vm-ip>
```

From the MemSQL Ops Web UI, add MemSQL nodes with one master and three leaf node roles across the two VMs.

For starting/stopping the cluster use the below commands
```
memsql-ops memsql-stop --all
memsql-ops memsql-start --all
```

## Schema Design

- Create database

```
create database mymemsqldb;
show databases;
use mymemsqldb;
```

- Create payment transactions table

```
CREATE TABLE `transactions` (
  `OUTLET_KEY` mediumint(9) DEFAULT NULL,
  `TRADING_DATE` date DEFAULT NULL,
  `PROCESSING_DATE` date DEFAULT NULL,
  `TRANSACTION_DATETIME` datetime DEFAULT NULL,
  `TRANSACTION_DATE` date DEFAULT NULL,
  `CARD_NUMBER_LENGTH` tinyint(4) DEFAULT NULL,
  `CARD_NUMBER_LEFT` mediumint(9) DEFAULT NULL,
  `CARD_NUMBER_RIGHT` smallint(6) DEFAULT NULL,
  `CARD_EXPIRY_DATE` varchar(5) CHARACTER SET utf8 COLLATE utf8_general_ci DEFAULT NULL,
  `CARD_ISSUER_NUMBER` varchar(2) CHARACTER SET utf8 COLLATE utf8_general_ci DEFAULT NULL,
  `CARD_START_DATE` varchar(5) CHARACTER SET utf8 COLLATE utf8_general_ci DEFAULT NULL,
  `TRANSACTION_TYPE_KEY` tinyint(4) DEFAULT NULL,
  `TRANSACTION_SOURCE_KEY` tinyint(4) DEFAULT NULL,
  `SETTLEMENT_AMOUNT` float DEFAULT NULL,
  `SETTLEMENT_CURRENCY_KEY` varchar(3) CHARACTER SET utf8 COLLATE utf8_general_ci DEFAULT NULL,
  `TRANSACTION_AMOUNT` float DEFAULT NULL,
  `TRANSACTION_CURRENCY_KEY` varchar(3) CHARACTER SET utf8 COLLATE utf8_general_ci DEFAULT NULL,
  `ACQUIRED_PROCESSED_KEY` smallint(6) DEFAULT NULL,
  `CARD_PRODUCT_KEY` smallint(6) DEFAULT NULL,
  `CARD_PRODUCT_TYPE_KEY` smallint(6) DEFAULT NULL,
  `CARD_SCHEME_KEY` smallint(6) DEFAULT NULL,
  `SEQUENCE_NO` varchar(10) CHARACTER SET utf8 COLLATE utf8_general_ci DEFAULT NULL,
  `AUTH_CODE` varchar(20) CHARACTER SET utf8 COLLATE utf8_general_ci DEFAULT NULL,
  `AUTH_METHOD_KEY` smallint(6) DEFAULT NULL,
  `RECEIPT_NUMBER` varchar(20) CHARACTER SET utf8 COLLATE utf8_general_ci DEFAULT NULL,
  `CASH_AMOUNT` float DEFAULT NULL,
  `ORIGINATORS_TRANSACTION_REF` varchar(100) CHARACTER SET utf8 COLLATE utf8_general_ci DEFAULT NULL,
  `TICKET_NUMBER` varchar(20) CHARACTER SET utf8 COLLATE utf8_general_ci DEFAULT NULL,
  KEY `TRANSACTION_DATETIME` (`TRANSACTION_DATETIME`,`CARD_SCHEME_KEY`,`TRANSACTION_TYPE_KEY`,`TRANSACTION_CURRENCY_KEY`,`PROCESSING_DATE`) /*!90619 USING CLUSTERED COLUMNSTORE */,
  /*!90618 SHARD */ KEY `OUTLET_KEY` (`OUTLET_KEY`)
) /*!90621 AUTOSTATS_ENABLED=TRUE */
```

- Used Oracle SQL query to generate data and exported to csv 

```
SELECT mod(column_value,15013) as OUTLET_KEY,
trunc(current_date - interval '1' second * column_value) as TRADING_DATE,
trunc(current_date - interval '1' second * column_value + interval '1' day) as PROCESSING_DATE,
(current_date - interval '1' second * column_value) as TRANSACTION_DATETIME,
trunc(current_date - interval '1' second * column_value) as TRANSACTION_DATE,
16 as CARD_NUMBER_LENGTH,
trunc(dbms_random.value() * (999999-100000) + 100000) as CARD_NUMBER_LEFT,
trunc(DBMS_RANDOM.VALUE() * (9999-1000) + 1000) as CARDT_NUMBER_RIGHT,
'12/19' as card_expiry_date,
'01' as card_issuer_number,
'01/17' as card_start_date,
mod(column_value,4) transaction_type_key, 
mod(column_value,16) transaction_source_key,
0 as settlement_amount,
'GBP' as settlement_currency_key,
round(dbms_random.value()*200,2) as transaction_amount,
'GBP' as transaction_currency_key,
mod(column_value,2) as acquired_processed_key,
mod(mod(column_value,29),11) as CARD_PRODUCT_KEY,
mod(column_value,29) as CARD_PRODUCT_TYPE_KEY,
mod(mod(column_value,29),3) as CARD_SCHEME_KEY,	
mod(column_value,11) as sequence_no,
lpad(round(dbms_random.value()*4000),4,1) as auth_code,
mod(column_value,11) as auth_method_key,
round(dbms_random.value()*4000) as receipt_number,
0 as cash_amount,
'W0400000S8157200942'||round(mod(column_value,15013)) as originators_transaction_ref,
round(dbms_random.value()*50000) as ticket_number
FROM table(swe_datatools.generate_series(1,100000000));
```

- Load 2.3B rows into transactions table.

```
memsql> load data infile './export11.csv' into table mymemsqldb.transactions columns terminated by ',' optionally enclosed by '"' lines terminated by '\r\n'  ignore 1 lines (OUTLET_KEY,@TRADING_DATE,@PROCESSING_DATE,@TRANSACTION_DATETIME,@TRANSACTION_DATE,CARD_NUMBER_LENGTH,CARD_NUMBER_LEFT,CARD_NUMBER_RIGHT,CARD_EXPIRY_DATE,CARD_ISSUER_NUMBER,CARD_START_DATE,TRANSACTION_TYPE_KEY,TRANSACTION_SOURCE_KEY,SETTLEMENT_AMOUNT,SETTLEMENT_CURRENCY_KEY,TRANSACTION_AMOUNT,TRANSACTION_CURRENCY_KEY,ACQUIRED_PROCESSED_KEY,CARD_PRODUCT_KEY,CARD_PRODUCT_TYPE_KEY,CARD_SCHEME_KEY,SEQUENCE_NO,AUTH_CODE,AUTH_METHOD_KEY,RECEIPT_NUMBER,CASH_AMOUNT,ORIGINATORS_TRANSACTION_REF,TICKET_NUMBER) set TRADING_DATE = STR_TO_DATE(@TRADING_DATE, '%d-%b-%y'), PROCESSING_DATE = STR_TO_DATE(@PROCESSING_DATE, '%d-%b-%y'), TRANSACTION_DATETIME = STR_TO_DATE(@TRANSACTION_DATETIME, '%d-%b-%y %H.%i.%S'), TRANSACTION_DATE = STR_TO_DATE(@TRANSACTION_DATE, '%d-%b-%y');
```

```
-- Used two strategies of adding more outlets and more data (5 mins interval) for existing outlets.  
memsql> insert into mymemsqldb.transactions 
    -> select OUTLET_KEY + 64000,TRADING_DATE,PROCESSING_DATE,ADDDATE(TRANSACTION_DATETIME, INTERVAL 5 MINUTE),DATE(ADDDATE(TRANSACTION_DATETIME, INTERVAL 5 MINUTE)),
    -> CARD_NUMBER_LENGTH,CARD_NUMBER_LEFT,CARD_NUMBER_RIGHT,CARD_EXPIRY_DATE,CARD_ISSUER_NUMBER,
    -> CARD_START_DATE,TRANSACTION_TYPE_KEY,TRANSACTION_SOURCE_KEY,SETTLEMENT_AMOUNT,SETTLEMENT_CURRENCY_KEY,
    -> TRANSACTION_AMOUNT,TRANSACTION_CURRENCY_KEY,ACQUIRED_PROCESSED_KEY,CARD_PRODUCT_KEY,
    -> CARD_PRODUCT_TYPE_KEY,CARD_SCHEME_KEY,SEQUENCE_NO,AUTH_CODE,AUTH_METHOD_KEY,RECEIPT_NUMBER,
    -> CASH_AMOUNT,ORIGINATORS_TRANSACTION_REF,TICKET_NUMBER
    -> from mymemsqldb.transactions a;
```

- Create outlet table which stores user-outlet relationship i.e. a user has data role access to which outlets.

```
CREATE TABLE `outlet` (
  `OUTLET_KEY` mediumint(9) DEFAULT NULL,
  `USER_KEY` mediumint(9) DEFAULT NULL,
  KEY `USER_KEY` (`USER_KEY`) /*!90619 USING CLUSTERED COLUMNSTORE */,
  /*!90618 SHARD */ KEY `OUTLET_KEY` (`OUTLET_KEY`)
) /*!90621 AUTOSTATS_ENABLED=TRUE */
```

- Evenly distribute outlets numbering 120k into 20 users (~6k outlets/user) - Large corporates users

```
insert into mymemsqldb.outlet 
select a.outlet_key, mod(a.outlet_key,20) as user_key
from (select distinct outlet_key from mymemsqldb.transactions) a;
```

- Add two users (20 & 21) with only a few outlets - SME users

```
insert into mymemsqldb.outlet (outlet_key, user_key) values (0,20); 
insert into mymemsqldb.outlet (outlet_key, user_key) values (1,21);
insert into mymemsqldb.outlet (outlet_key, user_key) values (2,21);
insert into mymemsqldb.outlet (outlet_key, user_key) values (3,21);
insert into mymemsqldb.outlet (outlet_key, user_key) values (4,21);
insert into mymemsqldb.outlet (outlet_key, user_key) values (5,21);
```

## Memsql Ops screens

- Cluster
![cluster](images/cluster.PNG)

- Status
![status](images/status.PNG)

- Transactions table
![Transactions](images/transactions.PNG)

- Outlet table
![Outlet](images/outlet.PNG)

## Performance tests

- Query for a corporate user (~6k outlets, 12,413,120 records with below filters)
```
select a.*
from mymemsqldb.transactions a , mymemsqldb.outlet b
where a.outlet_key = b.OUTLET_KEY 
and transaction_datetime > '2017-10-03' 
and transaction_datetime < '2018-04-03' 
and b.USER_KEY = 0
order by transaction_datetime asc 
limit 10 offset 100;
```
Response - 300ms

- Query for a SME user (1 outlet, 2064 records with below filters)
```
select a.*
from mymemsqldb.transactions a , mymemsqldb.outlet b
where a.outlet_key = b.OUTLET_KEY 
and transaction_datetime > '2017-10-03' 
and transaction_datetime < '2018-04-03' 
and b.USER_KEY = 20
order by transaction_datetime asc 
limit 10 offset 100;
```
Response - 700ms

## Questions

1. I'm getting faster response time for a user having ~6k outlet access compared with a user with 1-5 outlet access. This was observed even after repeatedly running the queries. The user with 1 outlet is accessing very small dataset. 

2. Though the tables take about 24GB disk space, the total disk space usage is ~80GB across the cluster as shown on cluster screenshot. Another thing - the increase in disk space is observed during data load and on restarting the cluster it seems compaction happens and the disk space shrinks. (I would assume the ~80GB will likely come down next time I restart the cluster). 

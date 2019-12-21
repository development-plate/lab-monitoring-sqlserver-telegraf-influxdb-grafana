# Monitoring SQL Server with Telegraf, InfluxDB and Grafana
## Foreword
In this workshop a monitoring on a SQL Server 2019 with Telegraf, InfluxDB and Grafana will be realized. The whole thing runs with Docker Containers.

## Requirement
Computer with pre-installed docker. 

## SECTION MS SQL Server 2019 as Docker Container
In the first step we start a SQL Server 2019 as Docker container.

sudo mkdir /var/opt/mssql && sudo chgrp -R 0 /var/opt/mssql && sudo chmod -R g=u /var/opt/mssql

sudo docker run -e "ACCEPT_EULA=Y" -e "SA_PASSWORD=M0nit0ring" \\
   -p 1433:1433 --name monitoring-mssql \\
   -v /var/opt/mssql:/var/opt/mssql \\
   -d mcr.microsoft.com/mssql/server:2019-GA-ubuntu-16.04

## create telegraf user and give him the rights to read server state
Connect with any DB tool into the MS Sqlserver container (Login: SA, Password: M0nit0ring) and execute sqlserver-dbscript.sql on it.

## SECTION InfluxDB as Docker Container
In the second step we start InfluxDB as Docker container.

sudo mkdir /var/opt/influxdb && sudo chgrp -R 0 /var/opt/influxdb && sudo chmod -R g=u /var/opt/influxdb

sudo docker run -p 8086:8086 -p 8083:8083 -p 2003:2003 --name monitoring-influxdb \\
      -e INFLUXDB_DB=db0 -e INFLUXDB_ADMIN_ENABLED=true \\
      -e INFLUXDB_ADMIN_USER=admin -e INFLUXDB_ADMIN_PASSWORD=M0nit0ring \\
      -e INFLUXDB_USER=monitoring -e INFLUXDB_USER_PASSWORD=M0nit0ring \\
      -v /var/opt/influxdb:/var/lib/influxdb \\
      -d influxdb:1.7.9 

### create database for test

curl -X POST -G http://localhost:8086/query --data-urlencode "q=CREATE DATABASE mydb"

### show database and you see mydb in response

curl -G http://localhost:8086/query --data-urlencode "q=SHOW DATABASES"

# SECTION Telegraf as Docker Container

sudo mkdir /etc/telegraf && sudo chgrp -R 0 /etc/telegraf && sudo chmod -R g=u /etc/telegraf

## create telegraf file

sudo touch /etc/telegraf/telegraf.conf
sudo nano /etc/telegraf/telegraf.conf

copy and paste content of telegraf.conf in this blank file

## run telegraf as Docker Container

sudo docker run -d --name=monitoring-telegraf \\
      --network="host" \\
      -v /etc/telegraf/telegraf.conf:/etc/telegraf/telegraf.conf:ro \\
      -d telegraf

## check if database telegraf exists

curl -G http://localhost:8086/query --data-urlencode "q=SHOW DATABASES"

# SECTION Grafana

sudo docker volume create grafana-storage
sudo docker run --network="host" --name=monitoring-grafana -d -p 3000:3000 -v grafana-storage:/var/lib/grafana grafana/grafana

visit http://localhost:3000 in your browser. At first start you must log into with admin + admin and give at next step a new password

add data source 
select InfluxDB

name: mssql-ds
URL: http://localhost:8086
Database: monitoring-mssql
user: monitoring
Password: M0nit0ring


## import Dashboard SQL Server
we use this dashboard: https://grafana.com/grafana/dashboards/409

goto http://localhost:3000/dashboard/import

input id: 409

click on load

select datasource: mssql-ds

Finally, you'll see a dashboard like this one:
![Grafana MS SQLServer Dashboard](https://github.com/development-plate/lab-monitoring-sqlserver-telegraf-influxdb-grafana/blob/master/SQLServer_Monitoring.png)

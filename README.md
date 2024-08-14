- mysql 一主两从数据同步配置

| 名称 | 端口 | IP | 说明 |
| :-: | :-: | :-: | :-: |
| mysql-master | 3306 | 172.16.0.101 | 主 |
| mysql-slave1 | 3307 | 172.16.0.102 | 从 |
| mysql-slave2 | 3308 | 172.16.0.103 | 从 |

- 请先删除{master,slave01,slave02}/data/ 文件下的所有数据(你需要保持一个空的挂载目录)。

1. 启动服务
```sh
docker-compose -p master-slave up -d
```

1. 进入mysql-master容器，创建远程账户。
```sh
docker exec -it mysql-master bash
```

1. 连接mysql
```sh
mysql -uroot -P3306 -p123456
```

1. 创建账户
```sql
CREATE USER 'ic'@'%' IDENTIFIED WITH mysql_native_password BY 'Root@123456';
```

1. 分配权限
```sql
GRANT SELECT, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'ic'@'%';
```

1. 刷新权限
```sql
FLUSH PRIVILEGES;
```

1. 查看二进制文件
```sql
show master status;
    -- file: mysql-bin.000003
    -- position: 889
```

1. 进入从库容器
```sh
docker exec -it mysql-slave1 bash
```

1. 连接mysql
```sh
mysql -uroot -P3307 -p123456
```

1.  设置主从配置
```sql
CHANGE MASTER TO MASTER_HOST='mysql-master', MASTER_USER='ic', \
MASTER_PASSWORD='Root@123456', MASTER_PORT=3306, MASTER_LOG_FILE='mysql-bin.000003', \
MASTER_LOG_POS=889;
```

1.  开启同步
```sql
start slave;
```

1.  查看主从同步状态
```sql
show slave status \G;
    -- Slave_IO_Running: Yes
    -- Slave_SQL_Running: Yes
```

1.  进入从库容器
```sh
docker exec -it mysql-slave2 bash
```

1.  连接mysql
```sh
mysql -uroot -P3308 -p123456
```

1.  设置主从配置
```sql
CHANGE MASTER TO MASTER_HOST='mysql-master', MASTER_USER='ic', \
MASTER_PASSWORD='Root@123456', MASTER_PORT=3306, MASTER_LOG_FILE='mysql-bin.000003', \
MASTER_LOG_POS=889;
```

1.  开启同步
```sql
start slave;
```

1.  查看主从同步状态
```sql
show slave status \G;
    -- Slave_IO_Running: Yes
    -- Slave_SQL_Running: Yes
```

1.  测试，在master添加如下代码
```sql
create database db01;
use db01;
create table tb_user(
	id int(11) primary key not null auto_increment,
	name varchar(50) not null,
	sex varchar(1)
)engine=innodb default charset=utf8mb4;
insert into tb_user(id,name,sex) values(null,'Tom', '1'),(null,'Trigger','0'),(null,'Dawn','1');
```

1.  在slave01和slave02中分别验证。

2.  查看master数据有那些slave
```sql
select * from information_schema.processlist as p where p.command = 'Binlog Dump'; 
```
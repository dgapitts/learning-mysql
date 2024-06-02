## Getting started with docker mysql

## Download docker mysql-server:latest

```
docker pull mysql/mysql-server:latest
```

## Running my_sql container
```
docker run --name='my_sql' -d -p 3306:3306 mysql/mysql-server
```

## docker logs - get GENERATED ROOT PASSWORD
```
docker logs my_sql 2>&1 | grep 'GENERATED ROOT PASSWORD'
[Entrypoint] GENERATED ROOT PASSWORD: p?.*r#^V43.imv:1T2tR:4i7aD4S7tiv
```

## docker mysql connection to reset password
```
docker exec -it my_sql mysql -uroot -p
ALTER USER 'root'@'localhost' IDENTIFIED BY '...';
```


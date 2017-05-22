# Update root password in MySQL 5.7

```bash
$ mysql -u root

root:(none)> use mysql;

root:mysql> update user set password=password('abc') where user='root';
root:mysql> set password = password('abc');
root:mysql> flush privileges;
root:mysql> quit;
```

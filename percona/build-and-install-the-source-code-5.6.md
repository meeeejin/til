# Build and install the source code (5.6)

Building Percona Server 5.6 for MySQL from the source code enables you to customize build parameters, compiler optimizations, and installation location.

## Build and install

Run cmake to configure the build.

```bash
$ cmake -DCMAKE_BUILD_TYPE=RelWithDebInfo -DBUILD_CONFIG=mysql_release -DFEATURE_SET=community -DWITH_EMBEDDED_SERVER=OFF -DCMAKE_INSTALL_PREFIX=/path/to/dir
```

Then compile and install the source code.
(8: # of cores in your machine)

```bash
$ make -j8 install
```

Initialize tasks that must be performed before the Percona Server for MySQL, mysqld, is ready to use.

> If `mysql_install_db` is not executable, run `sudo chmod +x mysql_install_db`

- `--datadir` : the path to the Percona Server data directory
- `--basedir` : the path to the Percona Server installation directory.

```bash
$ ./scripts/mysql_install_db --user=mysql --datadir=/path/to/datadir --basedir=/path/to/basedir
```

Reset the root password.

```bash
$ ./bin/mysqld_safe --skip-grant-tables

$ ./bin/mysql -uroot

root:(none)> use mysql;

root:mysql> update user set authentication_string=password('abc') where user='root';
root:mysql> flush privileges;
root:mysql> quit;

$ ./bin/mysql -uroot

root:mysql> set password = password('abc');
root:mysql> quit;
```

Shut down and restart the server.

```bash
$ ./bin/mysqladmin -uroot -pabc shutdown

$ ./bin/mysqld_safe
```

# MySQL connection error

I modified three files to fix an error which *max_connection* is fixed to 214.

**/etc/pam.d/common-session**

```bash
session required    pam_limits.so
session	optional    pam_ck_connector.so nox11
```

**/etc/security/limits.conf**

```bash
*   hard    nofile  24000
*   soft    nofile  32000
```

**/etc/my.cnf**

```bash
...
[mysqld]
max_connections = 1024
wait_timeout = 30
open_files_limit = 24000
...
```
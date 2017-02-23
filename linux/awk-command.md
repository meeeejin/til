# AWK command

Print average of 6th column:

```bash
$ cat file | awk '{ sum += $6; n++ } END { if (n > 0) print sum / n; }'
```

Print sum of 6th column:

```bash
$ cat file | awk '{ sum += $6 } END { print sum }'
```

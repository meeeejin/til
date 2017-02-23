# Extracts certain rows from a file

Print lines which does no contain 'n':

```bash
$ sed -n '/n/!p' file
```

Print lines which contain 'a' or 'b':

```bash
$ sed -n '/[ab]/p' file
```

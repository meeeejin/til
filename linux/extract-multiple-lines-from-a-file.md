# Extract multiple lines from a file

## Using sed

Print even-numbered lines from filename:

```bash
sed -n 2~2p filename
```

Print odd-numbered lines from filename:

```bash
sed -n 1~2p filename
```

Print lines with remainder 1 when dividing by 9 (n % 9 == 1) from filename:

```bash
sed -n 1~9p filename
```

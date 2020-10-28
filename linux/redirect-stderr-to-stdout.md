# Redirect stderr to stdout

```bash
./runbench.sh |& tee -a result.txt
```

- `|&` is an abbreviation for `2>&1`
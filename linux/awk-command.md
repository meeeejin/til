# AWK command

Print the average value of 6th column:

```bash
$ awk '{ sum += $6; n++ } END { if (n > 0) print sum / n; }' input
```

Print the sum value of 6th column:

```bash
$ awk '{ sum += $6 } END { print sum }' input
```

Print max, min, and the average value of 6th column:

```bash
$ awk 'NR == 1 { min = $6; max = $6 } { sum += $6; if ($6 > max) { max = $6 }; if ($6 < min) { min = $6 }; } END { print max, min, sum / NR }' input 
```
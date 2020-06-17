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
$ awk 'NR == 1 { min = $6; max = 0 } { sum += $6; if ($6 > max) { max = $6 }; if ($6 < min) { min = $6 }; } END { print min, max, sum / NR }' input 
```

Print max, min, and the average value of 6th column satisfying `if ($2 > x)` condition:

```bash
$ awk 'NR == 1 { min = $6; max = 0; n = 0; } { if ($2 > x) { sum += $6; n++; if ($6 > max) { max = $6 }; if ($6 < min) { min = $6 }; } } END { print min, max, sum / n }' input
```

Print only the `6x+1` line:

```bash
$ awk 'NR % 6 == 1' input
```
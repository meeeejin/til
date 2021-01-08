# AWK command

1. Print the average value of 6th column:

```bash
$ awk '{ sum += $6; n++ } END { if (n > 0) print sum / n; }' input
```

2. Print the sum value of 6th column:

```bash
$ awk '{ sum += $6 } END { print sum }' input
```

3. Print max, min, and the average value of 6th column:

```bash
$ awk 'NR == 1 { min = $6; max = 0 } { sum += $6; if ($6 > max) { max = $6 }; if ($6 < min) { min = $6 }; } END { print min, max, sum / NR }' input 
```

4. Print max, min, and the average value of 6th column satisfying `if ($2 > x)` condition:

```bash
$ awk 'NR == 1 { min = $6; max = 0; n = 0; } { if ($2 > x) { sum += $6; n++; if ($6 > max) { max = $6 }; if ($6 < min) { min = $6 }; } } END { print min, max, sum / n }' input
```

5. Print only the `6x+1` line:

```bash
$ awk 'NR % 6 == 1' input
```

6. Calculate the difference between consecutive data in a column from a file

```bash
$ cat input
214
1241
43423

$ awk 'NR==1 { p = $1; next } { { if ($1 > 0) { print $1-p; } else { print "" }} p=$1 } END {}' input

1027
42182
```
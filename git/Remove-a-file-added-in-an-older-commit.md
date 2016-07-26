# Remove a file added in an older commit

We can use a tool called [BFG Repo-Cleaner](https://rtyley.github.io/bfg-repo-cleaner/).

```bash
bfg --strip-blobs-bigger-than 50M
```

or

```bash
java -jar bfg.jar --strip-blobs-bigger-than 50M
```

# Modify previous commit messages

I learned from this [blog](https://git-scm.com/book/ko/v1/Git-%EB%8F%84%EA%B5%AC-%ED%9E%88%EC%8A%A4%ED%86%A0%EB%A6%AC-%EB%8B%A8%EC%9E%A5%ED%95%98%EA%B8%B0).

## Modify the last commit message

```bash
git commit --amend
```

## Modify previous commit messages

Rebasing is the process of moving a branch to a new base commit. 

```bash
git rebase -i HEAD~3
```

The result is as follows:

```bash
pick f7f3f6d changed my name a bit
pick 310154e updated README formatting and added blame
pick a5f4a0d added cat-file
```

We can see the last 3 commits. Find the commit message to modify and change `pick` to `edit`

```bash
edit f7f3f6d changed my name a bit
pick 310154e updated README formatting and added blame
pick a5f4a0d added cat-file
```

Save and quit. Then run `commit` command.

```bash
git commit --all --amend
```

or

```bash
git commit --amend
```

After committing the fixed version, do:

```bash
git rebase --continue
```

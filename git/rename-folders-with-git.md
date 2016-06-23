# Rename folders with git

I learned from this [blog](https://www.patrick-wied.at/blog/rename-files-and-folders-with-git).

## How to use

```bash
git mv oldfolder newfolder
```

If you want to overwrite a new folder that is already in your repo, use -force option.

```bash
git mv -f oldfolder newfolder
```

## Renaming folder name on case insentive file systems

```bash
git mv foldername tempname
git mv tempname folderName
```

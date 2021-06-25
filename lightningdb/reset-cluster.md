# How to reset a LTDB cluster

- `~/.bashrc`:

```bash
export FBPATH=$HOME/.flashbase
export PATH=$HOME/.local/bin:$PATH

alias python=python2.7
alias fb=flashbase
```

Run below command to reset cluster **1**:

```bash
$ cfc 1
$ fb restart --reset --cluster --yes
```
# Building PMDK on Ubuntu

Building [PMDK](https://github.com/pmem/pmdk) from the source code.

## Pre-requisites

- autoconf

```bash
$ sudo apt-get install autoconf
```

- pkg-config

```bash
$ sudo apt-get install pkg-config
```

- libndctl-devel (v60.1 or later)

Install the following required packages on the build system.

```bash
$ wget http://launchpadlibrarian.net/385763707/libdaxctl1_61.2-0ubuntu1~18.04.1_amd64.deb
$ sudo dpkg -i libdaxctl1_61.2-0ubuntu1~18.04.1_amd64.deb
```

```bash
$ wget http://launchpadlibrarian.net/385763708/libndctl6_61.2-0ubuntu1~18.04.1_amd64.deb
$ sudo dpkg -i libndctl6_61.2-0ubuntu1~18.04.1_amd64.deb
```

```bash
$ wget http://launchpadlibrarian.net/385763703/libndctl-dev_61.2-0ubuntu1~18.04.1_amd64.deb
$ sudo dpkg -i libndctl-dev_61.2-0ubuntu1~18.04.1_amd64.deb
```

- libdaxctl-devel (v60.1 or later)

```bash
$ wget http://launchpadlibrarian.net/385763701/libdaxctl-dev_61.2-0ubuntu1~18.04.1_amd64.deb
$ sudo dpkg -i libdaxctl-dev_61.2-0ubuntu1~18.04.1_amd64.deb
```

## Build and install

To build from source, clone the repository:

```bash
$ git clone https://github.com/pmem/pmdk
$ cd pmdk
```

For a stable version, checkout a [release tag](https://github.com/pmem/pmdk/releases):

```bash
$ git checkout tags/1.5.1
```

Build the PMDK using the `make` command at the top level:

```bash
$ make
```

(Optional) Installs man pages and libraries in the standard system locations:

```bash
$ sudo make install
```

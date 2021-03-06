- infos = Information about the multifile plugin is in keys below
- infos/author = Thomas <thomas.waser@libelektra.org>
- infos/licence = BSD
- infos/needs =
- infos/provides = resolver storage
- infos/recommends =
- infos/placements = getresolver setresolver commit rollback getstorage setstorage
- infos/status = maintained conformant compatible specific shelltest tested nodep configurable
- infos/metadata =
- infos/description = mounts multiple files within a directory

## Introduction

For some applications it is beneficially to have multiple configuration files.
One way to achieve this, is to mount different files for the application.

In some situations we are not able to specify every configuration file with separate mounts
because new configuration files might be created any time.
Instead we want to include every configuration file matching a given pattern.

The multifile-resolver does so by calling resolver and storage plugins for each file matching a given pattern.


## Plugin Configuration

- `recursive`:
  If present, fts (3) will be used to traverse the directory tree and fnmatch to match `pattern` to the filename.
  If not present, glob (3) with `pattern` will be used on the directory
- `pattern`:
  The pattern to be used to match configuration files.
- `storage`:
  The storage plugin to use.
- `resolver`:
  The resolver plugin to use.
- 'child/<configname>':
  configuration passed to the child backends, `child/` part gets removed.

## Usage

`kdb mount -R multifile -c storage="ini",pattern="*/*.ini",resolver="resolver" /path /mountpoint`

`kdb mount -R multifile -c storage="ini",pattern="*.ini",recursive=,resolver="resolver" /path /mountpoint`

## Examples

```sh
rm -rf ~/.config/multitest || $(exit 0)
mkdir -p ~/.config/multitest || $(exit 0)

cat > ~/.config/multitest/lo.ini << EOF \
[lo]\
addr = 127.0.0.1\
Link encap = Loopback\
EOF

cat > ~/.config/multitest/lan.ini << EOF \
[eth0]\
addr = 192.168.1.216\
Link encap = Ethernet\
EOF

cat > ~/.config/multitest/wlan.ini << EOF \
[wlan0]\
addr = 192.168.1.125\
Link encap = Ethernet\
EOF

sudo kdb mount -R multifile -c storage="ini",pattern="*.ini",resolver="resolver" multitest user/multi

kdb ls user/multi
#> user/multi/lan.ini/eth0
#> user/multi/lan.ini/eth0/Link encap
#> user/multi/lan.ini/eth0/addr
#> user/multi/lo.ini/lo
#> user/multi/lo.ini/lo/Link encap
#> user/multi/lo.ini/lo/addr
#> user/multi/wlan.ini/wlan0
#> user/multi/wlan.ini/wlan0/Link encap
#> user/multi/wlan.ini/wlan0/addr

kdb set user/multi/lan.ini/eth0/addr 10.0.0.2

kdb get user/multi/lan.ini/eth0/addr
#> 10.0.0.2

cat > ~/.config/multitest/test.ini << EOF \
[testsection]\
key = val\
EOF

kdb ls user/multi
#> user/multi/lan.ini/eth0
#> user/multi/lan.ini/eth0/Link encap
#> user/multi/lan.ini/eth0/addr
#> user/multi/lo.ini/lo
#> user/multi/lo.ini/lo/Link encap
#> user/multi/lo.ini/lo/addr
#> user/multi/test.ini/testsection
#> user/multi/test.ini/testsection/key
#> user/multi/wlan.ini/wlan0
#> user/multi/wlan.ini/wlan0/Link encap
#> user/multi/wlan.ini/wlan0/addr

kdb rm -r user/multi/test.ini

stat ~/.config/multifile/test.ini
# RET:1

sudo kdb umount user/multi
```

Recursive:

```sh
rm -rf ~/.config/multitest || $(exit 0)
mkdir -p ~/.config/multitest ~/.config/multitest/a/a1/a12 ~/.config/multitest/a/a2/a22 ~/.config/multitest/b/b1|| $(exit 0)

echo "a1key = a1val" > ~/.config/multitest/a/a1/a12/testa1.file
echo "a2key = a2val" > ~/.config/multitest/a/a2/a22/testa2.file
echo "b1key = b1val" > ~/.config/multitest/b/b1/testb1.file

sudo kdb mount -R multifile -c storage="ini",pattern="*.file",recursive=,resolver="resolver" multitest user/multi

kdb ls user/multi
#> user/multi/a/a1/a12/testa1.file/a1key
#> user/multi/a/a2/a22/testa2.file/a2key
#> user/multi/b/b1/testb1.file/b1key

rm -rf ~/.config/multitest

sudo kdb umount user/multi
```

## Limitations

- You cannot get rid of the configuration file name.

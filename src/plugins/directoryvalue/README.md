- infos = Information about the directoryvalue plugin is in keys below
- infos/author = René Schwaiger <sanssecours@me.com>
- infos/licence = BSD
- infos/needs =
- infos/provides =
- infos/recommends =
- infos/placements = postgetstorage presetstorage
- infos/status = maintained unittest nodep preview
- infos/metadata =
- infos/description = This plugin converts directory values to leaf values

# Directory Value

## Introduction

The Directory Value plugin converts

1. directory (non-leaf) values to leaf values in the “set” direction, and
2. converts them back to directory values in the “get” direction.

A directory value is a key that contains children. For example in the key set:

```
user/grandparent                = Grandparent
user/grandparent/leaf           = Leaf
user/grandparent/parent         = Parent
user/grandparent/parent/child   = Child
user/mother                     = Mother
user/mother/daughter            = Daughter
user/mother/son                 = Son
```

the keys

```
user/grandparent
user/grandparent/parent
user/mother
```

represent directory values, while the keys

```
user/grandparent/leaf
user/grandparent/parent/child
user/mother/daughter
user/mother/son
```

specify leaf values. You can easily check this by drawing the key set in the form of a rooted tree:

```
             user
      /               \
  granparent        mother
  /      |          /    \
leaf  parent    daughter son
         |
       child
```

. The Directory Value plugin converts all directory values to leaf values in the “set” direction by adding new keys with the postfix
`___dirdata`. Theses keys then store the old value of their parent keys

```
user/grandparent                    =
user/grandparent/___dirdata         = Grandparent
user/grandparent/leaf               = Leaf
user/grandparent/parent             =
user/grandparent/parent/___dirdata  = Parent
user/grandparent/parent/child       = Child
user/mother                         =
user/mother/___dirdata              = Mother
user/mother/daughter                = Daugther
user/mother/son                     = Son
```

. You might ask why we need the Directory Value plugin at all. The reason why we created this plugin is that some storage plugins like
[`yajl`](../yajl/README.md) or [`yamlcpp`](../yajl/README.md) are only able to save values inside leaf keys. By loading the Directory Value
plugin these storage plugins are also able to represent directory values properly.

### Array Values

The Directory value plugin also converts array values. Let us take  a look at an example key set:

```
user/array    = Array Value
user/array/#0 = Firt Value
user/array/#1 = Second Value
user/array/#2 = Third Value
```

. The plugin **does not** convert this key set into:

```
user/array            =
user/array/___dirdata = Array Value
user/array/#0         = First Value
user/array/#1         = Second Value
user/array/#2         = Third Value
```

, since then `user/array` **would not be an array** any more. Instead the plugin inserts a new element at index 0 with the **value prefix**
`___dirdata: `:

```
user/array            =
user/array/#0         = ___dirdata: Array Value
user/array/#1         = First Value
user/array/#2         = Second Value
user/array/#3         = Third Value
```

. This way a storage plugin such as YAJL or YAML CPP are still able to store `user/array` as an array.

**Remark:** The plugin only converts array parents that store **string values**.

## Usage

To mount the plugin use the command:

```sh
# Mount plugin to cascading namespace `/examples/directoryvalue`
sudo kdb mount config.file /examples/directoryvalue directoryvalue
```

. To unmount the plugin use the command

```sh
sudo kdb umount /examples/directoryvalue
```

.

## Example

```sh
# Mount plugin to cascading namespace `/examples/directoryvalue`
sudo kdb mount config.file /examples/directoryvalue directoryvalue

# Add a directory value
kdb set /examples/directoryvalue/harold 'Father of SpongeBob SquarePants'
# Add a leaf value
kdb set /examples/directoryvalue/harold/spongebob 'I am ready!'

# Add an array
kdb set /examples/directoryvalue/patrick Star
kdb set /examples/directoryvalue/patrick/#0 'Being grown-up is boring. Besides, I don’t get Jazz.'

# Since the plugin converts values back in the get direction
# a user of the database will not notice any changes.

kdb ls /examples/directoryvalue
#> user/examples/directoryvalue/harold
#> user/examples/directoryvalue/harold/spongebob
#> user/examples/directoryvalue/patrick
#> user/examples/directoryvalue/patrick/#0

kdb get /examples/directoryvalue/harold
#> Father of SpongeBob SquarePants
kdb get /examples/directoryvalue/harold/spongebob
#> I am ready!

kdb get /examples/directoryvalue/patrick
#> Star
kdb get /examples/directoryvalue/patrick/#0
#> Being grown-up is boring. Besides, I don’t get Jazz.

# Retrieve index of last element in array.
# This also works if the storage plugin does not store this index.
kdb getmeta /examples/directoryvalue/patrick array
#> #0

# Undo changes to the key database
kdb rm -r /examples/directoryvalue
sudo kdb umount /examples/directoryvalue
```

# Limitations

**Escaping** is currently **not possible**. If you use the Directory Value plugin you can not

- use the name `___dirdata` as the last part of a normal key,
- use `___dirdata: ` at the beginning of a normal value in the first array element

!

---
author: Markus Raab <elektra@markus-raab.org>
brief: Describes validation capabilities of Elektra.
tags: validation, spec, metadata
review: pending
---

# Validation

## Introduction

Configuration in <abbr title="Free/Libre and Open Source Software">FLOSS</abbr> unfortunately is often stored
completely without validation. Notable exceptions are sudo
(`visudo`), or user accounts (`adduser`) but in most cases
you only get feedback of non-validating configuration when
the application fails to start.

Elektra provides a generic way to validate any configuration
before it is written to disc.

## User Interfaces

Any of Elektra's user interfaces will work with the technique
described in this tutorial, e.g.:

1. `kdb qt-gui`: graphical user interface

2. `kdb editor`: starts up your favorite text editor and
    allows you to edit configuration in any syntax.
    (generalization of `visudo`)

3. `kdb set`: manipulate or add individual configuration
    entries.
    (generalization of `adduser`)

4. Any other tool using Elektra to store configuration
    (e.g. if the application itself has capabilities to
     modify its configuration)


## Metadata Together With Keys

The most direct way to validate keys is

```sh
sudo kdb mount validation.dump user/tutorial/together dump validation
kdb vset user/tutorial/together/test 123 "[1-9][0-9]*" "Not a number"
kdb set user/tutorial/together/test abc
# STDERR: The command kdb.* set failed while accessing the key database .*
# ERROR:  42
# RET:5
```

For all other plugins (except `validation`) the convenience tool `kdb vset`
is missing. Let us see what `kdb vset` actually did:

```sh
kdb lsmeta user/tutorial/together/test
#> check/validation
#> check/validation/match
#> check/validation/message
```

So it only appended some metadata (data describing the data) next to the key,
which we also could do by:

```sh
# Following lines are (except for error conditions) identical to
# kdb vset user/tutorial/together/test 123 "[1-9][0-9]*" "Not a number"
kdb setmeta user/tutorial/together/test check/validation "[1-9][0-9]*"
kdb setmeta user/tutorial/together/test check/validation/match LINE
kdb setmeta user/tutorial/together/test check/validation/message "Not a number"
kdb set user/tutorial/together/test 123
#> Set string to "123"

# Undo modifications
kdb rm -r user/tutorial/together
sudo kdb umount user/tutorial/together
```

The approach is not limited to validation via regular expressions, but
any values-validation plugin can be used, e.g.  [enum](/src/plugins/enum).
For a full list refer to the section "Value Validation" in the
[list of all plugins](/src/plugins/README.md).

Note that it also easy [to write your own (value validation) plugin](/doc/tutorials/plugins.md).

The drawbacks of this approach are:

- The administrator needs to take care that the validation plugins are available
  where needed.
- Some metadata (in this case `check/validation`) needs to be stored next to
  the key which won't work with most configuration files.
  This is the reason why we explicitly used `dump` as storage in `kdb mount`.
- After the key is removed, the validation information is gone, too.
- It only works for the [namespace](/doc/tutorials/namespaces.md) where `vset` was used.
  In the example above we could override the cascading key `/tutorial/together/test`
  with the unvalidated key `dir/tutorial/together/test`.
- You cannot validate structure of which keys must be present or absent.


## Get Started with `spec`

These issues are resolved straightforward by separating the configuration from
its configuration specification (often called schemata in XML or JSON).
The purpose of the [spec namespace](/doc/tutorials/namespaces.md) is to hold the
configuration specification, i.e., the description of how to validate the keys of
all other namespaces.

To make this work, we need a plugin that applies all metadata found in the `spec`-namespace
to all other namespaces. This plugin is called `spec` and needs to be mounted
globally (will be added by default and also with any `kdb global-mount` call).

Before we start, let us make a backup of the current data in the spec and user namespace:

```sh
kdb export spec dump > /tmp/spec.dump
kdb export user dump > /tmp/user.dump
```

We write metadata to the namespace `spec` and the plugin `spec` applies it to every cascading key:

```sh
kdb setmeta spec/tutorial/spec/test hello world
kdb set /tutorial/spec/test value
#> Using name user/tutorial/spec/test
#> Create a new key user/tutorial/spec/test with string "value"
kdb lsmeta spec/tutorial/spec/test | grep -v '^internal/ini'
#> hello
kdb lsmeta /tutorial/spec/test | grep -v '^internal/ini'
#> hello
kdb getmeta /tutorial/spec/test hello
#> world
kdb getmeta user/tutorial/spec/test hello
#> world
```

But it also supports globbing (`_` for any key, `?` for any char, `[]` for character classes):

```sh
kdb setmeta "spec/tutorial/spec/_" new metaval
kdb set /tutorial/spec/test value
#> Using name user/tutorial/spec/test
#> Set string to "value"
kdb lsmeta /tutorial/spec/test | grep -v '^internal/ini'
#> hello
#> new

# Remove keys and metadata from the commands above
kdb rm -r spec/tutorial/spec
kdb rm -r user/tutorial/spec
```

So let us combine this functionality with validation plugins.
So we would specify:

```sh
kdb setmeta spec/tutorial/spec/test check/validation "[1-9][0-9]*"
kdb setmeta spec/tutorial/spec/test check/validation/match LINE
kdb setmeta spec/tutorial/spec/test check/validation/message "Not a number"
```

If we now set a new key with
```sh
kdb set /tutorial/spec/test "not a number"
#> Using name user/tutorial/spec/test
#> Create a new key user/tutorial/spec/test with string "not a number"
```
this key has adopted all metadata from the spec namespace:
```sh
kdb lsmeta /tutorial/spec/test | grep -v '^internal/ini'
#> check/validation
#> check/validation/match
#> check/validation/message
```
Note that this key should not have passed the validation that we defined in the
spec namespace.
Nonetheless we were able to set this key, because the validation plugin was not
active for this key.
On that behalf we have to make sure that the validation plugin is loaded for
this key with:
```
kdb mount tutorial.dump user/tutorial dump validation
```
This [mounts](/doc/tutorials/mount.md) the backend `tutorial.dump` to the mountpoint
**user/tutorial** and activates the validation plugin for the keys below the mountpoint.
The validation plugin now uses the metadata of the keys below **user/tutorial**
to validate values before storing them in `tutorial.dump`.

Although this is better than defining metadata in the same place as the data
itself, we can still do better.
The reason for that is that one of the aims of Elektra is to remove the trouble
of validation and finding the files that hold your configuration from the users.
At the moment a user still has to know which files should hold the configuration
and which plugins must be loaded when he mounts configuration files.

This problem can be addressed by recognizing that the location of the
configuration files and the plugins that must be loaded is part of the
_schema_ of our configuration and therefore should be stored in the spec namespace.

```sh
# Undo modifications
kdb rm -r spec/tutorial/spec/test
kdb rm -r user/tutorial/spec/test
```

### Specfiles
We call the files, that contain a complete schema for configuration
below a specific path in form of metadata, _Specfiles_.

Particularly a _Specfile_ contains metadata that defines
- the mountpoints of paths,
- the plugins to load and
- the behavior of these plugins.

Let us create an example _Specfile_ in the dump format, which supports metadata
(altough the specfile is stored in the dump format, we can still create it using
the human readable [ni format](/src/plugins/ni/README.md) by using `kdb import`):
```sh
sudo kdb mount tutorial.dump spec/tutorial dump
cat << HERE | kdb import spec/tutorial ni  \
[]                                         \
 mountpoint = tutorial.dump                \
 infos/plugins = dump validation           \
                                           \
[/links/_]                                 \
check/validation = https?://.*\..*         \
check/validation/match = LINE              \
check/validation/message = not a valid URL \
description = A link to some website       \
HERE
kdb lsmeta spec/tutorial
#> infos/plugins
#> mountpoint
```
We now have all the metadata that we need to mount and validate the data below
`/tutorial` in one file.

Now we apply this _Specfile_ to the key database to all keys below `tutorial`.
```sh
kdb spec-mount /tutorial
```
This command automatically mounts `/tutorial` to the backend `tutorial.dump` and
loads the validation plugin.

```sh
kdb set /tutorial/links/url "invalid url"
#> Using name user/tutorial/links/url
# STDERR: .*key value failed to validate.*not a valid URL.*
# ERROR:  42
# RET:    5
```

Note that the backend `tutorial.dump` is mounted for all namespaces:
```sh
kdb file user/tutorial
# STDOUT-REGEX: /(home|Users)/.*/\.config/tutorial\.dump
kdb file system/tutorial
# STDOUT-REGEX: .*/tutorial\.dump
kdb file dir/tutorial
# STDOUT-REGEX: /.*/\.dir/tutorial\.dump
```

If you want to set a key for another namespace and do not want to go without validation,
consider that the spec plugin works only when you use cascading keys.
You can work around that by setting the keys with the `-N` option:
```sh
kdb set -N system /tutorial/links/elektra https://www.libelektra.org
#> Using name system/tutorial/links/elektra
#> Create a new key system/tutorial/links/elektra with string "https://www.libelektra.org"
```

## Rejecting Configuration Keys

Up to now we only discussed how to reject keys that have unwanted values.
Sometimes, however, applications require the presence or absence of keys.
There are many ways to do so directly supported by [the spec plugin](/src/plugins/spec).
Another way is to trigger errors with the [error plugin](/src/plugins/error):

```sh
kdb setmeta /tutorial/spec/should_not_be_here trigger/error 10
#> Using keyname spec/tutorial/spec/should_not_be_here
kdb spec-mount /tutorial
kdb set /tutorial/spec/should_not_be_here abc
#> Using name user/tutorial/spec/should_not_be_here
# RET:    5
# STDERR: .*error.*10.*occurred.*
kdb get /tutorial/spec/should_not_be_here
# RET: 1
# STDERR: Did not find key
```

If we want to reject every optional key (and only want to allow required keys)
we can use the plugin `required` as further discussed below.

Before we look further let us undo the modifications to the key database.

```sh
kdb rm -r spec/tutorial
kdb rm -r system/tutorial
kdb umount spec/tutorial
kdb umount /tutorial
kdb rm -rf spec
kdb rm -rf user
kdb import spec dump < /tmp/spec.dump
kdb import user dump < /tmp/user.dump
rm /tmp/spec.dump
rm /tmp/user.dump
```

## Customized Schemas

Sometimes we already have configuration specifications given in some other format
which is more compact and more directed to the needs of an individual application.
We can write a plugin that parses that format and transform the content to key-value
*and* metadata (describing how to validate).

For example, let us assume we have enum validations in the file `schema.txt`:

```
cat > "$PWD/schema.txt" << HERE           \
%: notation TBD ? graph text semi         \
%: tool-support* TBD ? none compiler ide  \
%: applied-to TBD ? none small real-world \
mountpoint file.txt                       \
plugins required                          \
HERE
```

And by convention for keys ending with `*`, multiple values are allowed.
So we want to transform above syntax to:

```
%:notation TBD ? graph text semi
%:tool-support* TBD ? none compiler ide
%:applied-to TBD ? none small real-world
```

Lucky, we already have a plugin which allows us to so:

```
kdb mount "$PWD/schema.txt" spec/tutorial/schema simplespeclang keyword/enum=%:,keyword/assign=TBD
kdb spec-mount /tutorial/schema
```

We configure the plugin `simplespeclang` so that it conforms to our "weird" syntax.
Because in `schema.txt` we have the line `mountpoint file.txt` we can also mount the
schema using `spec-mount`.

Now we have enforced that the 3 configuration options `notation tool-support* applied-to`
need to be present (and no other). For example we can import:

```
kdb import -s validate -c "format=% : %" /tutorial/schema simpleini << HERE \
notation : graph                                                            \
tool-support : ? none                                                       \
applied-to : small                                                          \
HERE                                                                        \
```

Or (afterwards) setting individual values:

```
kdb set /tutorial/schema/applied-to smal # fails, not a valid enum
```

Or (in `visudo` fashion):

```
kdb editor -s validate /tutorial/schema simpleini
```


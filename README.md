# CLBX Specification
CLBX is an extraction container format from Cellebrite, supporting modern
mobile filesystem acquisitions. This light-weight format is designed for
simplicity, interoperability, and storing complete forensic metadata for each
file.

This document describes version 0.3.0 of the format.
This first release specifically addresses iOS extractions, but Android and
other OS can and will be supported in future releases.

## Format

The following directory structure is stored inside a ZIP archive:

    filesystem{suffix}/   # extracted files go here
    metadata{suffix}/     # metadata goes here
      metadata.msgpack    #   metadata per file 
      filesystem.msgpack  #   filesystem-wide metadata
    extra/                # vendor-specific auxiliary files
    version               # text file containing "CLBX-{semver}"

Each CLBX archive can hold multiple distinct filesystems (e.g. both System and
Data partitions of an iOS device). Each will create a pair of `filesystem` and
`metadata` directories with the same `{suffix}`.
`{suffix}` has no intrinsic meaning and can be either a descriptive string or
a random token. An empty `{suffix}` is also valid.

`filesystem` directories contain a copy of the extracted file tree, as well as
partial metadata fields as natively supported by the ZIP specification.
`metadata` directories contain additional metadata, listing every file and all
extracted fields.


### `filesystem/`
Files are stored uncompressed to make parsing and referencing specific
artifacts inside the archive easier.

The following ZIP metadata fields are populated for convenience, to allow
easier manual inspection of the extracted contents without requiring any
proprietary tools:

 - Mode - Filetype and UNIX permissions are stored in the External Attributes
   field of each file.
 - Timestamps - Modification, Access, and Creation (birth) time are stored in
   1-second resolution in a Timestamps Extra Field (0x5455).
 - GID/UID - Stored in a 0x7875 New Unix Extra Field.
 - Symbolic links are stored as per Info-ZIP extensions.
 - Special files (such as special char/block devices, sockets, etc. but not
   including symlinks) are stored as empty files in the zip with proper mode
   and metadata set.


### `metadata.msgpack`
The complete and authoritative source of metadata is stored in a
[MessagePack][msgpack] encoded dictionary mapping between paths and metadata
objects. Each metadata object is a dictionary mapping between text
representations of fields.

Importantly, `metadata.msgpack` may include references to files whose contents
are inaccessible or could not be extracted for any reason. This metadata may 
provide special value for partial extractions (BFU, AFU) where file content is
inaccessible but file metadata is available for forensic analysis.

| field   | description                                      | example value                                                                                                                                         |
|---------|--------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------|
| `mode`  | as returned from `stat()`                        | 16877
| `uid`   | as returned from `stat()`                        | 33                                                                                                                                                    |
| `gid`   | as returned from `stat()`                        | 33                                                                                                                                                    |
| `size`  | as returned from `stat()`                        | 2016                                                                                                                                                  |
| `inode` | as returned from `stat()`                        | 36516                                                                                                                                                 |
| `mtime` | as returned from `stat()`                        | 1583760561632663172                                                                                                                                   |
| `ctime` | as returned from `stat()`                        | 1583760562436419507                                                                                                                                   |
| `atime` | as returned from `stat()`                        |                                                                                                                                                       |
| `btime` | as returned from `stat()`                        | 1570675980000000000                                                                                                                                   |
| `links` | as returned from `stat()`                        | 63                                                                                                                                                    |
| `prot`  | enum value of Protection Class (iOS only)        | 0                                                                                                                                                     |
| `xattr` | key-value dictionary of Extended Attributes      | {'com.apple.installd.installType': b'\x00\x00\x00\x00\x00\x00\x00\x00',     'com.apple.installd.uniqueInstallID': b'c63ad8cedd13171ad9832bbd653e2a9e' |

Timestamps are stored with nano-second precision.



Example:
```
{
 'Applications/Apple TV Remote.app/PlugIns/TVRemoteIntentExtension.appex/zh_TW.lproj/Intents.strings': {'atime': 1570672647000000000,
  'uid': 0,
  'btime': 1570672647000000000,
  'size': 1945,
  'gid': 80,
  'inode': 9084,
  'ctime': 1570672647000000000,
  'links': 1,
  'mtime': 1570672647000000000,
  'mode': 33204,
  'prot': 0,
  'xattr': {}},
 'Applications/AskPermissionUI.app/Info.plist': {'atime': 1570672625000000000,
  'uid': 0,
  'btime': 1570672625000000000,
  'size': 1240,
  'gid': 80,
  'inode': 5022,
  'ctime': 1570672625000000000,
  'links': 1,
  'mtime': 1570672625000000000,
  'mode': 33204,
  'prot': 0,
  'xattr': {}},
 'Applications/AppStore.app/PlugIns/SubscribePageExtension.appex/zh_HK.lproj': {'atime': 1570672632000000000,
  'uid': 0,
  'btime': 1570672632000000000,
  'size': 96,
  'gid': 80,
  'inode': 6942,
  'ctime': 1570672635000000000,
  'links': 3,
  'mtime': 1570672632000000000,
  'mode': 16893,
  'prot': 0,
  'xattr': {}},
 'private/var/containers/Bundle/Application/8F1372CA-A085-400E-A163-76059AB7B941/MobileCal.app': {'atime': 1570676008000000000,
  'uid': 33,
  'btime': 1570675980000000000,
  'size': 2016,
  'gid': 33,
  'inode': 36516,
  'ctime': 1583760562436419507,
  'links': 63,
  'mtime': 1583760561632663172,
  'mode': 16877,
  'prot': 0,
  'xattr': {'com.apple.installd.installType': b'\x00\x00\x00\x00\x00\x00\x00\x00',
   'com.apple.installd.uniqueInstallID': b'c63ad8cedd13171ad9832bbd653e2a9e'}},
 ...}
```

#### Notes
When parsing the container for forensic purposes, you should rely on the
authoritative metadata copy as referenced in `metadata.msgpack`.

In cases of a live system acquisition, files and file metadata may change from
the moment of metadata collection to the moment of file collection itself.

### `filesystem.msgpack`
The file contains a [MessagePack][msgpack] encoded dictionary mapping between
string encoded key-value pairs describing the corresponding filesystem.

Keys:
 * `"mount_point"`: The logical path a given filesystem should exist. If
   missing, it should be treated as if `mount_point` is `/`.


Examples:

```
{"mount_point": "/"}
```
(suitable for iOS System partition)

```
{"mount_point": "/private/var"}
```
(suitable for iOS Data partition)


[msgpack]: https://msgpack.org/


### `extra/`
Holds vendor-specific auxiliary files for the extraction (e.g. ufd file, keychain/keystore records)

### `version`
Contains the magic `CLBX-` followed by a [SemVer](https://semver.org/) version
for the format. For example: `CLBX-0.3.0`.

###### Copyright (C) 2021 Cellebrite DI LTD

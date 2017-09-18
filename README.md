<!--
    This Source Code Form is subject to the terms of the Mozilla Public
    License, v. 2.0. If a copy of the MPL was not distributed with this
    file, You can obtain one at http://mozilla.org/MPL/2.0/.
-->

<!--
    Copyright (c) 2017, Joyent, Inc.
-->

# manta-mdstress: metadata stress-tester for Manta

`mdstress` is a development tool that connects directly to the Manta metadata
tier and generates certain pathological metadata workloads.  The primary goal
is to reproduce the pathological database performance observed under
[MANTA-3428](https://smartos.org/bugview/MANTA-3428), which involves a fairly
specific metadata layout:

- one directory that grows very large
- a large number of directories with one item in it

The generated metadata is not valid from Manta's perspective, as it creates
references to storage nodes and sharks that do not exist.  **Running this
against any Manta deployment that will ever become a production deployment is
strongly discouraged.**

## Synopsis

Build with:

    $ make

Then make a configuration file that looks like this:

    {
        "metadataService": {
            "srvDomain": "3.moray.emy-10.joyent.us",
            "cueballOptions": {
                "resolvers": [ "nameservice.emy-10.joyent.us" ]
            }
        },
        "concurrency": 1,
        "largeDirectory": "/dap/bigdir/a",
        "smallDirectoryRoot": "/dap/smalldir"
    }

where:

* `metadataService` describes the Moray endpoint to work with, in terms of the
  Moray client constructor options.  In a typical Manta deployment, this would
  point at electric-moray, but for testing this tool, it's more effective to
  point it at a particular set of Moray instances (for one shard).
* `concurrency` specifies how many concurrent application-level operations to
  run at a time.  Each application-level operation creates an entry
  `largeDirectory`, a few directories in `smallDirectoryRoot`, and an object
  inside one of those directories.
* `largeDirectory` points to a directory where one entry will be added per
  application-level operation
* `smallDirectoryRoot` points to a path where a small multi-level directory tree
  will be created for each application-level operation

While the database is small, you can monitor the behavior using PostgreSQL
queries like:

    moray=# select _key, entries from manta_directory_counts ORDER BY entries DESC limit 20;
                   _key                | entries 
    -----------------------------------+---------
     /dap/bigdir/a                     |    2415
     /dap/smalldir/ac                  |     178
     /dap/smalldir/a4                  |     164
     /dap/smalldir/ad                  |     162
     /dap/smalldir/af                  |     159
     /dap/smalldir/a6                  |     158
     /dap/smalldir/a1                  |     153
     /dap/smalldir/aa                  |     152
     /dap/smalldir/a3                  |     152
     /dap/smalldir/a8                  |     151
     /dap/smalldir/a0                  |     148
     /dap/smalldir/ae                  |     146
     /dap/smalldir/a5                  |     146
     /dap/smalldir/a2                  |     141
     /dap/smalldir/a7                  |     138
     /dap/smalldir/ab                  |     134
     /dap/smalldir/a9                  |     132
     /dap/smalldir                     |      16
     /dap/smalldir/a4/a421e29a-2daf-4a |       1
     /dap/smalldir/a4/a41821b1-504d-4b |       1
    (20 rows)

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
        "artediPort": 1809
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
* `artediPort` is a TCP port number on which a Prometheus endpoint will be
  exposed (on all network interfaces) for metrics

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

You can also monitor the program by hitting the artedi port with curl(1) (or, of
course, Prometheus):

    $ curl -i  localhost:1809/metrics
    HTTP/1.1 200 OK
    Content-Type: text/plain; version=0.0.4
    Date: Tue, 19 Sep 2017 00:23:43 GMT
    Connection: keep-alive
    Transfer-Encoding: chunked

    # HELP nstarted count of composite operations started
    # TYPE nstarted counter
    nstarted{} 2179 1505780623275
    # HELP ndone count of composite operations completed (including failures)
    # TYPE ndone counter
    ndone{} 2178 1505780623274
    # HELP nfail count of composite operations that have failed
    # TYPE nfail counter
    # HELP composite_latency_ms latency of composite operations
    # TYPE composite_latency_ms histogram
    composite_latency_ms{le="81"} 1946 1505780623274
    composite_latency_ms{le="243"} 2178 1505780623274
    composite_latency_ms{le="405"} 2178 1505780623274
    composite_latency_ms{le="567"} 2178 1505780623274
    composite_latency_ms{le="729"} 2178 1505780623274
    composite_latency_ms{le="+Inf"} 2178 1505780623274
    composite_latency_ms{le="9"} 0 0
    composite_latency_ms{le="27"} 0 0
    composite_latency_ms{le="45"} 705 1505780623274
    composite_latency_ms{le="63"} 1753 1505780623274
    composite_latency_ms_count{} 2178 1505780623274
    composite_latency_ms_sum{} 120322 1505780623274
    # HELP putmd_latency_ms latency of putmetadata operations
    # TYPE putmd_latency_ms histogram
    putmd_latency_ms{le="9"} 2361 1505780623247
    putmd_latency_ms{le="27"} 8271 1505780623299
    putmd_latency_ms{le="45"} 8663 1505780623300
    putmd_latency_ms{le="63"} 8707 1505780623300
    putmd_latency_ms{le="81"} 8712 1505780623300
    putmd_latency_ms{le="+Inf"} 8714 1505780623300
    putmd_latency_ms{le="1"} 0 0
    putmd_latency_ms{le="3"} 0 0
    putmd_latency_ms{le="5"} 1587 1505780623247
    putmd_latency_ms{le="7"} 2106 1505780623247
    putmd_latency_ms{le="243"} 8714 1505780623300
    putmd_latency_ms{le="405"} 8714 1505780623300
    putmd_latency_ms{le="567"} 8714 1505780623300
    putmd_latency_ms{le="729"} 8714 1505780623300
    putmd_latency_ms_count{} 8714 1505780623300
    putmd_latency_ms_sum{} 111929 1505780623300

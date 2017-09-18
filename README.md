<!--
    This Source Code Form is subject to the terms of the Mozilla Public
    License, v. 2.0. If a copy of the MPL was not distributed with this
    file, You can obtain one at http://mozilla.org/MPL/2.0/.
-->

<!--
    Copyright (c) 2017, Joyent, Inc.
-->

# manta-mdstress: metadata stress-tester for Manta

This tool connects directly to the Manta metadata tier and generates certain
pathological metadata workloads.  The primary goal is to reproduce the
pathological database performance observed under
[MANTA-3428](https://smartos.org/bugview/MANTA-3428), which involves a fairly
specific metadata layout:

- one directory that grows very large
- a large number of directories with one item in it

The generated metadata is not valid, as it creates references to storage nodes
and sharks that do not exist.  **Running this against any Manta deployment that
will ever become a production deployment is strongly discouraged.**

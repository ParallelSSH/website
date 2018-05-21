---
title: "parallel-ssh 1.6.0 release"
date: 2018-05-20T19:03:01+01:00
tags: [parallel-ssh]
---

Version `1.6.0` has been released with many additions and fixes. [Get it on PyPi](https://pypi.org/project/parallel-ssh/).

## Main Changes

* SCP send and receive support for native clients.
* Better exception messages showing underlying error for native clients.
* `host` parameter added to all exceptions raised by parallel clients to better support per-host exception handling.
* Clearer imports by moving clients to their own package.
* Deprecation warning for default client changing from paramiko to native client as of ``2.0.0``.
* Embedded ``libssh2`` upgrade.
  * Adds support for ECDSA host keys for native client.
  * Adds support for SHA-256 host key fingerprints for native client.
* SSH agent forwarding implementation for native client.
* Windows wheels switched to OpenSSL back end for native client.
* Windows wheels include zlib and have compression enabled for native client.
* OSX 10.13 binary wheels.

This release is fully backwards compatible with previous release and further enhances the soon to be default native client, bringing it closer to feature parity with the current default client.

In particular, Windows support is much improved as can be seen above as well as SCP support and binary wheels for a new OSX version.

Updating is highly recommended.

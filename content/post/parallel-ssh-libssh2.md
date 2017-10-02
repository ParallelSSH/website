---
title: "Parallel Ssh Libssh2"
date: 2017-10-01T19:57:01+01:00
subtitle: ""
tags: [parallel-ssh,libssh2,non-blocking,ssh]
---

In a [previous post](../ssh2-python) a comparison was shown between two SSH library options for Python, with emphasis on their performance using native threads for scaling purposes.

For this post, their non-blocking performance using the gevent library will be compared, both as the two SSH library options found in [parallel-ssh](https://github.com/ParallelSSH/parallel-ssh).

# Test Setup

Test script (see appendix) consists of `parallel-ssh` being used in exactly the same way with the two available parallel client in the library, using Paramiko and libssh2 respectively as the underlying SSH library.

The script creates SSH sessions in parallel to a local to the client SSH server (OpenSSH) via loop back device, with a pool size starting from one and increasing by one until completion, there by increasing concurrency by one each time.

A maximum pool size - and concurrency - of thirty is used for the comparison. The Paramiko based client starts exhibiting connection failures and dead locks beyond this number, though further scaling for the libssh2 based client is shown on its own subsequently.

A single remote command, `cat` of a static file, is performed and standard output read one line at a time.

Timings are shown for:

* Commands start - connection, authentication and execute command sent for all hosts
* Commands end - wait for execute completion, close execute channel (SSH session remains open)
* Channel read - read complete remote output line by line

The static file that is used for the `cat` remote command is a `26KB` [license file from the parallel-ssh](https://github.com/ParallelSSH/parallel-ssh/blob/master/COPYING).

All durations are in milliseconds (ms).

## Test Results

Graphs show median values per five second intervals.

Here is how the two SSH clients compare on the three above timings plus total time spent.

Click on image for a larger version.

[![parallel-ssh ssh2 paramiko](/static/pssh_clients.png "Execute and read")](/static/pssh_clients.png)

## Further scaling

This graph shows further scaling of the `libssh2` based client up to one hundred concurrent sessions.

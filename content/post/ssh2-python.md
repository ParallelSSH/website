---
title: "The State of Python SSH Libraries"
date: 2017-08-26T01:17:03+01:00
---

# Preface

*Update: This post has been updated to correct channel open numbers for Paramiko which incorrectly included authentication time. Only channel open and total numbers were affected. [parallel-ssh](../parallel-ssh-libssh2) test is not affected. Thank you Tobias for pointing out [the error in the test script](https://gist.github.com/pkittenis/f4a386ea38d09504a7ba429b45babde6/revisions#diff-91de25b15ce44d18484755e69e49b638).*

In this post a new option for Python SSH libraries, [ssh2-python](https://github.com/ParallelSSH/ssh2-python), shall be compared to Paramiko to demonstrate their respective performance with particular emphasis on concurrency and non-blocking requests.

In looking for a Python SSH library to use in an application, not many options exist. Indeed, the only general purpose library that has, up to now, been available is Paramiko, which implements the SSH2 API in Python code.

For better or worse - it is without doubt that the library has helped a great number of people with similar requirements, this author included as it has historically been the only option - Paramiko has been the de-facto stanard for Python SSH libraries so far.

However, it leaves a lot to be desired of from a performance, stability and resource consumption stand point.

Luckily, there is now another option in the form of `ssh2-python` which is based on the [libssh2](https://www.libssh2.org) C library.

Many automation applications like [Ansible](https://www.ansible.com/) and [Fabric](https://github.com/fabric/fabric) make use of Paramiko and would likely be interested in knowing other options now exist. Hopefully this post will help these and other applications determine which library is better suited.


# Performance Comparison

[ssh2-python](https://github.com/ParallelSSH/ssh2-python) is a new Python SSH library based on the [libssh2](https://www.libssh2.org) C library. It has no dependencies and provides Linux, OSX and Windows binary wheels with `libssh2` included.

[paramiko](https://github.com/paramiko/paramiko) is written in Python and makes use of native extension dependencies like cryptography. It supports Linux, OSX and Windows and its native dependencies provide binary wheels.


## Test Setup

Test is using Python's `threading` standard library for concurrency. A subsequent blog post will examine non-blocking performance in the libraries via the [gevent](http://gevent.org) co-routine library. 

While threading is not a good model for scaling concurrency of network I/O, it is the only concurrency mode Paramiko supports natively and is what is used by applications like Ansible. Other projects like [parallel-ssh](https://github.com/ParallelSSH/parallel-ssh), of which am also author of, use gevent's monkey patching to make Paramiko co-operatively concurrent. That too comes with significant drawbacks which the currently in-development `ssh2-python` based natively non-blocking client aims to solve. See the [work in progress pull request](https://github.com/ParallelSSH/parallel-ssh/pull/86) for more details on that.

The test script creates SSH sessions in parallel to an SSH server (OpenSSH) via loop back device (localhost), starting from one and increasing by one each iteration until completion. This shows how the two libraries scale as number of parallel sessions increases.

Maximum number of threads and therefor parallel sessions are set to 50. All tests are performed on a quad physical core CPU.

In all tests the latest available version of each library is used, ``2.2.1`` and ``0.5.3`` for Paramiko and ssh2-python respectively. For `ssh2-python` an embedded `libssh2` was used, latest available version `1.8.0`. All tests performed under Python `2.7`.

The libraries are compared in six separate SSH operations, as well as total time spent from start to finish per SSH session.

The six operations compared are:

* Session initialisation and authentication with SSH agent (`auth`)
* Channel open (`channel_open`)
* Channel execute (`execute`)
* Channel read - (`channel_read`)
* Channel close and get exit status (`close_and_exit_status`)
* SFTP read (`sftp_read`)

A quick explanation of what these operations do:

* Session initialisation and authentication - perform handshake with SSH server and authenticate via a system SSH agent.
* Channel open - one new SSH channel on an authenticated session is required to be opened per remote command to run.
* Channel execute - run shell command `cat` on a static file
* Channel read - retrieving command output.
* Channel close - performed when command is finished in order to gather exit status.
* SFTP read - initialise SFTP session, open remote file handle, read data. Read data is not written anywhere.


The static file that is used for the `cat` remote command is a `26KB` [license file from the ssh2-python repository](https://github.com/ParallelSSH/ssh2-python/blob/master/LICENSE).

The file used for SFTP read is an 11MB compressed tar archive.

All durations are in milliseconds (ms).


### Test Results

Graphs show median values per thirty second intervals.

#### All Operations Graph

Here is how the two libraries compare on the five separate operations plus total time spent.

Click on image for a larger version.

[![ssh2 paramiko comparison](/static/ssh2_paramiko.png "All Operations Comparison")](/static/ssh2_paramiko.png)

As can be seen above, Paramiko durations are dominated by SFTP read, authentication and channel open which keep increasing as concurrency ramps up.

For ssh2-python, biggest time spent is in SFTP read followed by auth and channel open. 

Both libraries show intermittent spikes in duration that are expected given the number of threads and blocking calls used.


#### Individual Operations

Times for operations not including SFTP and total to show a closer view of the rest of timings.

Click on image for a larger version.

[![ssh2 paramiko comparison](/static/ssh2_paramiko_median.png "Individual Operations Comparison")](/static/ssh2_paramiko_median.png)

# Relative Performance

Relative performance of the averages of *median* operations of the two libraries for the duration of the test. Total and SFTP read durations are for averages.

For example if a `Paramiko` operation were twice as fast as the equivalent `ssh2-python` operation, its relative performance would be `x0.5` of ssh2-python whereas identical durations would result in `x1` relative performance.


Operation| Paramiko | ssh2 | Paramiko/ssh2-python relative difference
---------| ---------| -----| -------------
`auth` | 1.16 sec | 675 ms | `x1.71`
`channel open` | 1.248 sec | 141 ms | `x8.85`
`channel read` | 78 ms |  29 ms | `x2.68`
`close and exit status` | 4 ms | 1 ms | `x4`
`execute` | 24 ms | 3 ms | `x8`
`sftp_read` | 19.35 sec | 1.13 sec | `x17.12`
`total` | 20.82 sec | 2.04 s | `x10.2`


# Postface

In all, `ssh2-python` is shown to be considerably faster, particularly in heavy operations like SFTP, for which reading is some `x17` times faster on average compared to Paramiko. Other operations that particularly benefit are channel open, `x8` faster, and execute, also `x8`. In total in the above test case `ssh2-python` is shown to be about an order of magnitude (`x10`) faster on average.

Also note that memory consumption is not graphed, but is significantly lower with `ssh2-python` as expected as it is based on a native library.

As for comparisons between a pure Python code SSH library and one based on a C library being "*unfair*" - the "*pure*" Python code library also uses native code extensions for cryptography operations, making the argument moot. Note how the operation with the *least* performance difference is authentication which is mostly handled by native code extensions in Paramiko.

There is also no requirement to write a low-level SSH API implementation purely in Python. Many reasons not to in fact, as these results go some way in showing.


# Appendix

Test script used can be found below:

* [Performance test script](https://gist.github.com/pkittenis/f4a386ea38d09504a7ba429b45babde6#file-perf_test_ssh2-py)

The test script writes duration data to a local [InfluxDB](https://portal.influxdata.com/downloads) via its Graphite service using a `measurement.field*` [template](https://github.com/influxdata/influxdb/tree/master/services/graphite#templates).

Graphs above were generated by [Grafana](https://grafana.com), as queried from InfluxDB by [InfluxGraph](https://github.com/InfluxGraph/influxgraph) using [docker-compose](https://github.com/InfluxGraph/influxgraph/tree/master/docker/compose) with database set to `graphite`.

To replicate results, take care to use the exact versions of the libraries used here, including `libssh2`. Older versions of `libssh2` are not completely thread safe and will cause crashes.

Disclaimer - While I am the author of `ssh2-python`, I have no vested interest in either `libssh2` or `Paramiko`. My interest in their respective performance is for use in [parallel-ssh](https://github.com/ParallelSSH/parallel-ssh).

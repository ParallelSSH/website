---
title: "The Sad State of Python SSH Libraries"
date: 2017-08-21T12:06:06+01:00
draft: true
---


# Preface

In this post a new option for Python SSH libraries, [ssh2-python](https://github.com/ParallelSSH/ssh2-python), shall be compared to Paramiko to demonstrate their respective performance with particular emphasis on concurrency and non-blocking requests.

In looking for a Python SSH library to use in an application, not many options exist. Indeed, the only general purpose library that has, up to now, been available is Paramiko, which implements the SSH2 API in Python.

For better or worse - it is without doubt that the library has helped a great number of people with similar requirements as it has historically been the only option - Paramiko has been the de-facto stanard for Python SSH libraries until now.

However, it leaves a lot to be desired from a performance, stability and resource consumption stand point.

Luckily, there is now another option in the form of `ssh2-python` which is based on the [libssh2](https://www.libssh2.org) C library.

Many automation applications like [Ansible](https://www.ansible.com/) and [Fabric](https://github.com/fabric/fabric) make use of Paramiko and would likely be interested in knowing other options now exist. Hopefully this post will help these and other applications determine which library is better suited.

# Performance Comparison

[ssh2-python](https://github.com/ParallelSSH/ssh2-python) is a new Python SSH library based on the [libssh2](https://www.libssh2.org) C library. It has no dependencies and provides Linux, OSX and Windows binary wheels with `libssh2` included.

[paramiko](https://github.com/paramiko/paramiko) is written in Python and makes use of native extension dependencies like cryptography. It supports Linux, OSX and Windows and its native dependencies provide binary wheels.

## Test Setup

Two separate test setups will be examined, one using threading and one using non-blocking network I/O via the [gevent](https://gevent.org) library.

In all cases a test script creates SSH connections in parallel to a local to the test script SSH server (OpenSSH), starting from one and increasing by one each time until completion. This shows how the two libraries scale as number of parallel connections increases.

Max parallel clients differ in the two test cases, fifty (50) for threading and one hundred (100) for non-blocking due to diminishing returns in the threading case.

In all tests the latest available version of each library is used, ``2.2.1`` and ``0.5.3`` for Paramiko and ssh2-python respectively. For `ssh2-python` an embedded `libssh2` was used, latest available version `1.8.0`.

The libraries are compared in five separate SSH operations, as well as total time spent from start to finish per SSH session.

The four operations compared are:

* Session initialisation and authentication (`auth`)
* Channel open (`channel_open`)
* Channel execute shell command, `cat` on a static file (`execute`)
* Channel read - (`channel_read`)
* Channel close and get exit status (`close_and_exit_status`)

A quick explanation of what these operations do:

* Session initialisation and authentication - perform handshake with SSH server and authenticate via a system SSH agent.
* Channel open - one new SSH channel on an authenticated session is required to be opened per remote command to run.
* Channel read - retrieving command output.
* Channel close - performed when command is finished in order to gather exit status.

The static file that is used for the `cat` remote command is a `26KB` [license file from the ssh2-python repository](https://github.com/ParallelSSH/ssh2-python/blob/master/LICENSE).

All durations are in milliseconds (ms).


### Test 1 - Threading <a name="threading"></a>

In the threading test, each SSH session is run in a separate thread via Python's `threading` standard library. No external dependencies are used.


#### All Operations Graph

Here is how the two libraries compare on the four separate operations plus total time spent.

Click on image for a larger version.

[![ssh2 paramiko comparison](/static/ssh2_paramiko.png "All Operations Threading Comparison")](/static/ssh2_paramiko.png)

As can be seen above, Paramiko durations are dominated by authentication and channel open which keep increasing as concurrency ramps up.

For ssh2-python, biggest time spent is in channel read, followed by channel open and auth, with spikes in latency that are somewhat expected, given the blocking calls used.


#### Median Operations

Median times for operations, not including total time, to give a more normalised view of timings excluding outliers.

Click on image for a larger version.

[![ssh2 paramiko comparison](/static/ssh2_paramiko_median.png "All Operations Threading Comparison")](/static/ssh2_paramiko_median.png)


### Test 2 - Non-Blocking <a name="non-blocking"></a>

In the non-blocking case, each SSH session is run in a separate `greenlet`. Since Paramiko does not support non-blocking requests, gevent's monkey patching was used to patch the standard library as non-blocking. This allows the code between the two scripts to remain identical, other than the monkey patching.

While ssh2-python supports non-blocking mode natively via `libssh2`, keeping the test code identical allows for a more fair apples-to-apples comparison with Paramiko. A native non-blocking client could, however, perform better with `ssh2-python`.


#### All Operations Graph

Click on image for a larger version.

[![ssh2 paramiko comparison](/static/ssh2_paramiko_gevent.png "All Operations non-blocking Comparison")](/static/ssh2_paramiko_gevent.png)


In the non-blocking case, authentication and channel open again dominate Paramiko durations. Predominantly blocking calls like channel read, close and exit status durations have reduced significantly.

For `ssh2-python`, it appears that the non-blocking socket provided by `gevent` has smoothed out spikes in latency in its operations and as a result operations that predominantly wait for remote output *do not increase in duration* as concurrency ramps up. 

This matches similar reductions in the Paramiko case but in a larger number of operations, pointing to in-code overhead for these operations in Paramiko.

Highest `ssh2-python` durations are in authentication and channel open in this case.


#### Median Operations

Here are median times for the non-blocking case, excluding total time.

Click on image for a larger version.

[![ssh2 paramiko comparison](/static/ssh2_paramiko_gevent_median.png "Individual Operations non-blocking Comparison")](/static/ssh2_paramiko_gevent_median.png)


# Relative Performance

Performance of the averages of *median* operation of the two libraries for the duration of each test compared relative to each other. Total duration is an absolute time, not a median.

For example if a `Paramiko` operation is twice as fast as the equivalent `ssh2-python` operation, its relative performance difference is `50%` compared to ssh2-python. Identical durations result in `100%` relative performance.

## Threading


Operation| Paramiko | ssh2 | Paramiko/ssh2-python relative difference
---------| ---------| -----| -------------
`auth` | 1.131 sec | 120 ms | `-942.5%`
`channel open` | 1.217 sec | 93 ms | `-1308.6%`
`channel read` | 81 ms | 181 ms | `+44.75%`
`close and exit status` | 5ms | 17 ms | `+29.41%`
`execute` | 34 ms | 37 ms | `+91.89%`
`total` | 33.01 sec | 10.16 sec | `-324.77%`


## Non-blocking


Operation| Paramiko | ssh2 | Relative Performance
---------| ---------| -----| -------------
`auth` | 1.662 sec | 314 ms | `-529.29%`
`channel open` | 1.818 sec | 20 ms | `-9090.0%`
`channel read` | 4 ms | 4 ms | `100%`
`close and exit status` | 0 ms | 0 ms | `100%`
`execute` | 151 ms | 0 ms | `-15100%`
`total` | 56.68 sec | 9.39 sec | `-603.62%`

# Postface


For the threading case, `ssh2-python` is `3.24` times faster in total duration.

For the non-blocking case, `ssh2-python` is `6.03` times faster in total duration.

Individual operations wise, for the threading case `ssh2-python` is `9.4` and `13` times faster in authentication and channel open respectively. Execute is a bit slower at `0.9` times slower than Paramiko. Channel read is `0.4` times slower on average.

For the non-blocking case, `ssh2-python` is `5.29` and `90` times faster in authentication and channel open respectively. Execute is `151` times faster. Channel read is identical.

Note that memory consumption is not graphed, but is significantly lower with `ssh2-python` as expected as it is based on a native library.

For a non-blocking parallel SSH library see [parallel-ssh](https://github.com/ParallelSSH/parallel-ssh) which I am also the author of. It currently uses Paramiko with an `ssh2-python` client currently in development and in line for the next release.


# Appendix

Test scripts used for the two test cases can be found below:

* [Threading](https://gist.github.com/pkittenis/f4a386ea38d09504a7ba429b45babde6#file-perf_test_ssh2-py)
* [Non-blocking](https://gist.github.com/pkittenis/f4a386ea38d09504a7ba429b45babde6#file-perf_test_non_blocking_ssh2-py)

The test scripts write duration data to a local [InfluxDB](https://portal.influxdata.com/downloads) via its Graphite service. 

Graphs above were generated by [Grafana](https://grafana.com), as queried from InfluxDB by [InfluxGraph](https://github.com/InfluxGraph/influxgraph).

To replicate results, take care to use the exact versions of the libraries used here, including `libssh2`. Older versions of `libssh2` are not completely thread safe and will cause crashes.

Disclaimer - While I am the author of `ssh2-python`, I have no vested interest in either `libssh2` or `Paramiko`. My interest in their respective performance is for use in [parallel-ssh](https://github.com/ParallelSSH/parallel-ssh).

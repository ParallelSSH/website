---
title: "Parallel Ssh Libssh2"
date: 2017-10-14T15:00:01+01:00
subtitle: ""
tags: [parallel-ssh,libssh2,non-blocking,ssh]
---

In a [previous post](../ssh2-python) a comparison was shown between two SSH library options for Python, with emphasis on their performance using native threads for scaling purposes.

For this post, their non-blocking performance using the gevent library will be compared as the two SSH library options available now in [parallel-ssh](https://github.com/ParallelSSH/parallel-ssh).

# Test Setup

Test script (see appendix) consists of identical `parallel-ssh` code for the two available parallel clients in the library, using `paramiko` and `libssh2` via `ssh2-python` respectively as the underlying SSH libraries. The latest version of the library as of this time of writing was used - `1.2.0` - with latest underlying libraries as installed by its requirements and latest version of `gevent` - `1.2.2`.

The script creates SSH sessions in parallel to a local to the client SSH server via loop back device, with concurrencystarting from one and increasing by one until completion.

A maximum concurrency of one hundred sessions is used for the comparison test.

A single remote command, `cat` of a [26KB static file](https://github.com/ParallelSSH/ssh2-python/blob/master/LICENSE), is performed and standard output read one line at a time as parsed by the respective libraries.

Timings are shown for:

* Auth and execute - connection, authentication and execute command sent for all hosts
* Close and exit status - wait for execute completion, close execute channel and get exit status (SSH session remains open)
* Execute - same remote command executed again on the same SSH session
* Channel read - read complete remote output line by line

All durations are in milliseconds (ms) or multiples of. All tests performed on Linux, kernel version `4.4` on a six core CPU and OpenSSH `7.5r1` with OpenSSL `1.0.2l`.

Click on images for larger versions.

## Test Results

Graphs show median values per five second intervals.

Here is how the two SSH clients compare on the above timings plus total time spent.

### Clients Comparison

Shown below is time taken for *all* hosts in total.

[![parallel-ssh ssh2 paramiko](/static/pssh_clients.png "Clients Comparison")](/static/pssh_clients.png)

## Individual Results

Results for the clients individually, including CPU and memory usage graphs.

[![parallel-ssh ssh2](/static/pssh_ssh2.png "Individual SSH2")](/static/pssh_ssh2.png)

[![parallel-ssh paramiko](/static/pssh_paramiko.png "Individual paramiko")](/static/pssh_paramiko.png)

The `ssh2-python` based client is shown to use about 250MB less memory and fully utilising a little more than three cores at highest concurrency, with the `paramiko` client a little more than one core.

## Notes

Further scaling of the ``ssh2`` client is possible, though ``gevent`` and event loop overhead leads to diminishing returns, seen in the further scaling graph as intermittent spikes in latency that increase in frequency as concurrency ramps up. No limit was found as to how far concurrency could be increased.

On already authenticated sessions, executing new commands remains fairly low latency on the `ssh2` client even at high concurrency levels.

The paramiko client would also often experience dead locks at all concurrency levels.

# Relative Performance

Relative performance of average times of the two libraries for the duration of the test at one hundred concurrent sessions.

For example if a `paramiko` operation were twice as fast as the equivalent `ssh2-python` operation, its relative performance would be `x0.5` of `ssh2-python` whereas identical durations would result in `x1` relative performance.


Operation| paramiko | ssh2 | paramiko/ssh2 relative performance
---------| ---------| -----| -------------
`auth and execute` |  3.018 sec | 1.013 sec | `2.98x`
`close and exit status` | 4 ms | 0 ms | `inf`
`execute` | 205 ms |  100 ms | `2.05x`
`channel read` | 165 ms | 89 ms | `1.85x`
`total` | 3.392 sec | 1.203 sec | `2.82x`


# Postface

On average, the `ssh2-python` (`libssh2`) based client in `parallel-ssh` is shown to be about three times faster than the existing paramiko based client.

Note that SFTP operations which [were previously shown to benefit the most](../ssh2-python) are not shown here as tests are against a single, local, SSH server and `parallel-ssh` SFTP operations are for copying files which would overwrite each other if copied to/from the same server.

It is also worth pointing out that this test puts a lot of strain on gevent and the event loop as the commands are short lived, causing contention on the event loop from all the coroutines wanting to execute. This can be seen in the graphs by the intermittent spikes in latency for both clients which increase in frequency as concurrency ramps up.

For longer lived remote commands, scaling can continue before diminishing returns stemming from event loop and gevent overhead.

In all, tests show good scaling for the native library based client and the benefits of using native code extensions in Python libraries.

Combining native code extensions for network libraries with a non-blocking mode and non-blocking network libraries like `gevent` allows for very good performance while at the same time benefiting from the ease of use and conciseness of the Python language.

In future it would be interesting to examine further scaling of gevent and the event loop via native threads which is only possible with native code extensions that release Python's GIL, putting `parallel-ssh` in a good position to take advantage of.

# Appendix

Test script and dashboard used can be found below:

* [Performance test script](https://gist.githubusercontent.com/pkittenis/e0221acae77fd0fc251951dd9c6aa69b)
* [Grafana Dashboard for graphs](https://gist.github.com/pkittenis/c47938539d2e8880ca920f2f1cc02abe)

[Docker compose file](https://github.com/InfluxGraph/influxgraph/blob/master/docker/compose/) can be used as-is to easily setup the dashboard, API and DB services as described below.

The test script writes data to a local [InfluxDB](https://portal.influxdata.com/downloads) via its Graphite service using a `measurement.field*` [template](https://github.com/influxdata/influxdb/tree/master/services/graphite#templates).

Graphs were generated by [Grafana](https://grafana.com), as queried from InfluxDB by [InfluxGraph](https://github.com/InfluxGraph/influxgraph) using the [same template configuration as InfluxDB](https://github.com/InfluxGraph/influxgraph#influxdb-graphite-metric-templates).

To replicate results, take care to use the exact versions of the libraries used here, including `libssh2`.

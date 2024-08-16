# Apache CXF JAX-RS Performance

We’ve investigated [Apache CXF Soap
Performance](https://github.com/savoirtech/apache-cxf-soap-performance)
in our testing lab, now it’s time to focus on JAX-RS. We’ll be using the
same systems, so we’ll get to see the through-put difference between
these approaches.

We’ll also get to see some of the config and tuning required to ready
the systems to stably run the performance suite. All of our testing will
use Adoptium Eclipse Temurin 17 LTS as it’s available for both PPC64LE
and X64 Linux systems.

<figure>
<img src="./assets/images/PPC64LEvsX64.png" alt="PPC64LEvsX64" />
</figure>

# The Setup

<figure>
<img src="./assets/images/Systems.png" alt="Systems" />
</figure>

For our lab test we’ll be using the following hardware:

- Dell PowerEdge R250

  - Intel Xeon E-2378 (8c, 16t)

  - 128 GB DDR4 RAM

  - 1 Gigabit Ethernet

  - Ubuntu 22.04 LTS

- Raptor Blackbird

  - IBM POWER9 v2 SMT4 Sforza (8c, 32 t)

  - 128 GB DDR4 RAM

  - 1 Gigabit Ethernet

  - CentOS Stream 9

- Dell PowerConnect 2808 (network switch)

The machines are co-located on the same switch, reducing the number of
packet hops.

# The Performance Harness

As of CXF 4.1 the binary distribution will contain a set of performance
scripts in the samples folder. Options to test JAX-WS and JAX-RS are
present.

<figure>
<img src="./assets/images/Apache-CXF-Perf-Harness.png" alt="Perf" />
</figure>

At its core, the performance harness is a client-server request/response
automation. On startup the script initializes and warms up the JVM for
executing mass calls.

## How it works

The client host runs a number of threads, each running a CXF client
which calls the server host. For JAX-RS testing, we have a choice of
calling using a verb (GET, POST, PUT, DELETE). The client side harness
will run N threads for M times for the specified duration.

<figure>
<img src="./assets/images/RestCalls.png" alt="Rest" />
</figure>

Once the time duration has been met, it will cease the executing
clients, and tabulate the total calls.

# Theory Time!

In our previous performance lab we were attempting to achieve 1 Billion
invocations in an eight-hour period. Let’s see what JAX-RS can do.

Before we start our labs we shall run a few 60-second quick tests to
dial in client counts for our systems (x64 client → PPC64LE server,
PPC64LE client → x64 server).

| Clients | Target Calls/Second per client | Quick Test (Reality) Calls Per Second Per Thread on x64 client | Quick Test (Reality) Calls Per Second Per Thread on PPC64LE client |
|----|----|----|----|
| 1 | 34722.2 | 1338.6 | 665.55 |
| 8 | 4340.27 | 2386.96 | 2325.85 |
| 16 | 2170.14 | 1728.17 | 1694.91 |
| 32 | 1085.07 | ***1414.77*** | 867.71 |
| 64 | 542.53 | ***852.64*** | 470.66 |
| 128 | 271.27 | ***510.38*** | 229.56 |
| 256 | 135.63 | ***237.67*** (sweet spot) | 117.10 |
| 512 | 67.81 | ***116.97*** | 57.68 |
| 1024 | 33.90 | ***59.10*** | 32.07 |
| 2048 | 16.95 | ***30.58*** | ***16.98*** (best fit) |

<table>
<colgroup>
<col style="width: 50%" />
<col style="width: 50%" />
</colgroup>
<thead>
<tr>
<th style="text-align: center;">PPC64LE</th>
<th style="text-align: center;">X64</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align: left;"><figure>
<img src="./assets/images/PPC64LETests.png" alt="PPC64LETests" />
</figure></td>
<td style="text-align: left;"><figure>
<img src="./assets/images/X64Tests.png" alt="X64Tests" />
</figure></td>
</tr>
<tr>
<td style="text-align: left;"><p>When running PPC64LE as the server-side
we hope to achieve 60843.52 calls per second (256 threads * 237.67 =
60843.52).</p></td>
<td style="text-align: left;"><p>When running x64 as the server-side we
hop to achieve 34775.04 calls per second (2048 threads * 16.98 =
34775.04).</p></td>
</tr>
<tr>
<td style="text-align: left;"><p>1,752,293,376 projected eight-hour
served request goal.</p></td>
<td style="text-align: left;"><p>1,001,521,152 projected eight-hour
served request goal.</p></td>
</tr>
</tbody>
</table>

# Lets get this test case running

To run the performance harness we change directory into samples. Within
this folder we’ll build the base harness and the various scenarios.

On each host we will open a terminal to the CXF distribution samples
folder.

We’ll ensure we have JAVA_HOME and MAVEN_HOME environment variables set.

For our runs we’ll use Adoptium Eclipse Temurin 17 LTS as Client and
Server side JVM.

We set our Heap size to 8GB.

``` bash
MAVEN_OPTS="-Xms32m -Xmx8192m -Dmaven.artifact.threads=5"
```

``` bash
$ cd samples
$ mvn clean install
$ cd performance/jaxrs
```

On the Server host we’ll execute the following maven profile:

``` bash
$mvn -Pserver -Dhost=0.0.0.0 -Dprotocol=http
```

On the Client host we’ll execute the client profile, supplying
instructions to use get operation, 256 threads (simulate 256 clients),
over a time of 8 hours (60 x 60 x 8 = 28800 seconds).

``` bash
$mvn -Pclient -Dhost=192.168.50.154 -Dprotocol=http -Doperation=get -Dthreads=256 -Dtime=28800
```

For the purposes of our lab test, we’ll allow the suite to execute
without added agents to the JVM.

# Lab Time!

## First Iteration

On our first iteration we quickly encountered a runtime error.

Client Side:

``` bash
ConnectException invoking http://192.168.50.154:9000/customerservice/customers/123: Cannot assign requested address
```

Given our quick tests indicated we have valid configuration for
connection between client and server side, we’ll attempt reduce thread
count on our second run.

## Second Iteration

``` bash
$mvn -Pclient -Dhost=192.168.50.154 -Dprotocol=http -Doperation=get -Dthreads=128 -Dtime=28800
```

Client Side:

``` bash
ConnectException invoking http://192.168.50.154:9000/customerservice/customers/123: Cannot assign requested address
```

## Third Iteration

The "Cannot assign requested address" tends to indicate that we’re
saturating the port with so many connections.

``` bash
$mvn -Pclient -Dhost=192.168.50.154 -Dprotocol=http -Doperation=get -Dthreads=64 -Dtime=28800
```

This quickly failed as well.

Checking ulimits, file count was restricted to 1024. We update this to
10240 and retest.

## Fourth Iteration

``` bash
$mvn -Pclient -Dhost=192.168.50.154 -Dprotocol=http -Doperation=get -Dthreads=256 -Dtime=28800
```

Server Side:

``` bash
Aug 08, 2024 8:43:42 AM org.eclipse.jetty.server.AbstractConnector handleAcceptFailure
WARNING: Accept Failure
java.io.IOException: Too many open files
```

## Fifth Iteration

We need to increase the number of available file handles on our systems.

``` bash
$sudo vi /etc/security/limits.conf
*           soft    nofile          655350
*           hard    nofile          655350
```

Restart system.

``` bash
$ulimit -n unlimited
$ulimit -n
655350
```

Lets retry our initial test case:

``` bash
$mvn -Pclient -Dhost=192.168.50.154 -Dprotocol=http -Doperation=get -Dthreads=64 -Dtime=28800
```

Results in:

``` bash
Cannot assign requested address
```

<figure>
<img src="./assets/images/LabTest.png" alt="LabTest" />
</figure>

The server side file handle exhaustion appears to be managed. The client
side is still experiencing bind exceptions. We are going to resolve the
bind exceptions and get this lab system rolling!

## Sixth Iteration

So the issue we’re hitting is called ephemeral port exhaustion.

``` bash
[jgoodyear@localhost jaxrs]$ cat /proc/sys/net/ipv4/ip_local_port_range
32768   60999
```

Our systems local port range is about 28k connections (60999 - 32768).
Our testing scenario has been attempting to push 256 threads x 237.67
calls/second == ~60843 calls/second - we exhaust the range, which
reports as a bind exception to us.

We have a couple of options to improve our performance:

- Increase port range (this has limits 65535 for IPV4 or IPV6)

- Tweak time wait settings (not something we generally want to do)

- Add NIC ports to scale range (load balancing clients over addresses)

### Theory Time Revisited!

We extend our port range as follows:

``` bash
$ sudo sysctl -w net.ipv4.ip_local_port_range="15000 64000"
net.ipv4.ip_local_port_range = 15000 64000
```

This provides us with some 49000 ephemeral ports.

Now lets re-run our table of values, with 49k ports in use as a ceiling
value (also retaining the other configuration changes).

| Clients | PPC64LE Server / X64 Client in Calls/Second | New Connections (Threads x Calls/Second) | PPC64LE Client / X64 Server in Calls/Second | New Connections (Threads x Calls/Second) |
|----|----|----|----|----|
| 1 | 1196.50 | 1196.50 | 1264.84 | 1264.84 |
| 8 | 2448.19 | 19585.52 | 2182.08 | 17456.64 |
| 16 | 1886.69 | 30187.04 | 1590.65 | 25450.4 |
| 32 | 1449.83 | 46394.56 | 1019.92 | 32637.44 |
| 64 | 942.33 | 60309.12 | 553.47 | 35422.08 |

These numbers represent new connections happening in a 1-second period -
many of those ports are going to be in use, so we do not expect new
connections/second to be through put sweet spot.

In theory having 49000 active connections/second will get us to 49000 x
28800 = 1,411,200,000 calls processed in an eight-hour period.

``` bash
$mvn -Pserver -Dhost=0.0.0.0 -Dprotocol=http
```

``` bash
$mvn -Pclient -Dhost=192.168.50.154 -Dprotocol=http -Doperation=get -Dthreads=16 -Dtime=28800
```

While running the perf suite, we observe:

Server Side:

``` bash
[jgoodyear@localhost ~]$ ss -s
Total: 34371
TCP:   39980 (estab 16011, closed 6131, orphaned 0, timewait 6131)
```

Client Side:

``` bash
jgoodyear@jgoodyear-PowerEdge-R250:~$ ss -s
Total: 41580
TCP:   40883 (estab 16010, closed 0, orphaned 0, timewait 0)
```

Several minutes later however we observed:

``` bash
jakarta.ws.rs.ProcessingException: java.net.ConnectException: ConnectException invoking http://192.168.50.154:9000/customerservice/customers/123: Cannot assign requested address
```

We still ran out of ephemeral ports!

## Seventh Iteration

Our performance client is not closing out connections.

### Let’s play spot the connection leak!

Our original client code:

``` java
try {
    Response respGet = webClient.get();
    Asserts.check(respGet.getStatus() == 200, "Get should have been OK");
}
```

Can you spot the connection leak?

Here’s a hint - the Response object retains an input stream.

We update the test client code to force response objects to close their
streams. We resolve this by allowing the auto close feature to close out
connections.

``` java
try (Response respGet = webClient.get()) {
    Asserts.check(respGet.getStatus() == 200, "Get should have been OK");
}
```

With this change in place, we setup to run another test.

``` bash
$mvn -Pserver -Dhost=0.0.0.0 -Dprotocol=http
```

``` bash
$mvn -Pclient -Dhost=192.168.50.154 -Dprotocol=http -Doperation=get -Dthreads=16 -Dtime=28800
```

This resulted in:

``` bash
=============Overall Test Result============
Overall Throughput: get 1772.9019709123222 (invocations/sec)
Overall AVG. response time: 0.5640469785734444 (ms)
8.16959077E8 (invocations), running 460803.29899999994 (sec)
============================================
```

In this run our system stability obtained 816,959,077 calls in an
eight-hour period. Given we were running just 16 clients, this number is
pretty good (comparing to our JAX-WS perf testing).

During the test run we observed the following socket statistic:

PPC64LE Server Side:

``` bash
[jgoodyear@localhost ~]$ ss -s
Total: 545
TCP:   23 (estab 18, closed 0, orphaned 0, timewait 0)
```

x64 Client Side:

``` bash
jgoodyear@jgoodyear-PowerEdge-R250:~$ ss -s
Total: 720
TCP:   5013 (estab 18, closed 4990, orphaned 0, timewait 4990)
```

Our concurrent connections appeared to be stable.

We’re ready to ramp up connections!

## Eighth Iteration

Lets run 32 Client threads and check for stability, and throughput.

This resulted in:

``` bash
=============Overall Test Result============
Overall Throughput: get 1363.052828409949 (invocations/sec)
Overall AVG. response time: 0.7336472799565198 (ms)
1.256210783E9 (invocations), running 921615.624 (sec)
============================================
```

1,256,210,783 calls processed in eight-hour period.

## Ninth Iteration

Our system still appears stable, lets run 64 Clients (PPC64LE server,
x64 Clients).

This resulted in client side exceptions:

``` bash
jakarta.ws.rs.ProcessingException: java.net.ConnectException: ConnectException invoking http://192.168.50.154:9000/customerservice/customers/123: Cannot assign requested address
```

We hit the port range limit again.

## Tenth Iteration

One more configuration to try before swapping machine roles, in this run
we’ll try 48 clients.

This resulted in :

``` bash
=============Overall Test Result============
Overall Throughput: get 1092.5105255070816 (invocations/sec)
Overall AVG. response time: 0.9153229892552811 (ms)
1.510322517E9 (invocations), running 1382432.921 (sec)
============================================
```

This time we managed 1,510,322,517 calls in eight-hour period.

## Eleventh Iteration

Lets turn roles around, running x64 server-side, and PPC64LE clients.
Same configurations and tunings applied to each host. We will start with
32 clients.

This resulted in:

``` bash
=============Overall Test Result============
Overall Throughput: get 831.0453771395204 (invocations/sec)
Overall AVG. response time: 1.203303727459535 (ms)
7.65905672E8 (invocations), running 921617.1500000001 (sec)
============================================
```

Our first run yielded 765,905,672 calls in an eight-hour period.

## Twelfth Iteration

Lets double clients to 64.

This resulted in:

``` bash
=============Overall Test Result============
Overall Throughput: get 413.4362013002846 (invocations/sec)
Overall AVG. response time: 2.4187528737322297 (ms)
7.62073003E8 (invocations), running 1843266.266 (sec)
============================================
```

This run managed to process fewer calls at 762,073,003 in eight hours.

## Thirteenth Iteration

Our prior tables suggested we need a higher number of threads to achieve
maximum throughput, so we’ll try 256 clients for our next run. We’ll
observe socket statistics for system pressure.

This resulted in:

``` bash
=============Overall Test Result============
Overall Throughput: get 105.19034791566503 (invocations/sec)
Overall AVG. response time: 9.506575648953428 (ms)
7.75651656E8 (invocations), running 7373791.1450000005 (sec)
============================================
```

This time we processed 775,651,656 calls in eight-hours. A slight
improvement, but our average response time is starting to suffer for a
slight throughput gain.

The PPC64LE acting as client host did manage to keep its connections
stable:

``` bash
[jgoodyear@localhost ~]$ ss -s
Total: 787
TCP:   7012 (estab 260, closed 6747, orphaned 0, timewait 6747)
```

# Results and Conclusion

As our first foray into JAX-RS performance testing, we quickly learned
about system resources that would become bottlenecks. Once we adjusted
those values we could start running our eight-hour test cases.

The key bottleneck per system turned out to be managing client side
ephemeral port exhaustion. Ensuring our clients close in-use ports as
quickly as possible was our first major improvement towards running our
test cases, dialing in the total number of client threads was the
second.

## System Config Settings TL;DR

| Parameter | Setting |
|----|----|
| MAVEN_OPTS | -Xms32m -Xmx8192m |
| file handle ulimit | /etc/security/limits.conf hard & soft limits increased to 655350 |
| ip_local_port_range | sysctl -w net.ipv4.ip_local_port_range="15000 64000" to allow 49K connections. |

## Observations:

<table>
<colgroup>
<col style="width: 50%" />
<col style="width: 50%" />
</colgroup>
<thead>
<tr>
<th style="text-align: center;">PPC64LE</th>
<th style="text-align: center;">X64</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align: center;"><figure>
<img src="./assets/images/CoolPPC64LE.png" alt="PPC64LE" />
</figure></td>
<td style="text-align: center;"><figure>
<img src="./assets/images/CoolX64.png" alt="X64" />
</figure></td>
</tr>
<tr>
<td style="text-align: left;"><p>Original Target: 1,752,293,376</p></td>
<td style="text-align: left;"><p>Original Target: 1,001,521,152</p></td>
</tr>
<tr>
<td style="text-align: left;"><p>Throughput Achieved:
1,510,322,517</p></td>
<td style="text-align: left;"><p>Throughput Achieved:
775,651,656</p></td>
</tr>
</tbody>
</table>

Compared to our [JAX-WS performance
testing](https://github.com/savoirtech/apache-cxf-soap-performance) our
JAX-RS runs managed to process more calls in total while running on
modest sized heaps.

The PPC64LE system acting as client or server appeared to be less
sensitive to port exhaustion than the X64 machine, however it still
would suffer the same bottleneck.

The X64 system appeared to run out of gas to process more requests (32 →
256 clients yielded very similar results), it would be interesting to
run JVM tunings here to see if there is another bottleneck at play.

## Future Work

There are of course more scenarios we could test, which we intend to
perform in follow-up posts.

- Retest on Java 21 LTS

- Larger Heap spaces

- Adjust thread stack size

# About the Authors

[Jamie
Goodyear](https://github.com/savoirtech/blogs/blob/main/authors/JamieGoodyear.md)

# Reaching Out

Please do not hesitate to reach out with questions and comments, here on
the Blog, or through the Savoir Technologies website at
<https://www.savoirtech.com>.

# With Thanks

Thank you to the Apache CXF community.

\(c\) 2024 Savoir Technologies

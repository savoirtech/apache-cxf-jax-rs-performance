# Apache CXF JAX-RS Performance

We’ve investigated [Apache CXF Soap
Performance](https://github.com/savoirtech/apache-cxf-soap-performance)
in our testing lab, now its time to focus on JAX-RS. We’ll be using the
same systems, so we’ll get to see the through put difference between
these approaches.

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

| Clients | Target Calls/Second per client | Quick Test (Reality) per x64 client | Quick Test (Reality) per PPC64LE client |
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

# Lets get this test case running

To run the performance harness we change directory into samples. Within
this folder we’ll build the base harness and the various scenarios.

On each host we will open a terminal to the CXF distribution samples
folder.

We’ll ensure we have JAVA_HOME and MAVEN_HOME environment variables set.

For our first run we’ll use Adoptium Eclipse Temurin 17 LTS as Client
and Server side JVM.

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
instructions to use get operation, ??? threads (simulate ??? clients),
over a time of 8 hours (60 x 60 x 8 = 28800 seconds).

``` bash
$mvn -Pclient -Dhost=192.168.50.154 -Dprotocol=http -Doperation=get -Dthreads=??? -Dtime=28800
```

For the purposes of our lab test, we’ll allow the suite to execute
without added agents to the JVM.

# Lab Time!

## First Iteration

## Second Iteration

## Third Iteration

# Results and Conclusion

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

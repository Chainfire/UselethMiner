# UselethMiner
Because CPU mining Ethereum is useleth; everybody knows this, *right*?

UselethMiner is an ETHASH CPU miner, and:
* speaks `EthereumStratum/1.0.0`, with automatic failover
* comes with an aggregating (potentially encrypted) proxy you can also use for GPU/ASICs
* runs on Linux (x86-64), Windows (x86-64) and OS X (x86-64 and arm64)
* can auto-scale CPU usage based on machine load, so you can run it in the background
* has JSON-RPC/TCP control
* has a 1% devfee (for both miner and proxy)
* may be profitable
* may be useleth

##### Table of contents

* [About](#about)
* [Benchmarks](#benchmarks)
* [UselethMiner](#uselethminer)
* [UselethProxy](#uselethproxy)
* [Usage](#usage)
    * [Mine](#mine)
    * [Proxy](#proxy)
    * [RPC](#rpc)
    * [Huge pages](#huge-pages)
    * [Known issues](#known-issues)
* [OS-specific notes](#os-specific-notes)
    * [Linux](#linux)
    * [Windows](#windows)
    * [OS X](#os-x)
* [Download](#download)

### About

I've done some work in the Bitcoin world, and lately had to do some Ethereum
things. Finally a reason to *really* dig into it! At some point I was reading
into mining, and found various comments about CPU mining being useless. I 
didn't find any that specified exactly *how* useless though, and my curiosity
was piqued. As I couldn't really find a good CPU miner, I decided to read up
on the algorithm and built one.

So here is my re-implementation of ETHASH. Is it the fastest
CPU-based ETHASH miner? I honestly don't know. I haven't found one even
remotely close to its speed, but to be fair I didn't look *that* hard.

Is it profitable to run? Well, that depends - it *can* be. I wouldn't invest
in large quantities of CPUs to run it; but on equipment that's already
available, under-utilized, and of a well-performing CPU class, and/or if
you have low power costs, you too could earn *cents*. Don't even bother on
Atoms and the likes, though.

Of course, proof-of-work is being phased out on Ethereum very soon, so 
investing in any mining may not be wise anyway.

This a proof-of-concept built for the fun of it. Take it as such. Nobody
expects you (or me) to make anything off of this.

### Benchmarks

CPU | Threads | MT | Watt | MH | W/MH | Notes |
--- | ---: | ---: | ---: | ---: | ---: | :---: |
TR 2950x | 16 | 4x2933 | 115 | 6.1 | 18.9 | [1](#tr-2950x)
i7-7700HQ | 4 | 2x2400 | 9 | 1.1 | 8.2 | [2](#i7-7700hq)
AS M1 | 8 | 8x4266 | 14 | 4.0 | 3.5 | [3](#as-m1)
GKE Xeon | 2 | - | - | 0.6 | - | [4](#gke-xeon)
1080 ti | - | - | 180 | 32.0 | 5.6 | [5](#1080-ti)

More benchmarks are welcome!

###### TR 2950x
* Windows, hugepages, watts measured at PSU

###### i7-7700HQ
* Dell XPS15, Windows, hugepages, watts reported by XTU (excludes DRAM ?), not sure I trust these numbers

###### AS M1
* Apple Mac Mini M1 256/8, OS X, no hugepages, watts reported by powermetrics (includes DRAM)
* While the M1 starts at 4 MH/s, it slowly goes down to 2.8 MH/s, unsure why,
possibly thermal. Runs in the cloud, no physical access, so can't really check 
* powermetrics reported as low as 9 watt usage, sometimes going up to 17, 14 on average
* What if hugepages could boost this even more?
* Only minor tweaks for ARMv8, further (minor?) improvements possible

###### GKE Xeon
* n2-standard-4 class, exact CPU unknown, no hugepages, no power measurement
* Performance varies little with number of threads, likely bandwidth-limited

###### 1080 ti
* TR 2950x, Windows, ethminer, no ethlargementpill, watts measured at PSU

### UselethMiner

Performance is strongly based on your CPU class and memory bandwidth. Faster
memory, particularly with lower latency, directly translates to higher
hashrates, assuming your CPU can keep up as well.

UselethMiner comes with a number of differently optimized backends; which one
produces a higher hashrate depends on your system. It will run a short
benchmark on startup to figure out which one to use. You can skip the
benchmark and force a particular backend to be used with the `--flavor`
option.

The number of threads (cores) you let UselethMiner use to mine obviously also
strongly influences the hashrate. Per-thread hashrate drops off with the
number of threads used, due to how CPU and RAM interact. Additionally, it
generally does not make sense to use your CPU's extra "hyperthreads". The
performance sweet spot is generally around the `physical cores` number of 
threads, not the `logical cores` number of threads. Using the latter will
also grind your system to a near-halt outside of mining, usually for very
little gain in performance. Note that this can happen with fewer threads if
you have slower RAM as well: things tend to start getting noticeably slower
once the miner crosses the 50% of memory bandwidth mark.

UselethMiner can run with a fixed number of threads, but it can also
auto-scale, varying the number of threads between a set min and max, while
trying to keep total system CPU usage below a set percentage. Once your
computer starts doing other things, UselethMiner reduces its threads to
reduce interference with whatever you're wanting to do. The more cores you
have, the better this works. The `1,16,65` setting works well for me on
my 16-core, but you'll have to experiment to find out what works well for
you.

I advise against letting the number of threads drop to 0. As CPU mining
is relatively slow, you may not find a solution the pool accepts for a long
time. Pools only tend to lower the difficulty after each solution submit,
so you need to find a solution before the pool times out your connection,
or it will not register the work your CPU did at all. As the pool lowers
the difficulty in response to your slow submit, you will find solutions
quicker, and you'll actually get paid for your hashrate. If the connection
is dropped (which will happen if the number of threads goes to 0), this
little dance starts all over again. One solution to that is using the proxy
(see below). As a rule of thumb, you need about 1 MH/s to get past this
barrier on initial connection (or some luck).

The devfee is based on the exact number of hashes produced. The devfee is
mined using a single thread, so if you're running multiple threads, only one
of them will be temporarily diverted.

Stats shown in on-screen output are live and accurate.

Note that I've really only tested UselethMiner against its own proxy and the
ViaBTC pool at this point. ViaBTC is picked because their initial connection
difficulty is lower than most, which improves your chances of getting paid.
Note that where you are in the world gets you a different server, so this
may not be the case for you. 

### UselethProxy

UselethMiner can run in proxy mode to aggregate multiple downstream
connections into fewer upstream connections. Both are `EthereumStratum/1.0.0`
based. Each upstream connection can handle at most 256 downstream
connections (see the `--aggr` option), more upstream connections
are created as needed.

Particularly when you're running UselethMiner on multiple machines, you should
be running the proxy somewhere to increase your revenue. The proxy will keep
its upstream connection alive as long as possible, so if one of the miners
restarts or loses connection you don't have to overcome the difficulty
problem mentioned in the [UselethMiner section](#uselethminer) again. Due to
its aggregating nature it also strongly reduces that problem if you connect
multiple miners. Connecting four clients at 0.25 MH/s is just as good as 
connecting a single 1 MH/s client directly to the pool, but the proxy lets
the miners overcome the problem together, rather than each having to overcome
it individually.

A UselethMiner-to-UselethProxy connection is automatically (weakly) encrypted
after initial handshake, as some providers and networks get upset about
stratum connections (I'm looking at you Google Cloud, and your mandatory
monthly support case and timezone-ignoring midnight phone-call!). You can
force UselethMiner to encrypt the handshake as well if you know you're
connecting to a UselethProxy by using a negative port number. You probably
also want to use a common port (like 443) in that case.  

UselethProxy is based on the same code we've used to run Bitcoin pools, which
is known to be able to handle thousands of connections per core. That code
has been adjusted for this project though, and I did not have thousands of
CPUs available to test with, so I really can't say how much it can handle.

For the sake of performance, simplicity, and reduced resource usage, the proxy
is currently *not* a *verifying* proxy. As such it is important you only
connect *well-behaving* miners, as a rejected share can potentially bring
down an entire upstream connection. For this reason the proxy maintains
separate connection pools for UselethMiner clients (which are know to be
well-behaved) and non-UselethMiner clients (I've tested only `ethminer`).

From the a connected miner's perspective, UselethProxy accepts all shares and
rejects none, so you should check accept/reject rates pool-side, not
miner-side.

Downstream connections' submits will be logged, but they will only be
forwarded to the upstream connection if they match the required difficulty.
UselethProxy reduces the difficulty for downstream CPU connections for better
connection tracking, so a large share of submits will *not* actually be valid,
and will *not* be forwarded. This can be confusing if you look at the proxy's
output.   

The devfee is based on difficulty shares. The proxy will divert client
connections to the devfee pool as needed. This is usually just one of the
client connections, though it may become more if you have hundreds of clients
connected.

Stats shown in on-screen output are delayed and calculated back from submitted
share difficulties. They are an approximation that is statistically correct if
averaged long-term with a fixed number of clients with stable hashrate. In
other words, they're not exact numbers, newly connected client may take a
while to be calculated in, newly disconnected clients may still be counted,
etc. This is due to the nature of stratum and is the same reason your pool
hashrates are not exact.

As with mining, the proxy has mostly been tested against the ViaBTC pool. It
will only work properly if the upstream connection you use sends an extranonce
of *at most* two bytes (4 hex characters). You *cannot* connect the proxy
to another proxy! (Well - you can, but it will not work right)

### Usage

Please also read the [OS-specific notes](#os-specific-notes) section before
using.

Run `uselethminer` without any parameters for the full list of options.

#### Mine

UselethMiner requires a machine with at least 7 GB of RAM. You should have
at least 5 GB of free physical RAM (not swap) or you will probably have a
bad time. The RAM requirement grows with the Ethereum DAG.

Your home folder (`~` on Linux/OSX or `%USERPROFILE%` on Windows) should
have at least *double* that in free space *and* be located on an SSD.

You generally want to supply one (or more) stratum connections and a
threading strategy:

`uselethminer STRATUM -t THREAD`

Where `STRATUM` is a comma separated list (for failover) of stratum servers:

`account.worker@server.com:3333` or `account.worker@server.com:3333,account.worker@failover.com:443,account.worker@evenmorefailover.com:80`

And `THREAD` is either a single number for a fixed amount of threads, or
`min,max,cpu` to auto-scale between `min` and `max` threads, trying to stay
below `cpu` % cpu usage.

For example:

`uselethminer account.worker@server.com:3333 -t 1,8,60`

#### Proxy

For proxy mode, you need to supply the local host and port, and the same 
`STRATUM` connection string as with mining mentioned above:

`uselethminer LOCALHOST:LOCALPORT=STRATUM`

For example:

`uselethminer 0.0.0.0:3333=account.worker@server.com:3333`

The proxy is single-threaded. As it's non-verifying and pretty darn fast, 
odds are this is more than enough for you. If you need to go beyond
a single thread, run multiple instances (depending on the OS these can
be bound to the same host and port), use a load-balancer, etc.

#### RPC

Both when mining and serving as proxy, UselethMiner runs an RPC server at
`127.0.0.1:3326` (by default). It speaks basic JSON-RPC over TCP (rather
than over HTTP), the same as the stratum protocol.

You can connect to it with your own code, or run `uselethminer 127.0.0.1:3326`
to launch a console. If you suffix an RPC command to that line (quotes), it
will execute that command and exit the console, for scripting purposes.

The server speaks shortform too, so instead of `{"id": 1, "method": "somecommand", "params": [1, 2, 3]}`
you can just write `somecommand(1, 2, 3)`. The console may also shorten the
response displayed, but if you connect with your own code you will get a
full JSON-RPC reply.
   
Commands for both mining and proxy:

`hello`: greeting, keep-alive

`bye`, `exit`, `quit`: disconnect

`shutdown`: terminate

`connect(username[, password], host, port, [...])`: connect to stratum servers

`connect("username[:password]@host:port[,...]")`: connect to stratum servers

`hashrate`: show hashrate

`status`: show status

Additional commands for mining:

`start`: start mining

`stop`: stop mining

`threads(x)`: use `x` threads

`threads(min, max, cpu)`: auto-scale between `min` and `max` threads, keep
below `cpu` % cpu usage

`hashes`: show hashes

#### Huge pages

UselethMiner attempts to use huge pages if available (see [OS-specific notes](#os-specific-notes)).
On many machines this *doubles performance or more*, so you should care
about getting it to work.

UselethMiner tells you if huge pages are being used every DAG load: it logs
either `(hugepages:yes)` or `(hugepages:no)`.

#### Known issues

You cannot currently mine any other ETHASH chain than Ethereum mainnet. The DAG
will start conflicting and go into a reload loop. 

You can generally Ctrl+C to shutdown the miner, but if you do this during DAG
load/generation, UselethMiner hangs.

While the miner does its best to detect compatible CPUs for each backend flavor,
this is not perfect. It may crash with an illegal instruction, in which case you
should use the `--flavor` option to manually specify a backend to use.

DAG generation is still pretty slow and has room for improvement.

On first run, the benchmark will generate a DAG which will be immediately 
discarded when actual mining starts.

Neither TLS nor EthereumStratum/2.0.0 is currently supported.

### OS-specific notes

#### Linux

UselethMiner must be run from the actual directory its binaries are stored in.

Logs (if enabled) and DAGs are stored in `~/.uselethminer`.

Hugepages are supported, both if reserved and mounted (`hugeadm`, kernel
command line) and with `transparent hugepages (THP)`, the latter both in
`always` and `madvise` mode.

Mining has been tested reasonably well, and Linux is my primary development
and test target for the proxy. I run an Ubuntu 18 Bionic based box, your
mileage may vary on other distros. The proxy attemps to adjust ulimits as
needed.

#### Windows

UselethMiner must be run from the actual directory its binaries are stored in.

Logs (if enabled) and DAGs are stored in `%USERPROFILE%\.uselethminer`.

A `--hide-console` option is available to get rid of the console window if you
want to run it in the background, start it from a script, etc.

Hugepages are supported, but they have to be enabled for your user.
UselethMiner attempts to enable it, but UAC may not let it. On Windows 10 Pro,
you can check/add your user in `Edit group policy (gpedit.msc) -> Local Computer
Policy -> Computer Configuration -> Windows Settings -> Security Settings ->
User Rights Assignment -> Lock pages in memory`. Contrary to Microsoft's
documentation on the matter, you *will need to reboot* your computer for the
setting to take effect. Note that if you only check the policy *after* running
UselethMiner, even if your user appears to be set, you don't know if it was
there before or added by UselethMiner, and you should reboot to be sure. I've
had some issues getting it to work for `Administrator` and groups, but it
seems to work for individual users. There are ways to configure this on 
Windows 10 Home as well according to Google.

Windows 10 is my primary test and development target for the mining part of
UselethMiner, but the proxy part has only seen basic testing. If you want to
run the proxy on Windows - especially with more than a handful of clients - 
you probably want to disable the firewall on the incoming network interface,
exclude the binary from Windows Defender, and all the other things you might
need to do to have a server run properly on Windows.

#### OS X

UselethMiner must be invoked as `/usr/local/uselethminer/uselethminer`.

Logs (if enabled) and DAGs are stored in `~/.uselethminer`.

While UselethMiner attempts to use hugepages on OS X, I have not seen this
actually work at any point on either Intel or ARM platforms.

Only the bare minimum of testing has been done on OS X. I do not have local
OS X hardware, and both using it remotely as from a VM is a real PITA.

### Download

Downloads are available under the [Releases](https://github.com/Chainfire/UselethMiner/releases)
section on GitHub.

Linux and Windows releases can be extracted anywhere and run from there,
OS X releases are packages that install to `/usr/local/uselethminer`.

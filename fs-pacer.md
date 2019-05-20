Copyright 2019(c) [Wael El Oraiby](https://twitter.com/weloraiby), All rights reserved.

**Disclaimer:** Opinions are my own.

I work at [Samsung Ads/Adgear](https://adgear.com/en/). It's an online Ads platform. We sell Ads on the internet (Browser, Mobile, TVs...).

# 5M bid request/s, 2ms Max Response Time - The Road to Damascus
In the worst case, you have 5M bid requests a second hammering your node, each has to be served in less than 2ms. Your node is a 40 (virtual) cores server with at least 2 Gigabit Ethernet cards running Linux. Your system has to make fast decisions on whether it spends money or not to show an Ad. The bidding strategies have to be clear and hopefully bug free, otherwise you end up overspending and bankrupt.

This is a war story...

## Welcome to the Digital/Online Advertisement Business
You fire your browser or you open a mobile app or open your TV, an Ad space will send a request to an online advertisement platform (Samsung/Google/Facebook...). Most likely, the advertisement platform will act as an exchange (although not necessarly), it will run an auction (at Samsung Ads/Adgear, we run a [second-price auction](https://en.wikipedia.org/wiki/Generalized_second-price_auction)) by faning out the request to other advertisers and ask them to bid. The one who bid the most win the space to put an Ad (for second-price auction, the winner pays the second highest price).

Depending on how close your nodes are to the exchange, the time to decide is usually about 20ms with a 120ms for the roundtrip (although, Google can give you up to [300ms](https://developers.google.com/authorized-buyers/rtb/peer-guide)). Through its journey, an Ad bid request grows. It will go through different phases and filters each augmenting the amount of information it holds about you (have you been looking for a car lately?). On, each hop within our system, a database or a microservice is queried. The databases are not ordinary SQL databases, but rather, custom tailored ones, built for low latency and high throughput. At the end of the journey, lies a special microservice: The Pacer.

## The Pacer
The pacer microservice is the governing police at the end of line. Its duty is to ensure no advertisement campaign goes overbudget **and** tries its best to spread the spending accross the life-time of the campaign.

The strategies and the heuristics to spread the budget across time are outside the scope of this article. All to say that they range from the very simple to the complicated (complicated and not complex). What I will be focusing on however, is the architecture and the underlying optimization work.

## Atomics will explode
The early Pacer was written in C and using the infamously complicated [libck](http://concurrencykit.org/) for lock-free data structures. Regardless, the architecture of the old code makes it highly susceptible to race conditions and locks even when using [lock-free data structures](http://www.drdobbs.com/lock-free-data-structures/184401865) (Lock-free data structures rely on [atomic](https://software.intel.com/en-us/node/506090) operations for their functionalities). On a bad day, it was causing some Ad campaign to misbehave on external state updates, making it, literally a ticking atomic counter bomb...

I have hardly seen a real excuse to use lock-free data structures, unless you are doing very low level [kernel](https://lwn.net/Articles/262464/) stuff, a concurrent renderer (Most of I have seen are overengineered and didn't really need it) or your own custom [VM](https://github.com/eloraiby/tcpm) infrastructure (Read that warning twice). Both Erlang & Go don't use them, and they are highly concurrent.

Lock-free data structures and programs are tricky and very hard to debug. To get them right you often need formal proofs that they work or a ton of tests on different CPU architectures. If not, brace yourself for bugs when you try to run your program on a non x64 architecture or change compiler versions (I always test on ARM for their weaker memory model - and so should you).

Most of other problems (including rendering can be solved) with careful benchmark guided architectures.

***Note*** One lockfree renderer I have seen was twice as slow than the original locked one. The reason was again poor architecture choice.

## The irony of fate & state
The original pacer would accept a batch of bid requests, run the strategies and reply back in less than 2ms. It could answer up to a 1M bid req/s. However, all this was done on a single core. While budget changes and configuration reload (that could happen every 5 minutes) was run in parallel. One of the strategies we were using was very susceptible to changes and their order. With bit of bad luck and libck help, it would misbehave and causes overspending requiring manual intervention.

One of the major improvement was phasing out that strategy and make all the strategies stateless or at least less susceptible to instruction ordering. That almost eliminated incidence rate.

With continuous strategy addition, each new one was taking a long time. Each time, special considerations and tests have to be made for the lock-free part. It was clear that the code has reached its end of life and a new rewrite is necessary. A rewrite in a higher level language will make us more productive.

Needing stronger guaranties (type system and immutability by default) and a good support for pattern matching, my choices where Rust, Haskell, OCaml and F#/CoreClr (scala? that's another story for another day).

## Rusty code
My first attempt was to try rust for the rewrite. Rust is a good system language with the strong guarantees. After few months it becomes clear that the ecosystem is very far from being stable and usable. I love the language but I strongly dislike the ecosystem and the geological compilation time.

To this date, rust still lacks time in its standard library. Chrono you say? chrono required the time dependency back then, which in turn was marked as deprecated. And why do I need serde as a dependency? even as optional! Then there is the dependency hell, where the same dependency ends up in your code 4 times because other dependencies use different versions. And yes, you have to build all your dependencies from scratch! (I can only imagine how much time will it take to build Servo).

## Fire the minions
I like both Haskell and OCaml. Sadly Haskell emphasizes on laziness, makes it hard to debug, and the lack of debugging for native generated code was a no go for both Haskell and OCaml (OCaml native debugger is still a [work in progress](https://github.com/ocaml/ocaml/pull/574)).

F# stands out because it's jitted, can offer a strong type system and the tooling is great (if you accept to work with Visual Studio Code). Units of measure proved useful when writing budget related code :)

### Actors and Messages
If you want to be safely concurrent, common wisdom suggests to use the actor model and immutability. If you don't want to end with the [problems of shared variables](https://blog.acolyer.org/2019/05/17/understanding-real-world-concurrency-bugs-in-go/), this should be in the lines of [Erlang](https://en.wikipedia.org/wiki/Erlang_(programming_language)).

While not as sophisticated as Erlang, F# offers mailboxes (the basic building block of actor systems).

The simplified initial architecture looked like (Arrows represent bi-directional messages):
![grid](actor-pacer.png)

***Client:*** A client **actor** represent a connection to the previous node (which we'll call gateway as in gateway to the external world).

***Processor:*** It's where a strategy is applied to each incoming bid request, then answer back. Changes in budget or Ad campaigns get sent (as a message) to the processor from the watcher.

***Watcher:*** This actor will just loop waiting for a budget change, or Ad campaigns changes, and then synchronize the final state to the processors.

That's a simple design, communication is done with message passing and each actor keeps its own state. Everything will go smooth...I hoped.

At this point other team members joined the effort. They started deploying on test nodes...

... and few days later came the surprize! benchmarks are showing very poor throughput (maxing around 320k bid req/s using all 40 cores!).

Time to [profile](https://codeblog.dotsandbrackets.com/profiling-net-core-app-linux/)!

On the test machine:

```
$ ps aux | grep fs-pacer
user      5028  472  0.7 4901532 183796 pts/2  SLl+ 11:20   1:25 dotnet exec /local/pacer/fs-pacer/bin/Release/netcoreapp2.2/fs-pacer.dll
$ sudo ./perfcollect collect session -pid 5028
$ sudo perf script -f | FlameGraph/stackcollapse-perf.pl | FlameGraph/flamegraph.pl > flamegraph.svg
```

First benchmarks showed that most of the time spent was in socket connections. The other F# developer in the team gave [Pipelines](https://devblogs.microsoft.com/dotnet/system-io-pipelines-high-performance-io-in-net/) a try. Throughput went up more than 30% to 432k bid req/s. He went on translating the pacer and the testing/hammering tool from F# to Golang, while I continue profiling.

`htop` showed that all threads are never saturated, maximizing at around 54% usage per core. And a lot of hopping was happening.

Examining perf data with `perf report` and the flamegraph gave the following CPU consumptions:
1. 18.1% on `CLRLifoSemaphore::Wait`.
2. 11.35% parsing requests with whooping 7% on `JIT_NewArr1` allocating strings for pattern matching.
3. 10.51% `System.Net.Sockets.dll`.
4. 1.1% `ThreadPoolNative::NotifyRequestComplete`.
5. 0.6% `ThreadPoolNative::RequestWorkerThread`.

Here's the flamegraph with all these in purple (41.9% of CPU time):

![flame1](flamegraph-no-optim.png)

I made another attempt to reduce synchronization time. This time merging both processors and clients together. Each client actor will be its own processor. This also yielded about 48% more throughput, achieving 640k bid req/s. At the same time, Golang was able to handle about 2.8M bid req/s using the old architecture. So, still not Enough!

Examining CoreFX [code](https://github.com/dotnet/corefx/blob/27dae83598c87e3cf4b139c8c981c13fe8e9a81e/src/System.Net.Sockets/src/System/Net/Sockets/SocketAsyncEngine.Unix.cs#L303), showed that linux epoll loop runs in a single thread. An event will be dispatched afterward, to other tasks to consume it. This creates needless waiting time.

The least effort for maximized value return would be a native module for network handling and bidrequest parsing.

## C and thy shall receive
I didn't want to rewrite everything from scratch, and definitely, I didn't want to handle all edge cases for epoll. My choice was to use [libuv](https://libuv.org/). The architecture I opt for was to use 16 cores out of 40 for networking, having 16 `uv_loop` each running on its own thread. Callbacks will be passed from F# to each `uv_loop` instance. The event loop will call them after parsing the bid request in C11. After 900 lines of C11 code, the throughput ranged from 3.7M to 5.2M bid req/s wihout the need of thread pinning.

And the final flamegraph (in purple is the time to handle a req) showing that the time spent in F# is very minimal (CoreCLR, you'r good :)
![optim](flamegraph-optim.png)

## Conclusion
CoreFX async sockets under linux (with coreclr 2.2.5) are not on par with other frameworks, and definitely don't match MS Windows performance. While Golang achieved between 2.8M & 3.2M bid req/s out of the box, CoreFX was busy waiting...

Finally: Microsoft! we'd like to see more investment in CoreFX for Linux.

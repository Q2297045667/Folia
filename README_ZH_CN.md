<div align=center>
    <img src="./folia.png">
    <br /><br />
    <p> 将区域化多线程添加到专用服务器，的<a href="https://github.com/PaperMC/Paper">Paper</a> Fork</p>
</div>

## Overview

Folia将附近加载的区块分组，形成一个“独立区域”
参见 [PaperMC文档](https://docs.papermc.io/folia/reference/region-logic) 有关Folia的详细信息
将对附近的区块进行分组，每个独立的区域都有自己的tick循环
，在常规的Minecraft tickrate（20TPS）中勾选。
执行tick循环，常规的Minecraft tickrate（20TPS）。
执行tick循环 并行地在线程池上。
不再有主线，因为每个区域实际上都有自己的“主线程”执行整个tick循环。

未来Folia不会合并到Paper，这是个私人项目
当初Starlight项目好像也是这么说的，最后还不是屈服于Paper的神威（

更详细但抽象的概述: [项目概述](https://docs.papermc.io/folia/reference/overview).

## FAQ

### 哪些服务器类型可以从Folia中获益?
玩家数量多且自然分散的服务器类型
比如王国或者城镇，空岛类和多人生存类服务器

### Folia在什么硬件上运行得最好?
Folia对主频需求不是很高。
最好16个核心以上（注意是核心，不是线程）
像EPYC或者志强系列都可以很好的运行上百人的服务器

### 如何最好地配置Folia?
建议预先生成世界，这样所需的区块系统工作的线程数量会大大减少。

以下是基于第一次EOYC测试的粗略估计
在Folia在测试服务器上发布之前
只有330名玩家，所以他并不准确

第二次基于7950x3d测试结果
500人整体的可以稳定在TPS19以上
大多数区域可以稳定在20

The total number of cores on the machine available should be 
taken into account. Then, allocate threads for: 
- netty IO :~4 per 200-300 players
- chunk system io threads: ~3 per 200-300 players
- chunk system workers if pre-generated, ~2 per 200-300 players
- There is no best guess for chunk system workers if not pre-generated, as
  on the test server we ran we gave 16 threads but chunk generation was still
  slow at ~300 players.
- GC Settings: ???? But, GC settings _do_ allocate concurrent threads, and you need
  to know exactly how many. This is typically through the `-XX:ConcGCThreads=n` flag. Do not
  confuse this flag with `-XX:ParallelGCThreads=n`, as parallel GC threads only run when
  the application is paused by GC and as such should not be taken into account.

After all of that allocation, the remaining cores on the system until 80%
allocation (total threads allocated < 80% of cpus available) can be
allocated to tickthreads (under global config, threaded-regions.threads). 

The reason you should not allocate more than 80% of the cores is due to the
fact that plugins or even the server may make use of additional threads 
that you cannot configure or even predict.

Additionally, the above is all a rough guess based on player count, but
it is very likely that the thread allocation will not be ideal, and you 
will need to tune it based on usage of the threads that you end up seeing.

## Plugin compatibility

There is no more main thread. I expect _every_ single plugin
that exists to require _some_ level of modification to function
in Folia. Additionally, multithreading of _any kind_ introduces
possible race conditions in plugin held data - so, there are bound
to be changes that need to be made.

So, have your expectations for compatibility at 0.

## API plans

Currently, there is a lot of API that relies on the main thread. 
I expect basically zero plugins that are compatible with Paper to 
be compatible with Folia. However, there are plans to add API that 
would allow Folia plugins to be compatible with Paper.

For example, the Bukkit Scheduler. The Bukkit Scheduler inherently
relies on a single main thread. Folia's RegionScheduler and Folia's
EntityScheduler allow scheduling of tasks to the "next tick" of whatever
region "owns" either a location or an entity. These could be implemented
on regular Paper, except they schedule to the main thread - in both cases,
the execution of the task will occur on the thread that "owns" the
location or entity. This concept applies in general, as the current Paper
(single threaded) can be viewed as one giant "region" that encompasses
all chunks in all worlds. 

It is not yet decided whether to add this API to Paper itself directly
or to Paperlib.

### The new rules

First, Folia breaks many plugins. To aid users in figuring out which
plugins work, only plugins that have been explicitly marked by the
author(s) to work with Folia will be loaded. By placing
"folia-supported: true" into the plugin's plugin.yml, plugin authors
can mark their plugin as compatible with regionised multithreading.

The other important rule is that the regions tick in _parallel_, and not 
_concurrently_. They do not share data, they do not expect to share data,
and sharing of data _will_ cause data corruption. 
Code that is running in one region under no circumstance can 
be accessing or modifying data that is in another region. Just 
because multithreading is in the name, it doesn't mean that everything 
is now thread-safe. In fact, there are only a _few_ things that were 
made thread-safe to make this happen. As time goes on, the number 
of thread context checks will only grow, even _if_ it comes at a 
performance penalty - _nobody_ is going to use or develop for a 
server platform that is buggy as hell, and the only way to 
prevent and find these bugs is to make bad accesses fail _hard_ at the 
source of the bad access.

This means that Folia compatible plugins need to take advantage of 
API like the RegionScheduler and the EntityScheduler to ensure 
their code is running on the correct thread context.

In general, it is safe to assume that a region owns chunk data
in an approximate 8 chunks from the source of an event (i.e. player
breaks block, can probably access 8 chunks around that block). But,
this is not guaranteed - plugins should take advantage of upcoming
thread-check API to ensure correct behavior.

The only guarantee of thread-safety comes from the fact that a
single region owns data in certain chunks - and if that region is
ticking, then it has full access to that data. This data is 
specifically entity/chunk/poi data, and is entirely unrelated
to **ANY** plugin data.

Normal multithreading rules apply to data that plugins store/access
their own data or another plugin's - events/commands/etc. are called 
in _parallel_ because regions are ticking in _parallel_ (we CANNOT 
call them in a synchronous fashion, as this opens up deadlock issues 
and would handicap performance). There are no easy ways out of this, 
it depends solely on what data is being accessed. Sometimes a 
concurrent collection (like ConcurrentHashMap) is enough, and often a 
concurrent collection used carelessly will only _hide_ threading 
issues, which then become near impossible to debug.

### Current API additions

To properly understand API additions, please read
[Project overview](https://docs.papermc.io/folia/reference/overview).

- RegionScheduler, AsyncScheduler, GlobalRegionScheduler, and EntityScheduler 
  acting as a replacement for  the BukkitScheduler.
  The entity scheduler is retrieved via Entity#getScheduler, and the
  rest of the schedulers can be retrieved from the Bukkit/Server classes.
- Bukkit#isOwnedByCurrentRegion to test if the current ticking region
  owns positions/entities

### Thread contexts for API

To properly understand API additions, please read
[Project overview](https://docs.papermc.io/folia/reference/overview).

General rules of thumb:

1. Commands for entities/players are called on the region which owns
the entity/player. Console commands are executed on the global region.

2. Events involving a single entity (i.e player breaks/places block) are
called on the region owning entity. Events involving actions on an entity
(such as entity damage) are invoked on the region owning the target entity.

3. The async modifier for events is deprecated - all events
fired from regions or the global region are considered _synchronous_, 
even though there is no main thread anymore. 

### Current broken API

- Most API that interacts with portals / respawning players / some
  player login API is broken.
- ALL scoreboard API is considered broken (this is global state that
  I've not figured out how to properly implement yet)
- World loading/unloading
- Entity#teleport. This will NEVER UNDER ANY CIRCUMSTANCE come back, 
  use teleportAsync
- Could be more

### Planned API additions

- Proper asynchronous events. This would allow the result of an event
  to be completed later, on a different thread context. This is required
  to implement some things like spawn position select, as asynchronous
  chunk loads are required when accessing chunk data out-of-region.
- World loading/unloading
- More to come here

### Planned API changes

- Super aggressive thread checks across the board. This is absolutely
  required to prevent plugin devs from shipping code that may randomly
  break random parts of the server in entirely _undiagnosable_ manners.
- More to come here

### Maven information
* Maven Repo (for folia-api):
```xml
<repository>
    <id>papermc</id>
    <url>https://repo.papermc.io/repository/maven-public/</url>
</repository>
```
* Artifact Information:
```xml
<dependency>
    <groupId>dev.folia</groupId>
    <artifactId>folia-api</artifactId>
    <version>1.20.1-R0.1-SNAPSHOT</version>
    <scope>provided</scope>
</dependency>
 ```


## License
The PATCHES-LICENSE file describes the license for api & server patches,
found in `./patches` and its subdirectories except when noted otherwise.

The fork is based off of PaperMC's fork example found [here](https://github.com/PaperMC/paperweight-examples).
As such, it contains modifications to it in this project, please see the repository for license information
of modified files.

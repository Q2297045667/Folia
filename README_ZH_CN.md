<div align=center>
    <img src="./folia.png">
    <br /><br />
    <p> 将区域化多线程添加到专用服务器，的<a href="https://github.com/PaperMC/Paper">Paper</a> Fork</p>
</div>

## 概述

Folia将附近加载的区块分组，形成一个“独立区域”。请参阅[PaperMC文档](https://docs.papermc.io/folia/reference/region-logic)了解Folia将如何对附近的区块进行分组的详细细节。

每个独立区域都有自己的tick循环，在常规的Minecraft（20TPS）中tick。

tick循环在线程池上并行执行。

服务端不再有主线程，因为每个区域实际上都有自己的“主线程”来执行整个tick循环。

Spottedleaf说未来Folia不会合并到Paper，这是个私人项目，当初Starlight项目好像也是这么说的，最后还不是屈服于PaperMC的神威（

更详细但抽象的概述: [项目概述](https://docs.papermc.io/folia/reference/overview).

## 问答

### 哪些服务器类型可以从Folia中获益?
玩家数量多且自然分散的服务器类型，比如王国或者城镇，空岛类和多人生存类服务器

### Folia在什么硬件上运行得最好?
Folia对主频需求不是很高。最好16个核心以上（注意是核心，不是线程）每个核心应不低于3.5Ghz主频，像EPYC或者志强系列的大多数CPU都可以很好的运行上百人的服务器

### 如何最好地配置Folia?
建议预先生成世界，这样所需的区块系统工作的线程数量会大大减少。

以下是基于第一次EPYC 7713测试的粗略估计在Folia在测试服务器上发布之前只有330名玩家，所以他并不准确 [数据来源](https://paper-chan.moe/folia/#records).

第二次基于7950x3d的测试结果粗略估计500人整体可以稳定在TPS19以上大多数区域可以稳定在20 [数据来源](https://cubxity.dev/blog/folia-test-june-2023).

在1000人的时候TPS大多在15左右但是服务端进行了多项调优改动，因此不应该作为参考。

PS：以上的测试环境为玩家分散到每个区块分组的测试结果

应考虑机器上可用的物理核心总数（非超线程），然后为以下对象分配线程：

- 网络IO：每4个线程约200-300名玩家（spigot.yml文件settings: netty-threads: 4）

- 区块系统IO：每3个线程约200-300名玩家（paper-global.yml文件chunk-system: io-threads: 3）

- 区块系统读写IO（预先生成）:每2个线程200-300名玩家（paper-global.yml文件chunk-system: worker-threads: 2）

如果不是预先生成的，那么区块系统工作就没有很好的预测，因为在我们运行300人的测试服务器上，分配了16个线程，但区块生成仍然很慢。

- GC设置：？？？？但是，GC设置确实会分配并发线程，您需要确切地知道有多少个。

这通常是通过`-XX:ConcGCThreads=n`标志实现的,不要将此标志与`-XX:AParallelGCThreads=n`混淆，因为并行GC线程仅在GC暂停应用程序时运行，因此不应被考虑在内。

你不应该分配超过80%的物理核心的原因是，插件甚至系统可能会使用这些额外线程用来计算其他服务。

以上都是基于玩家数量的粗略猜测，但线程分配很可能并不理想，您需要根据最终看到的线程使用情况对其进行调整。

## 插件兼容性

现有的每个插件都需要进行一定程度的修改才能在Folia中运行。
此外，任何类型的多线程都会在插件保存的数据中引入可能的资源竞争，因此，必然需要进行更改。

如果你在寻找能够兼容Folia的插件那么我这里有几份兼容性列表
[folia-plugins](https://github.com/BlockhostOfficial/folia-plugins)
[modrinth](https://modrinth.com/plugins?g=categories:%27folia%27&g=categories:folia)

## API计划

目前，有很多API依赖于主线程。

我希望基本上没有与Paper兼容的插件与Folia兼容，有计划添加API，允许Folia插件与Paper兼容。

例如，Bukkit调度器。

Bukkit Scheduler固有地依赖于单个主线程。Folia的RegionScheduler和Folia的EntityScheduler允许将任务调度到“拥有”位置或实体的任何区域的"next tick"。

这些可以在常规的Paper上实现，除非它们调度到主线程 在这两种情况下，任务的执行都将发生在“拥有”位置或实体的线程上。

这一概念普遍适用，因为当前的Paper（单线程）可以被视为一个包含所有世界中所有块的巨大“区域”。

目前尚未决定是否将此API直接添加到Paper本身或发送到Paperlib。

### 新的规则

首先，Folia破坏了许多插件API。

为了帮助用户了解哪些插件有效，只加载作者明确标记过为与Folia兼容的插件。

通过将“folia-supported:true”放入插件的plugin.yml中，插件作者可以将他们的插件标记为与Folia兼容。

另一个重要规则是，区域在_paralle_中tick，而不是_concurrently_。
而且他们不共享数据，也不希望共享数据，共享数据会导致数据损坏和线程占用的情况。
在任何情况下，在一个区域中运行的代码都不能访问或修改另一个区域的数据。
仅仅因为多线程在名称中，但是这并不意味着一切线程操作都是安全的。
事实上，只有一个新的东西是线程安全的，才能实现这一点。
随着时间的推移，线程上下文检查的数量只会增加，即使它会带来性能损失——_nobody将使用或开发一个有漏洞的服务器平台，而预防和发现这些漏洞的唯一方法是在错误访问的源头使错误访问失败。

这意味着与Folia兼容的插件需要利用像RegionScheduler和EntityScheduler这样的API，以确保它们的代码在正确的线程上下文上运行。

一般来说，可以安全地假设一个区域拥有来自事件源的大约8个块中的块数据（即玩家打破块，可能可以访问该块周围的8个块）。

但是，这并不能保证——插件应该利用即将推出的线程检查API来确保正确的行为。

线程安全的唯一保证来自这样一个事实，即单个区域拥有某些区块中的数据，
如果该区域正在运行，那么它就可以完全访问这些数据。
此数据具体是实体/区块/poi数据，与**任何**插件数据完全无关。

正常的多线程规则适用于插件存储/访问自己的数据或另一个插件的数据的事件/命令等在_paralle_中调用，因为区域在_paralel_中tick（我们不能以同步方式调用它们，因为这会导致线程死锁并会影响性能）。
没有简单的方法可以解决这个问题，这完全取决于正在访问的数据。
有时并发集合（如ConcurrentHashMap）就足够了，而且经常不小心使用并发集合只会发现线程问题，这几乎不可能调试的。

### API当前新增的内容

要正确理解API添加内容，请阅读[Project overview](https://docs.papermc.io/folia/reference/overview).

- RegionScheduler, AsyncScheduler, GlobalRegionScheduler, and EntityScheduler 
  acting as a replacement for  the BukkitScheduler.
  The entity scheduler is retrieved via Entity#getScheduler, and the
  rest of the schedulers can be retrieved from the Bukkit/Server classes.
- Bukkit#isOwnedByCurrentRegion to test if the current ticking region
  owns positions/entities

### API的线程上下文

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

### 当前中断/遗弃的API

- Most API that interacts with portals / respawning players / some
  player login API is broken.
- ALL scoreboard API is considered broken (this is global state that
  I've not figured out how to properly implement yet)
- World loading/unloading
- Entity#teleport. This will NEVER UNDER ANY CIRCUMSTANCE come back, 
  use teleportAsync
- Could be more

### 计划新增的API

- Proper asynchronous events. This would allow the result of an event
  to be completed later, on a different thread context. This is required
  to implement some things like spawn position select, as asynchronous
  chunk loads are required when accessing chunk data out-of-region.
- World loading/unloading
- More to come here

### API计划变更

- Super aggressive thread checks across the board. This is absolutely
  required to prevent plugin devs from shipping code that may randomly
  break random parts of the server in entirely _undiagnosable_ manners.
- More to come here

### Maven信息
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


## 许可证
The PATCHES-LICENSE file describes the license for api & server patches,
found in `./patches` and its subdirectories except when noted otherwise.

The fork is based off of PaperMC's fork example found [here](https://github.com/PaperMC/paperweight-examples).
As such, it contains modifications to it in this project, please see the repository for license information
of modified files.

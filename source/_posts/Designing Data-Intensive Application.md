---
title: 读 Designing Data-Intensive Application
date: 2019-02-13 16:57:48
tags: 阅读
categories: [设计思想]
---

## 前言
> 拜读大佬的书籍，把重点的语句做个笔记。
---
书本内容：
## 序言
>1. 近些年的软件工程，特别是服务器端和后端系统开发，大量关于数据存储和处理的时髦词汇涌现出来： **NoSQL！大数据！Web-Scale！分片！最终一致性！
ACID！ CAP定理！云服务！MapReduce！实时！**
2. 而**数据密集型应用**（data-intensive applications）正在通过使用这些技术进步来推动可能性的边界。当一个应用被称为**数据密集**型的引用，
如果数据是其主要挑战（数据量，数据复杂度或数据变化速度）—— 与之相对的是**计算密集型**，即**处理器速度**是其瓶颈。
3. 在技术迅速变化的背后总是存在一些持续成立的原则，无论我们使用了特定工具的哪个版本。如果理解了这些原则，就可以领会这些工具的适用场景，如何充分
利用它们，以及如何避免其中的陷阱.
## 第一部分：数据系统的基石
1. 第一章将介绍本书使用的术语和方法。可靠性，可扩展性和可维护性 ，这些词汇到底意味着什么？如何实现这些目标？
2. 第二章将对几种不同的数据模型和查询语言进行比较。从程序员的角度看，这是数据库之间最明显的区别。不同的数据模型适用于不同的应用场景。
3. 第三章将深入存储引擎内部，研究数据库如何在磁盘上摆放数据。不同的存储引擎针对不同的负载进行优化，选择合适的存储引擎对系统性能有巨大影响。
4. 第四章将对几种不同的 数据编码进行比较。特别研究了这些格式在应用需求经常变化、模式需要随时间演变的环境中表现如何。

### 第一章：可靠性可，扩展性，可维护性
- 现今很多应用程序都是 **数据密集型（data-intensive）** 的，而非 **计算密集型（compute-intensive）** 的。因此CPU很少成为这类应用的瓶颈，
更大的问题通常来自数据量、数据复杂性、以及数据的变更速度。一旦系统的负载被描述好，就可以研究当负载增加会发生什么。我们可以从两种角度来看：  
    - 增加负载参数并保持系统资源（CPU、内存、网络带宽等）不变时，系统性能将受到什么影响？
    - 增加负载参数并希望保持性能不变时，需要增加多少系统资源？
这两个问题都需要性能数据，所以让我们简单地看一下如何描述系统性能。
​对于Hadoop这样的**批处理系统**，通常关心的是**吞吐量**（throughput），即每秒可以处理的记录数量，或者在特定规模数据集上运行作业的总时间。
对于在线系统，通常更重要的是服务的**响应时间**（response time），即客户端发送请求到接收响应之间的时间。
使不断重复发送同样的请求，每次得到的响应时间也都会略有不同。现实世界的系统会处理各式各样的请求，响应时间可能会有很大差异。因此我们需要将
**响应时间**视为一个可以测量的**数值分布**（distribution），**而不是单个数值**

### 第二章：数据模型与查询语言
- 数据模型主要分为两种：**关系模型**(sql,典型的如mysql,oracle)与**文档模型**(noSql[Not Only SQL],典型的如mongo) 
- 采用NoSQL数据库的背后有几个驱动因素，其中包括：
  - 需要比关系数据库更好的可扩展性，包括非常大的数据集或非常高的写入吞吐量
  - 相比商业数据库产品，免费和开源软件更受偏爱。
  - 关系模型不能很好地支持一些特殊的查询操作
  - 受挫于关系模型的限制性，渴望一种更具多动态性与表现力的数据模型
- 文档和关系数据库的融合:自2000年代中期以来，大多数关系数据库都已经开始支持文档结构：如json，xml格式的数据
- 随着时间的推移，关系数据库和文档数据库似乎变得越来越相似，这是一件好事：数据模型相互补充 ，如果一个数据库能够处理类似文档的数据，
并能够对其执行关系查询，那么应用程序就可以使用最符合其需求的功能组合。
- 关系模型和文档模型的混合是未来数据库一条很好的路线。

- 文档数据库有时称为无模式（schemaless），但这具有误导性，因为读取数据的代码通常假定某种结构——即存在隐式模式，但不由数据库强制执行。
一个更精确的术语是读时模式（schema-on-read）（数据的结构是隐含的，只有在数据被读取时才被解释），相应的是写时模式（schema-on-write）
（传统的关系数据库方法中，模式明确，且数据库确保所有的数据都符合其模式）
-  模式变更的速度很慢，而且要求停运。它的这种坏名誉并不是完全应得的：大多数关系数据库系统可在几毫秒内执行 ALTER TABLE  语句。
MySQL是一个值得注意的例外，它执行 ALTER TABLE  时会复制整个表，这**可能意味着在更改一个大型表时会花费几分钟甚至几个小时的停机
时间，尽管存在各种工具来解决这个限制**。
- 大型表上运行 UPDATE 语句在任何数据库上都可能会很慢，因为每一行都需要重写。要是不可接受的话，应用程序可以将 first_name  设置为默认值 NULL
  ，并在读取时再填充，就像使用文档数据库一样。
> SQL是一种声明式查询语言，而IMS和CODASYL使用命令式代码来查询数据库。那是什么意思？许多常用的编程语言是命令式的。例如，给定一个动物物种的列表，
返回列表中的鲨鱼可以这样写：
function getSharks() {
var sharks = [];
for (var i = 0; i < animals.length; i++) {
if (animals[i].family === "Sharks") {
sharks.push(animals[i]);
}
}
return sharks;
}
命令式语言告诉计算机以特定顺序执行某些操作。可以想象一下，逐行地遍历代码，评估条 件，更新变量，并决定是否再循环一遍。
在声明式查询语言（如SQL或关系代数）中，你只需指定所需数据的模式 - 结果必须符合哪些 条件，以及如何将数据转换（例如，排序，分组和集合）
但不是如何实现这一目标。数据库 系统的查询优化器决定使用哪些索引和哪些连接方法，以及以何种顺序执行查询的各个部分。

- 声明式查询语言是迷人的，因为它通常比命令式API更加简洁和容易。但更重要的是，它还隐
藏了数据库引擎的实现细节，这使得数据库系统可以在无需对查询做任何更改的情况下进行
性能提升。  

- 最后，声明式语言往往适合并行执行。现在，CPU的速度通过内核的增加变得更快，而不是
以比以前更高的时钟速度运行【31】。命令代码很难在多个内核和多个机器之间并行化，因
为它指定了指令必须以特定顺序执行。声明式语言更具有并行执行的潜力，因为它们仅指定
结果的模式，而不指定用于确定结果的算法。在适当情况下，数据库可以自由使用查询语言
的并行实现

- 我们会研究两大类存储引擎：日志结构（log-structured）的存储引擎，以及面向页面（page-oriented）的存储引擎（例如B树）。
- 冻结段的合并和压缩可以在后台线程中完成，在进行时，我们仍然可以继续使用旧的段文件来正常提供读写请求。合并过程完成后，我们将读取请求转换为
使用新的合并段 而不是旧段然后可以简单地删除旧的段文件。
- **简而言之，一些真正实施中重要的问题是:**
  - **文件格式**:
  - **删除记录**:
  - **崩溃恢复**:
  - **部分写入记录**:
  - **并发控制**: 乍一看，只有追加日志看起来很浪费：为什么不更新文件，用新值覆盖旧值？但是只能追加设计的原因有几个:
    - 追加和分段合并是顺序写入操作，通常比随机写入快得多，尤其是在磁盘旋转硬盘上。
    - 在某种程度上，顺序写入在基于闪存的固态硬盘（SSD）上也是优选的【4】。我们将在
    - 第83页的“比较B-树和LSM-树”中进一步讨论这个问题。
    - 如果段文件是附加的或不可变的，并发和崩溃恢复就简单多了。例如，您不必担心在覆盖值时发生崩溃的情况，而将包含旧值和新值的一部分的文件保留在一起。
    - 合并旧段可以避免数据文件随着时间的推移而分散的问题。  
- 与往常一样，大量的细节使得存储引擎在实践中表现良好。例如，当查找数据库中**不存在的键**时，LSM树算法可能会很慢：您必须检查内存表，然后将这些段一
直回到最老的（这可能必须从磁盘读取每一个），然后才能确定键不存在。为了优化这种访问，存储引擎通常使用额外的Bloom过滤器
- 布隆过滤器是用于近似集合内容的内存高效数据结构，它可以告诉您数据库中是否出现键，从而为不存在的键节省许多不必要的磁盘读取操作。
  (布隆过滤有这样的特性：没有那就一定是没有，有，有百分之95%以上的几率是有的，用很小的错误率，和很小的空间获得特别大的效率)
-  即使有许多微妙的东西，LSM树的基本思想 —— 保存一系列在后台合并的SSTables —— 简 单而有效。即使数据集比可用内存大得多，它仍能继续正常工作。
由于数据按排序顺序存储， 因此可以高效地执行范围查询（扫描所有高于某些最小值和最高值的所有键），并且因 为磁盘写入是连续的，所以LSM树可以支持
非常高的写入吞吐量。
- 在B树的一个页面中对子页面的引用的数量称为分支因子。例如，在图3-6中，分支因子是 6 。在实践中，分支因子取决于存储页面参考和范围边界所需的空间量，
但通常是几百个。
- 日志结构化的方法在这方面更简单，因为它们在后台进行所有的合并，而不会干扰传入的查询，并且不时地将旧的分段原子交换为新的分段。
- 这种差异在磁性硬盘驱动器上尤其重要，顺序写入比随机写入快得多。
- 在许多关系数据库中，事务隔离是通过在键范围上使用锁来实现的，在B树索引中，这些锁可以直接连接到树
- 在新的数据存储中，**日志结构化索引**变得越来越流行。没有快速和容易的规则来确定哪种类型的存储引擎对你的场景更好，所以值得进行一些经验上的测试.
- 索引中的关键字是查询搜索的内容，但是该值可以是以下两种情况之一：它可以是所讨论的实际行（文档，顶点），也可以是对存储在别处的行的引用。在后一种
情况下，行被存储的 地方被称为堆文件（heap file），并且存储的数据没有特定的顺序（它可以是仅附加的，或 者可以跟踪被删除的行以便用新数据覆盖它们后来）
- 在聚集索引（在索引中存储所有行数据）和非聚集索引（仅在索引中存储对数据的引用）之间的折衷被称为包含列的索引或覆盖索引，其存储表的一部分在索引内。
这允许通过单独使用索引来回答一些查询（这种情况叫做：索引覆盖（cover）了查询）
- 某些内存中的键值存储（如Memcached）仅用于缓存，在重新启动计算机时丢失的数据是可 以接受的。但其他内存数据库的目标是持久性，可以通过特殊的硬件
（例如电池供电的 RAM），将更改日志写入磁盘，将定时快照写入磁盘或通过复制内存来实现，记忆状态到其他机器。
>诸如VoltDB，MemSQL和Oracle TimesTen等产品是具有关系模型的内存数据库，供应商声称，通过消除与管理磁盘上的数据结构相关的所有开销，
他们可以提供巨大的性能改进【41,42】。 RAM Cloud是一个开源的内存键值存储器，具有持久性（对存储器中的数据以及磁盘上的数据使用日志结构化方法）
【43】。 Redis和Couchbase通过异步写入磁盘提供了较弱的持久性。
 
>磁盘的顺序读速度能打到在100MB/S上下使用列式存储，分析的时候，可以只扫描需要的那部分数据的时候，减少CPU和磁盘的访问量。同时面向列的存储通常
很适合压缩，使用压缩，可以综合CPU和磁盘，发挥最大的效能。
>面向行的存储将每一行保存在一个地方（在堆文件或聚簇索引中）
> 在OLTP方面，我们看到了来自两大主流学派的存储引擎： 日志结构学派
> 只允许附加到文件和删除过时的文件，但不会更新已经写入的文件。 Bitcask，SSTables，LSM树，LevelDB，Cassandra，HBase，Lucene等都属于这个组。 
原地更新学派将磁盘视为一组可以覆盖的固定大小的页面。 B树是这种哲学的最大的例子，被用在所有主要的关系数据库中，还有许多非关系数据库。
 
### 第三章：存储与检索
> 日志结构的存储引擎是相对较新的发展。他们的主要想法是，他们系统地将随机访问写入顺
> 序写入磁盘，由于硬盘驱动器和固态硬盘的性能特点，可以实现更高的写入吞吐量。在完成
> OLTP方面，我们通过一些更复杂的索引结构和为保留所有数据而优化的数据库做了一个简短
> 的介绍。 然后，我们从存储引擎的内部绕开，看看典型数据仓库的高级架构。这一背景说明了为什么
> 分析工作负载与OLTP差别很大：当您的查询需要在大量行中顺序扫描时，索引的相关性就会
> 降低很多。相反，非常紧凑地编码数据变得非常重要，以最大限度地减少查询需要从磁盘读
> 取的数据量。我们讨论了列式存储如何帮助实现这一目标。
> 作为一名应用程序开发人员，如果您掌握了有关存储引擎内部的知识，那么您就能更好地了
> 解哪种工具最适合您的特定应用程序。 如果您需要调整数据库的调整参数，这种理解可以让
> 您设想一个更高或更低的值可能会产生什么效果。
> 尽管本章不能让你成为一个特定存储引擎的调参专家，但它至少有大概率使你有了足够的概
> 念与词汇储备去读懂数据库的文档，从而选择合适的数据库。

>
- 作为程序员，为什么要关心数据库内部存储与检索的机理？你可能不会去从头开始实现自己的存储引擎，但是你确实需要从许多可用的存储引擎中选择一个合适的。
而且为了协调存储引擎以适配应用工作负载，你也需要大致了解存储引擎在底层究竟做什么。
- 特别需要注意，针对事务性负载和分析性负载优化的存储引擎之间存在巨大差异。稍后我们
  将在 “事务处理还是分析？” 一节中探讨这一区别，并在 “列存储”中讨论一系列针对分析优化存储引擎。   
>世界上最简单的数据库可以用两个Bash函数实现：
> \#!/bin/bash
> db_set () {
 echo "$1,$2" >> database
 }
 db_get () {
 grep "^$1," database | sed -e "s/^$1,//" | tail -n 1
 }
 这两个函数实现了键值存储的功能。执行  db_set key value  ，会将 键（key）和值
 （value） 存储在数据库中。
   
- db_set
  函数对于极其简单的场景其实有非常好的性能，因为在文件尾部追加写入通常是非常高效的。与
  db_set 做的事情类似，许多数据库在内部使用了日志（log），也就是一个仅追
  加（append-only）的数据文件。真正的数据库有更多的问题需要处理（如并发控制，回收
  磁盘空间以避免日志无限增长，处理错误与部分写入的记录），但基本原理是一样的。日志
  极其有用，我们还将在本书的其它部分重复见到它好几次。
>日志（log）这个词通常指应用日志：即应用程序输出的描述发生事情的文本。本书在更
 普遍的意义下使用日志这一词：一个仅追加的记录序列。它可能压根就不是给人类看
 的，使用二进制格式，并仅能由其他程序读取。 
 
- 索引背后的大致思想是，保存一些额外的元数据作为路
  标，帮助你找到想要的数据。如果您想在同一份数据中以几种不同的方式进行搜索，那么你
  也许需要不同的索引，建在数据的不同部分上。
  
- 索引是从主数据衍生的附加（additional）结构。许多数据库允许添加与删除索引，这不会影
响数据的内容，它只影响查询的性能。维护额外的结构会产生开销，特别是在写入时。写入
性能很难超过简单地追加写入文件，因为追加写入是最简单的写入操作。任何类型的索引通
常都会减慢写入速度，因为每次写入数据时都需要更新索引。 
> 这是存储系统中一个重要的权衡：精心选择的索引加快了读查询的速度，但是每个索引都会
  拖慢写入速度。因为这个原因，数据库默认并不会索引所有的内容，而需要你（程序员或
  DBA）通过对应用查询模式的了解来手动选择索引。你可以选择能为应用带来最大收益，同
  时又不会引入超出必要开销的索引。

- 哈希索引: 很常见与字典（dictionary）类型非常相似   
- 所以如何避免最终用完磁盘空间？一种好的解决
  方案是，将日志分为特定大小的段，当日志增长到特定尺寸时关闭当前段文件，并开始写入
  一个新的段文件。然后，我们就可以对这些段进行压缩（compaction），如图3-2所示。压
  缩意味着在日志中丢弃重复的键，只保留每个键的最近更新。
  
> 乍一看，只有追加日志看起来很浪费：为什么不更新文件，用新值覆盖旧值？但是只能追加设计的原因有几个：
- 追加和分段合并是顺序写入操作，通常比随机写入快得多，尤其是在磁盘旋转硬盘上。
- 在某种程度上，顺序写入在基于闪存的固态硬盘（SSD）上也是优选的【4】。我们将在
- 第83页的“比较B-树和LSM-树”中进一步讨论这个问题。
- 如果段文件是附加的或不可变的，并发和崩溃恢复就简单多了。例如，您不必担心在覆
- 盖值时发生崩溃的情况，而将包含旧值和新值的一部分的文件保留在一起。
- 合并旧段可以避免数据文件随着时间的推移而分散的问题。 

>但是，哈希表索引也有局限性：
 **散列表必须能放进内存**如果你有非常多的键，那真是倒霉。原则上可以在磁盘上保留一个哈希映射，不幸的是 **磁盘哈希映射很难表现优秀**。它需要
 **大量的随机访问I/O**，当它变**满时增长是很昂贵的**， 并且解决散列冲突需要很多的逻辑 范围查询效率不高。例如，您无法轻松扫描kitty00000
 和kitty99999 之间的所有键—您 必须在散列映射中单独查找每个键。 
### 第四章：编码与演化
> 本章中将介绍几种编码数据的格式，包括 JSON，XML，Protocol Buffers，Thrift和 Avro。

- 需要在两种表示之间进行某种类型的翻译。 从内存中表示到字节序列的转换称为编码（Encoding）（也称为序列化（serialization）
或编组（marshalling）），反过来称为解码
  （Decoding） （解析（Parsing），反序列化（deserialization），反编组() unmarshalling）） 。
- Java的内置序列化由于其糟糕的性能和臃肿的编码而臭名昭着【8】

>程序通常（至少）使用两种形式的数据：
> 1.在内存中，数据保存在对象，结构体，列表，数组，哈希表，树等中。 这些数据结构针
>对CPU的高效访问和操作进行了优化（通常使用指针）。
>2.如果要将数据写入文件，或通过网络发送，则必须将其编码（encode）为某种自包含的
>字节序列（例如，JSON文档）。 由于每个进程都有自己独立的地址空间，一个进程中
>的指针对任何其他进程都没有意义，所以这个字节序列表示会与通常在内存中使用的数
>据结构完全不同 。

>语言特定的格式
许多编程语言都内建了将内存对象编码为字节序列的支持。例如，Java
有 java.io.Serializable  【1】，Ruby有 Marshal  【2】，Python有 pickle【3】等等。许多
第三方库也存在，例如 Kryo for Java  【4】。
这些编码库非常方便，可以用很少的额外代码实现内存对象的保存与恢复。但是它们也有一些深层次的问题：这类编码通常与特定的编程语言深度绑定，其他语言
很难读取这种数据。如果以这类编码存储或传输数据，那你就和这门语言绑死在一起了。并且很难将系统与其他组织的系统（可能用的是不同的语言）进行集成。
为了恢复相同对象类型的数据，解码过程需要实例化任意类的能力，这通常是安全问题的一个来源【5】：如果攻击者可以让应用程序解码任意的字节序列，他们
就能实例化任意的类，这会允许他们做可怕的事情，如远程执行任意代码【6,7】。在这些库中，数据版本控制通常是事后才考虑的。因为它们旨在快速简便地
对数据进行编码，所以往往忽略了前向后向兼容性带来的麻烦问题。效率（编码或解码所花费的CPU时间，以及编码结构的大小）往往也是事后才考虑的。
例如，Java的内置序列化由于其糟糕的性能和臃肿的编码而臭名昭着【8】。
**因此，除非临时使用，采用语言内置编码通常是一个坏主意。**

## 第二部分：分布式数据
### 第五章：复制

### 第六章：分区
### 第七章：事务
### 第八章：分布式系统的麻烦
### 第九章：一致性与共识  

---
## 第三部分：派生数据
### 第十章：批处理
### 第十一章：流处理
### 第十二章：数据系统的未来
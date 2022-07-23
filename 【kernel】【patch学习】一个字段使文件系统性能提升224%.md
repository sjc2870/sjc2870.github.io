# 背景

最新在看linux邮件列表时，看到了一个以前的patch[1]，使得xfs对小文件的读性能提升了224%[2]，可以说是一个相当大的优化，而看了patch之后，发现这是一个相当简洁易懂的patch，没有那么多复杂的背景知识，所以萌生了把它记录下来的想法。

让我们从一个简单的需求开始，如果你有一个元数据结构体，并且需要将元数据持久化到磁盘上，而你想知道某个操作后，元数据是否有变化，你会如何做呢？

一个简单的想法是，在操作前，保存元数据的一个拷贝，操作后，再拿拷贝数据和当前数据进行字段的一一比较(不可以memcmp，因为如果有指针的话，指针指向数据的比较会失灵)，这样可以很轻易的得知元数据是否有变化。但这样做法的缺点是：1. 如果元数据结构体比较复杂，整个比较函数会相当的复杂，并且以后维护也很麻烦，每次增减字段都要小心的维护这个比较函数 2. 因为我们需要持久化到磁盘上，那么任何一个元数据字段的改变，都会引起一次磁盘IO，如果元数据操作比较多，会引起性能的下降。

改进想法是，实现一个seq number，初始为0记录到磁盘上。每次进程初始化时，都从磁盘上读取整个seq number字段，如果其他线程改变了元数据并递增了seq number，那么你可以很简单的通过对比操作前后的seq number就可以得知元数据是否有改变。这样改进了我们的第一个缺点，但是第二个缺点仍然存在，至少我们在改变seq number时，需要持久化在磁盘上，我们可以通过这个patch[1]学习下在linux中是如何优化的。

在linux文件系统中，也有类似seq number的这样一个机制。只不过在linux文件系统中，元数据指的是`inode`，而seq number则是inode的其中一个字段`iversion`，你可以通过在`mount`时指定`iversion`选项来启用它。顺便，xfs是无视了mount时的这个选项的，它只会根据磁盘格式来决定是否启用，一旦格式化磁盘，除非你重新格式化，否则你无法控制xfs的行为[3]。

# 思路

再此仔细思考一下我们的需求，我们是想知道，自从上次使用/查询iversion以来，iversion是否有变化。我们之前的策略是每次改变元数据时都递增iversion，但如果一直没有线程查询，只有元数据的改变呢，我们还需要递增iversion么？patch作者Jeff Layton给出了答案：不需要。毕竟我们递增iversion就是为了线程方便观察，既然没有线程观察，那我们还递增它干嘛呢？

以下是作者的原文表述[1]：

```
  We have traditionally incremented that field on every inode data or
  metadata change. Typically this increment needs to be logged on disk
  even when nothing else has changed, which is rather expensive.

  It turns out though that none of the consumers of that field actually
  require this behavior. The only real requirement for all of them is
  that it be different iff the inode has changed since the last time the
  field was checked.

  Given that, we can optimize away most of the i_version increments and
  avoid dirtying inode metadata when the only change is to the i_version
  and no one is querying it. Queries of the i_version field are rather
  rare, so we can help write performance under many common workloads.
  
  当inode数据或者元数据改变时，我们一如既往地递增该字段。一般地该递增需要记录在磁盘上，即使
  没有其他的改变时，而这个开销是比较昂贵的。
  事实证明，该字段的消费者/使用者实际上都不需要这种行为。对所有这些唯一真正的需求是，自上次
  检查该字段以来如果inode已更改，则它得是不一样的。
  鉴于此，我们可以优化大部分 i_version的递增，并避免在只更改i_version并且没有人查询它时
  弄脏 inode 元数据。i_version 字段的查询相当少见，因此我们可以帮助提升许多常见工作负载下的性能。

  
```

总结就是，如果有人查询，那么下次递增时会真正的递增iversion。但如果没人查询，下次递增时，就不递增iversion。



# 实现

在此，我们暂且忽略系统crash带来的内存数据与磁盘数据的不一致问题，在内核中有日志系统，可以相当程度的缓解这个问题。

patch思路已经有了，看来我们需要一个flag来记录iversion是否被查询，如果已被查询，则需要真正的递增iversion并消除flag。如果没有被查询，则不需要递增iversion。

按我们常规的实现方法，新增一个`int quried_flag`就好了，如果为了节省内存可以改为`char quried_flag`。但别忘了，我们此时在linux内核中，更改的又是vfs inode，按照linux中一切皆文件的设计，在一个不大的系统中，vfs inode的数量都是数以千万甚至亿计的。按我们的改法，一千万个inode就会多使用将近80MB内存，这样的浪费是不允许的。那么，又该如何改呢？



## 用比特位来记录flag

再次梳理我们的需求，我们需要记录的只是一个flag，用0/1表示只使用一个比特位就完全够用，那么我们需要新增一个bit来记录么。答案也是不允许的，因为vfs inode有大小要求，一些文件系统也依赖于它的大小，比如ext4的inode就固定为128bit或者256bit，虽然ext4有自己的控制方法，但随意在inode中增加字段的行为最好还是不要做。

既然不能随意增加字段，又该如何做呢。patch作者给出了标准答案，iversion的最低位用做flag，而其他位则用来记录iversion的大小，见[4]，因为iversion是一个64位的数字，因为完全不用担心溢出。我们记录时，则将数字向左移动一位，递增时，只递增最低第二位，而查询时，则将查询到的数字向右移动一位然后返回给调用者。不得不说，这样的设计相当巧妙。



## 不同文件系统的差异

我们需要知道，不是每个文件系统的iversion字段都需要由vfs来维护并递增的，有的文件系统需要自己维护该字段，如nfs afs等网络文件系统。这些网络文件系统只把iversion当作一个server提供的值[5]，比如nfs客户端以这种方法存储一个nfsv4的改变属性，afs客户端存储server的`data_version`[6]。

对这种不需要vfs维护iversion的文件系统，我们称为其有一个self-managed i_version，而其他的则称为kernel-managed i_version。

对kernel-managed iversion，就按照我们的设计来做即可，最低位用作flag，其他位用来记录数字。而对于self-managed i_version，则不能这样使用，因为他们应该保证`i_version`更新在导致它们的更改之前永远不可见。此外，`i_version`更新的延迟时间不应超过原始更改到达磁盘所需的时间[7]。

因为这个原因，作者提供了`_raw`后缀的api，这些api直接进行赋值和读取操作，并不进行左移和右移，见[1]。



## 存储到磁盘

ok，按照我们上述设计，看起来相当完美。只在需要递增时递增，并且没有造成额外的内存负担，也适配了不同类型的文件系统。但我们思考这样一个场景，我们从磁盘读取`iversion`到内存时，因为是写入到内存的，不会设置`quried_flag`，考虑极端场景，只写不读，我们就不会设置`quried_flag`也不会递增iversion，那么再次存储到磁盘时，inode元数据已经变了，但iversion还是没变，这与我们原本的想法冲突，所以我们在从磁盘读取`iversion`到内存时，需要设置`quried_flag`，以此来保证下次递增会真正的递增iversion,存储到磁盘时iversion也会递增。



# 后续

## patch后续

在我们看来，这是一个比较完美的patch，设计考虑比较全面，提升了文件系统性能。但是linus在接收patch时，却回复`no, I won't pull this garbage`，虽然这么说，但他觉得问题很隐晦并不实际，而且可以轻易修复，所以还是接收了该patch合入主线[8]。那么是哪里出了问题呢？

看完linus与patch作者的邮件讨论后，原来问题是在一个函数`s64 inode_cmp_iversion(const struct inode *inode, u64 old)`上。patch作者原意是用这个函数来比较`inode->iversion`与`old`是否相同，并返回一个s64，如果返回值为0则说明它们是相同的。

但是考虑以下场景，如果调用者用以下语句

```c
int cmp = inode_cmp_iversion(inode, old);

if (cmp < 0 ...
```

仔细看，我们返回了一个s64，但是调用者使用int来接收，这样会造成数据被截断，并且这种场景在自己测试99.99999%都是可以正常工作的，linus认为有了

1. 微妙的bug代码
2. 看起来很好
3. 测试时正常

作者使用u64来返回，这样可以保证数字绝对不会溢出，但是如果调用者使用int来接收，那么会造成调用者只拿到了返回值的低32位，而低32位在一个元数据操作比较多运行时间又比较长的系统中，则是有可能溢出的，这是一个相当隐晦的问题，甚至这不是patch作者的问题，只是调用者没有处理好，但是我们需要考虑到调用者的使用。

后续patch作者出了另一个patch来修复这个问题。

## ext4后续

这个patch在xfs上工作的相当好，提升了小文件读写的性能。但是最近(2022.7.19)作者又提出了一个新的问题，是否需要在ext4挂载时默认启用`-o iversion`,原因是因为NFSV4在以ext4(没有启用`iversion`)作为server时，会有缓存一致性的问题[9]。

是否在ext4挂载时默认启用`-o iversion`还在讨论中，我们拭目以待它的后续。

# 学习总结

总结一下，我们从这个patch中学到的知识。

1. 若想记录数据的变化，可以考虑使用iversion/seq number等类似机制
2. 记录flag时，若想节省内存，可以把最低位作为flag
3. 接口的定义需要清晰，并且需要考虑到调用者的调用方法，不要为调用者埋下潜在的雷。
4. 接口和文档记录清晰。读完作者的patch后，我最大的感慨是作者把这个功能在文件开头的注释中写的清清楚楚明明白白，包括一些注意点，设计思路，甚至有的接口还注释了调用场景和用例，对读代码的人相当友好。

顺便，这个功能是分为了19个子patch来提交的，每个子patch都独立完成一个功能，比如为每个文件系统切换到新的api，这些子patch共同组成了这个大功能，所以还有第5点

5. 每个patch完成独立的功能，patch开头说明功能和原因。不要把一堆功能混入到一个大patch中，这样reviewer读起来相当难受。



# 参考

\[1]: [kernel/git/torvalds/linux.git - Linux kernel source tree](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=a4b7fd7d34de5765dece2dd08060d2e1f7be3b39)

\[2]: [LKML: kernel test robot: [lkp-robot\] [fs] 8e4a22b1c4: fio.read_bw_MBps +244.4% improvement](https://lkml.org/lkml/2017/12/25/8)

\[3]: [Re: should we make "-o iversion" the default on ext4 ? - Dave Chinner (kernel.org)](https://lore.kernel.org/all/20220721223244.GP3600936@dread.disaster.area/)

\[4]: [iversion.h - include/linux/iversion.h - Linux source code (v5.13) - Bootlin](https://elixir.bootlin.com/linux/v5.13/source/include/linux/iversion.h#L77)

\[5]: [iversion.h - include/linux/iversion.h - Linux source code (v5.13) - Bootlin](https://elixir.bootlin.com/linux/v5.13/source/include/linux/iversion.h#L50)

\[6]: [iversion.h - include/linux/iversion.h - Linux source code (v5.13) - Bootlin](https://elixir.bootlin.com/linux/v5.13/source/include/linux/iversion.h#L89)

\[7]: [iversion.h - include/linux/iversion.h - Linux source code (v5.13) - Bootlin](https://elixir.bootlin.com/linux/v5.13/source/include/linux/iversion.h#L32)

\[8]:  [Linux-Kernel Archive: Re:GIT PULL\] inode->i_version rework for v4.16 (iu.edu)](https://lkml.iu.edu/hypermail/linux/kernel/1801.3/03958.html)

\[9]: [should we make "-o iversion" the default on ext4 ? - Jeff Layton (kernel.org)](https://lore.kernel.org/all/69ac1d3ef0f63b309204a570ef4922d2684ed7f9.camel@kernel.org/)
# 前言

写这个文档有两个目的，一个是为了捋思路，真的挺复杂，有时候将想法写下来会更好思考，而且有一种鼓舞的力量；其次是为了给各位面试官大人审阅，当然也给自己看，避免以后问到：这个项目有什么难点？结果自己全忘了

代码位置在私人仓库，这里只有实现的心得，因为课程说过不能分享代码，当然我自己也没有去找过代码，包括实现思路也从没去打听过，一切都只参考以下列出的资料。

# 参考资料

计划表 [ 6.824 Schedule: Spring 2021 (mit.edu)](http://nil.csail.mit.edu/6.824/2021/schedule.html) 

Go学习：[Go 语言之旅 (go-zh.org)](https://tour.go-zh.org/welcome/1) 

资源： [chaozh/MIT-6.824: Basic Sources for MIT 6.824 Distributed Systems Class (github.com)](https://github.com/chaozh/MIT-6.824) 

[MIT 6.824 2021 分布式系统 [中英文字幕\]_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV16f4y1z7kn/) 

中文文档仅供参考，翻译不好，重点地方**需要看[原文](http://nil.csail.mit.edu/6.824/2021/schedule.html)**，按照学习顺序添加

[mapreduce论文翻译 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/141657364) 

 [Google File System](https://www.cnblogs.com/zzjhn/p/3834729.html) 

 [The Design of a Practical System for Fault-Tolerant VirtualMachines ](https://zhuanlan.zhihu.com/p/539874472)  

Raft-Extended 没找到合适的翻译  [这篇文章](https://blog.csdn.net/Hedon954/article/details/119186225) 篡改了原文，有些地方误导性很大

[ZooKeeper论文总结（Lec8)-CSDN博客](https://blog.csdn.net/qq_40229166/article/details/129266051)  

# Go

学了两天Go，感觉Go在很多方面比C++要便利，像是c++的伪代码版本，尤其是学到管道<-和Go程的时候。它能很方便的进行Go之间的通信，虽然c++也有管道和线程，但是Go只需一个关键字即可创建，Go的指针和引用我学的不是很明白，使用的时候全靠编辑器提示，它的指针语法似乎和C++差别挺大的，不过并没有颠覆我印象中的指针的定义，能用就行。

除此之外还有关于RPC远端调用，基本流程是服务端注册对象方法，客户端拨号连接，然后就可以直接调用了。

学的不深，希望不会对lab的实现产生太大负面影响。

# mapreduce

这个框架用于处理大规模数据集的并行计算任务。具有良好的可扩展性、易用性和容错性，并适用于各种应用场景。比如日志分析，分布式grep，为推荐系统和机器学习提供一定程度的底层支持。

**可扩展性**：能很轻易的添加更多的计算节点，可以实现更高的并行度和更快的计算速度。

**易用性**：提供简单易用的接口和编程模型，使开发人员能够方便地编写和执行 MapReduce 任务。隐藏了底层的分布式计算细节，让用户专注于业务逻辑的实现。

**容错性**：容错机制，能够处理计算节点的故障和数据丢失，自动重新分配任务和恢复中断的计算过程，并没有实现master容错。

master负责接收任务，分配任务，和返回结果集给用户，map worker负责将数据分片进一步划分为更小的数据块，处理输入数据并生成中间键值对，reduce worker负责将相同键的值进行聚合、合并、计算等操作，用户定义的map和reduce函数决定最终生成的结果。

任务分配策略是worker主动发起心跳请求，如果有任务则进行分配，程序完全模拟分布式下的情况：使用rpc通信，地址为ip+端口号，两种worker之间信息完全隔离，只能通过master交互信息。

mapreduce最大的特点是所有操作幂等性，不论什么任务都可以重复执行，而不会出错。

# raft

Raft 是一种分布式一致性算法，旨在解决分布式系统中的数据一致性和容错性问题。它通过将系统的状态复制到多个节点，并使用领导者选举的机制来实现容错和一致性。

**节点角色**:  Leader：负责处理客户端请求，并将日志复制到其他节点。leader通过心跳机制维持其领导地位。 Follower：复制日志，并响应客户端请求。 Candidate：在领导者选举过程中的临时角色，负责发起选举并请求投票。

**领导者选举**: 系统保证有且只有一个leader，随机化超时时间实现leader的快速选举。

**日志复制**(一致性)：leader接收客户端的请求，并将其作为日志条目追加到自己的日志中，通过发送附带日志条目的心跳消息来复制日志到其他节点，节点在收到来自领导者的心跳消息时，将日志条目追加到自己的日志中，并发送响应给领导者，当大多数节点确认接收到同一日志条目后，该日志条目被视为已提交，并可以应用到系统状态机上。

**容错性**：leader负责决策和日志复制，follower被动地复制和执行领导者的指令，如果leader失效，其他节点会启动新的选举过程，并选举新的leader，失效的leader如果挂机，通过持久化日志和安装快照快速同步。



# mapreduce

## mapreduce1.0

worker包含map和reduce两阶段，均是自己主动调用服务器获取任务，当且仅当map完成，才会进行reduce阶段。

难点：设计通信协议，设计worker和master交互的协议很头疼，最后是放弃设计协议，先写逻辑，实际通信要用到什么字段就添加什么字段，这种操作可能不规范，但是确实是迈出第一步的好方法。

具体设计：

master保存map和reduce状态，然后map请求任务并将计算结果保存在本地，并通知master，reduce从master获取中间文件位置，(这里还没有实现用rpc调用获取map结果) 计算保存在临时文件，master收到reduce完成的通知后会关闭程序

reduce的问题很大：如何获取全部中间文件并对中间结果排序，论文就是很简单的说master将任务分配给reduce，并告诉reduce从哪一台机器取数据，然后。。。，然后实现的时候发现一个reduce取到的是一部分数据，如果一个reduce取得全部数据并且排序的话，再进行二次分发？那岂不是还要再设置一个reduce分配器？还有最终结果的整合，也是一样的情形。如果把这些任务交给master的话，似乎又和论文不符，论文中明确说了中间结果在map机器上，由reduce worker主动取获取。

## mapreduce2.0

再读论文发现漏了一个关键点，map输出文件不是一个单一的文件，而是很多个，对key使用hash得到中间文件的命名，那么这就合理了，可以保证同一个key值不会出现在第二个文件里面，那么在1.0的基础改：

map将key通过ihash计算值，再%NReduce（直到这一步我才明白ihash是做什么的），得到输出文件名，保存在本地磁盘，通知master，需要发送的消息应该包含所有输出文件的位置，直接用map\<bool\>传递消息，（可以实现一个bitmap优化，但是我想先实现一个无bug版的，后期考虑优化），服务器应该累计每一个reduce应当获取的文件所在机器和路径，等map执行完毕，master通知reduce启动，然后根据map的结果给每个reduce worker分配任务，其实还是worker主动请求，如果有未完成的任务，就分配给worker

我的分配策略是master只告诉map文件位置，而不是由master直接给数据，需要map自己去找，假设我有GFS的场景，虽然实际上我只有一台机器。(这让我想起了论文中说给map分配的数据大小一般是16或64M，而GFS文件系统的chunk刚好默认是64M。)然后map保存的中间文件在map主机，reduce通过rpc得到数据，然后进行工作，最后产生临时文件，通知master，如果没有重复的话就改名，否则丢弃。

问题总是很多，新出现的问题是使用gob压缩文件内容时，gob编码解码的问题：

写入文件后就读不出来了

```cpp
for ... {
    ofile := [应该写入的文件]
    enc := labgob.NewEncoder(ofile) // 编码器绑定文件
	enc.Encode(&kva[i])       // 编码并且写入
}
```

```cpp
	repy, err := ioutil.ReadAll(file)
	fmt.Print(len(repy), "\n") // 4521502 ，说明数据存在
	if err != nil {
		log.Fatalf("cannot read %v", filename)
		return err
	}
	r := bytes.NewBuffer(repy)
	d := labgob.NewDecoder(r)
	var intermediate []KeyValue

	for {
		var kv KeyValue
		err := d.Decode(&kv)
		if err == io.EOF {
			break
		} else if err != nil {
			log.Fatalf("cannot decode data: %v", err)
			return err
		}
		fmt.Print(kv, "\n") // 能输出一个kv，但只能输出一个，很难评价，我直接气晕
		intermediate = append(intermediate, kv)
	}
```

读取操作只能输出一对kv，然后就是`cannot decode data: gob: duplicate type received` 直接百度找不到答案，可能需要对gob有一定了解才行(而我只想傻瓜式调用)

## mapreduce3.0

经过三四个小时的排查，最终发现是因为编码时追加写模式导致的编码错误。太恶心了，发生错误是在解码阶段，一直以为是解码时参数设置不正确之类的缘故，写test的时候也没关心循环外创建文件还是在循环内的区别，直到把所有可能都想了一遍，只能说是被狠狠的恶心到了。

那么解决思路就是存下每个打开文件的指针，如果当前需要写入的文件已经被打开过，那么就使用上一次打开过的，否则就打开，不过我选择更加省事的操作，一次性打开所有文件，需要的时候直接获取，这样操作对效率应该没有什么影响，从两方面考虑，这些文件从打开后就不能关闭，直到任务结束，所以动态打开和一次性打开消耗的内存一致，而且一次性打开时间上也会快些。

于是，终于得到了一个能输出的分布式框架（bushi)

接下来是容错的计划，论文上提过我们的程序必须容忍错误，比如map就算执行完毕，只要宕机那么它的任务需要重新执行，reduce宕机则不用，因为reduce执行完毕后，存储在全局文件系统。还有当个别任务迟迟等不到回复的时候，需要重新分配任务，此时不管是因为worker主机宕机，还是执行速度慢，都重新分配。论文似乎没有说到有心跳检测机制，那我也不实现了，所有回复慢的一律再分配。

改着改着，突然又跑不了了，而且出现了贼抽象的bug

```
// mapworker
reply.Inputfiles = ""
reply.Taskid = -1
fmt.Print(args.Port, " 1111111 ", reply, "\n")
err := call("Coordinator.GetMapTask", &args, &reply)
fmt.Print(args.Port, " 2222222 ", reply, "\n")

// GetMapTask func
	c.mutex.Lock()
	defer c.mutex.Unlock()
	reply.NReduce = c.nreduce
	for i, filename := range c.filename {
		if c.mapTask[i] == false { // 未分配的任务
			fmt.Println("map分配 ", i, ' ', filename)
			c.mapTask[i] = true
			reply.Inputfiles = filename
			reply.Taskid = i
			return nil
		}
	}
```

按理说，reply作为传出参数，要么全被改，要么不修改，它在GetMapTask里面只被修改了一半....但是GetMapTask并没有出现内部崩溃的现象（err==nil），哎我真服了，这都是什么奇葩，具体就是reply传回了文件名，但是没有传回Taskid，通过代码可以看到，我首先清空了Inputfiles和Taskid=-1，紧接着直接call协调器，然后协调器会给Inputfiles和Taskid赋值，离谱的是当filename ==  "pg-being_ernest.txt" && i == 0的时候，就出现这样的输出：

```
32833 1111111 {false -1  10}
32833 2222222 {false -1 pg-being_ernest.txt 10}
```

可以看到Taskid未被赋值为0，但Inputfiles被赋值了，难道是0不配？多测试几组，只要是遍历到了0，就会出这个错误，无一例外：

```
37079 1111111 {false -1  0}
37079 2222222 {false 2 pg-frankenstein.txt 10}
maptask sussess 2
37079 1111111 {false -1  10}
37079 2222222 {false 4 pg-huckleberry_finn.txt 10}
maptask sussess 4
37079 1111111 {false -1  10}
37079 2222222 {false 6 pg-sherlock_holmes.txt 10}
maptask sussess 6
37079 1111111 {false -1  10}
37079 2222222 {false 3 pg-grimm.txt 10}
maptask sussess 3
37079 1111111 {false -1  10}
37079 2222222 {false 5 pg-metamorphosis.txt 10}
maptask sussess 5
37079 1111111 {false -1  10}
37079 2222222 {false -1 pg-being_ernest.txt 10} // 注意就是这一行出错
```

之前测试必有0号任务，但是一点问题没有，这究竟是为什么：首先0号任务是一定被分配了的，因为fmt.Println("map分配 ", i, ' ', filename)被执行了，Inputfiles也被赋值了，那么Taskid是为什么没有被赋值，因为后续任务的报告都是依靠Taskid，这里对不上账，后面几乎全错了。

进一步输出发现Taskid被赋值了，但是call完传回去又变成-1了，真是开眼了，reply不是指针吗，如果Taskid无法传递那么Inputfiles也不应该有值才对，凭什么！！！算了，不解决了，我直接把下标设置为从1开始。



好好好，这个bug完美解决，虽然还是不知道为什么会成为bug。

## mapreduce4.0

来到4.0，本来3.0版本就该解决容错机制的，但是那个bug查了好几个小时写吐了，那么就把容错机制甩到4.0来好了。

3.0采用的策略是只要worker请求了，master就分配，但是从不主动分配任务，worker端开始执行reduce后就不会再请求map任务，这样导致的问题：如果reduce阶段开始了，运行到一半，map主机宕机了，那么woker也不会再向master反馈了。

我采用比较"懒"的做法：

**第一种情况：**

master设置一个通信方法，用于验证worker是否还能用。每分配出一个任务，就启动定时器1绑定任务，定时器1结束之前如果没有收到响应，那么启动通信验证，如果worker不在了就重新分配，否则就重新定时，如果三次后还没有完成任务，那么就当它不在了，重新分配。这里的情况比较单一，等待worker请求就好，如果不想分配的话就先阻塞，等master判断可以分配了就分配任务，如果完成了当前阶段的任务，就既不分配同时也放掉阻塞。

完成以上大概需要添加任务结构体，分为maptask和reducetask，要存储执行task的worker的通信地址，任务id等，还有定时器。

除此之外，还需要解决**reduce阶段，map连接不上的情况：**

reduce callMap会尝试连接三次，三次失败之后，暂时不通知master，先查看输入文件列表是否有其它可用文件，如果没有，那就通知master，否则使用新的可用文件，master必须让原先map得到的任务重新执行，并且停止reduce任务分配，防止出现大量reduce失效的情况，正在运行的reduce如果没出问题的话，就不受影响，不过大概率会出问题，除非reduce赶在map失效前读取到了数据。

失败的reduce会选择继续执行，不过我觉得相比于复杂的容错机制，重新执行并不会造成太大的额外开销。如果有reduce第二次报告错误，那么会直接更新reduce输入文件列表。

完成这些动作需要在reduce callMap失败之后，callMaster，master修改文件列表，并返回当前正确的文件地址，使用maptask记录id就不用在第二次报告失效的时候重新执行了。还有在master收到失效消息的时候，要先callMap进行一次确认，如果可以call，那么就直接返回了，不用进行其它操作。

思路清晰了，开始动手开始动手开始动手开始动手开始动手开始动手开始动手

![mapreduce](picture/xuehuasu/mapreduce.png)

好好好，完结撒花！！！



最终的rpc通信协议：

```Go
type MapArgs struct {
	Addr       string // worker地址
	Port       int    // worker端口
	Taskid     int
	Tasktype   string
	Outputfile map[int]string
}

type MapReply struct {
	Taskid     int
	Inputfiles string
	NReduce    int
	Timeout    time.Duration
}

type ReduceArgs MapArgs

type ReduceReply struct {
	Taskid     int
	Inputfiles []string
	Outputfile string
	Timeout    time.Duration
}
```

master字段

```Go
type Coordinator struct {
	// Your definitions here.
	idx     int
	nreduce int

	filename map[int]string

	// 已分配的任务为true，未分配的任务为false
	mapTask    map[int]bool
	reduceTask map[int]bool

	// 已完成的任务为 结果地址，未完成的任务为 ""
	// taskid -> fileaddr
	achieveMapTask    map[int]string
	achieveReduceTask map[int]bool

	// 用于判断是否完成任务
	achievemap    chan bool
	achievereduce chan bool

	// 互斥锁
	mutex sync.Mutex
	// 条件变量
	cond *sync.Cond

	// 中间文件id对文件地址的映射 int -> []string , 文件id是根据nreduce分配的
	midfile map[int][]string

	// 完成map
	achievmap     bool
	achievmapchan chan bool
	// 完成所有
	achieveall     bool
	achieveallchan chan bool

	// 为每一个Task设置超时器
	maptimer    map[int]*timer.Timer
	reducetimer map[int]*timer.Timer
	timeout     time.Duration
}
```

worker

```
type Maphost struct {
	Addr string
	Port int
}
```

4.0 感想：

刚开始最难的部分是制定通信协议，可能万事开头难，刚上手根本想不到后面该干什么，除非有足够经验，否则过早的制定通信会给后面带来很大的麻烦，最好的办法是不要一开始写死，留一些扩展的余地，缺少什么字段就往协议里面加，有多余的就减，不要一开始就苦思协议，这样意义不大，除非写多了经验丰富，可能就是所谓架构师吧。

第二个难点是debug，这一阶段实现的是基础功能，就是分布式下能正常运行就可以了，但是这种多线程系统要debug真的很麻烦，多线程下的gdb调试只会c++的，go的还没用过，原理应该一致，但我就是头铁，只使用fmt调试，效率不太高，不过想想使用gdb的话也需要各种手动切换线程查看堆栈变量信息，大佬操作，我不弄可能还是正确的选择（狡辩）。关于那个Taskid=0传回原值的bug至今也不知道是什么原因，之后也遇到过一次远端调用值不改变的bug，但是那是由于我通信字段没有首字母大写，Go里面有一个导出变量的概念，想要访问不属于本文件的变量，必须首字母大写(我没有深入了解，可能有误)，我回过头看Taskid当时应该是已经是大写了，不存在这个问题才对，而且bug在于只有等于0的时候才无法改变值，其余的数字均正常赋值。

第三个难点是容错机制的实现，论文提到：map失效必须重新执行，reduce不用，超时的任务选择重新分配

寥寥三句话，花了我好几个小时实现，主要还是思路想半天，还有debug：超时使用定时器定期发布任务，仅仅是发布，只有等worker过来取的时候我才分配，master会根据当前进度选择不同的心跳时间，心跳时间一到，worker就会与master相连，此时如果有任务就直接分配，没有就下个心跳时间再来。如果map挂了，而此时又是reduce阶段的话，会阻塞所有reduce分配，直到map完成为止。



# raft

通过对lab1的实现，初步了解了分布式系统，和前几篇行为单一的论文不一样，raft论文描述的东西很多，很复杂，花了三四天才看完论文，每次都看着看着就没心思看下去了，一直在拖延，中文文档翻译的很一般，感觉像机翻，硬着头皮看完原文。感觉面对难以弄懂的知识，很有效的一个手段是反复学习，而不是从头懂到尾（过于理想的状态），初期看raft根本看不下去，多看几遍之后感觉很有意思。

raft的复杂性远高于mapreduce，lab2被分为abcd四个小lab。 论文用一张表简要说明了大致的实现方向：

![image-20231003205708748](picture/xuehuasu/raft.png)

一些要点：

1. 如何定义日志的“最新”，如果lastTerm不同，则数字越大的越新，如果lastTerm相同，则索引（日志长度）越大的越新，candidate不能投票给比自己“旧”的。

2. leader不能提交往期的日志，只能通过提交当前任期日志，间接提交往期的日志。

   ![image-20231013223537152](picture/xuehuasu/raft-figure8.png)

3. 安全性通过以上两点保证，但不完全只有以上两点

![image-20231004094837640](picture/xuehuasu/raft-installSnapShot.png)

## raft 2a

2a主要是leader选举方面的实现

万事开头难放在任何时候都是这样的，拿到代码框架完全不知道从哪里入手。。。磨蹭半天根据图2写了一些结构体，做了下初始化。

虽然2a只检查领导选举，但是又不能只写选举，选举机制建立在日志的新旧程度上，那么要先把日志相关的写了，这个过程遇到很烦人的问题，日志这个东西不管在哪里都掺和了一脚，导致只要有一个地方出现纰漏都要改很多个相似的代码，于是决定把日志抽象成类，给它定义一些管理方法。

```go
type Entry struct {
	Term    int
	Command interface{}
}

type Log struct {
	Entries []Entry
	index0  int // 日志起始下标，压缩日志时使用，默认为0
}

func (log *Log) endIndex() int {
	return log.index0 + len(log.Entries)
}

func (log *Log) beginIndex() int {
	return log.index0
}

func (log *Log) lastTerm() int {
	if len(log.Entries) == 0 {
		return -1
	}
	return log.Entries[len(log.Entries)-1].Term
}

func (log *Log) append(entry Entry) {
	log.Entries = append(log.Entries, entry)
}

func (log *Log) get(index int) Entry {
	return log.Entries[index-log.index0]
}

```

目前来说就这些，随着功能的增加还会不断完善，抽象出来后主体代码简洁多了

![image-20231005211821985](picture/xuehuasu/raft2a.png)

中间出了一个bug，产生原因是reply参数被多个线程共享了，一开始没有注意，gob的警告一直被我忽略(一直以为是其它地方造成的，在寻找逻辑bug)，直到我重视gob警告说对象可能无效，才发现这个问题。所以warning不是没有道理的(捂脸)。



## raft 2b

2b主要实现日志的复制，Start 和 AppendEntries，大部分的代码2a已经实现了，在这个基础上改一改就能通过（希望如此）

日志索引从0开始的话会出现下标为-1之类的边界情况，特判起来很麻烦，不仅需要考虑下限还要考虑上限，容易引起bug，从1开始比较好，此时就体现出2a中抽象出log的优势了，我只要改一下初始化和个别方法就可以了。

同样的，任期也改为从1开始，可以避免很多麻烦。

debug:

1.日志无限输出，陷入了死循环，排查后发现是**死锁**，是ticker函数占用锁，而负责发送心跳的函数也申请了锁导致的问题，解决办法是冗余函数代码，对有锁和无锁分别写一个实现，并加上后缀L进行区别，有L表示i调用方有锁，无需申请，没有L则需要主动申请。这个bug2a的时候应该就存在，但是没有测试出来。

题外话：现在打日志是越来越熟练了，把可能要的信息尽可能详细的全部打印出来，然后使用搜索工具检索关键字就好了，日志信息最好包含时间戳，对象唯一标识，和一定的提示信息，现在想想以前的调试真像原始人，打日志一定不要偷懒，日志信息越少调试越困难。了解到更高级的方法是对日志进行解析，分颜色显示，甚至是分列显示，不过目前还没做到这步。

2.有一个问题是follower在复制完上一条日志前，又收到了新的同步，导致一直在同步，多出了很多rpc请求，这里的策略的是失败立即重试，可能是这个策略的缘故，那么我就改为失败后等待下一个心跳同步。

3.八个点过了六个，其中一个：`cmd 100 missing in [1 101 101 104 104] ` 猜测是复制日志的时候有细节没处理得当。检查了一下发现日志全部都完整的提交了，主从都提交了完整的日志上去。有点奇怪，follower和leader已经收到了完整日志并且全部提交了，难道是发送的数据没有被正确接收？或许我误解了测试程序让我提交的东西。

![image-20231007162405918](picture/xuehuasu/raft2b-debug1.png)

我真不知道该说什么了，测试程序只收集22345，按理说应该是123456的，而且`set logs`是测试程序收到的，并且记录的日志（可以看到有6个），`wait index`是测试程序协商阶段产生的下标(~嘶，我突然想到问题出在并发Start，可是我加锁了呀)。

万万没想到居然是Start的问题：leader添加日志的时候，因为要求Start立即返回，所以我开了一个线程去添加日志，而问题就出在这里，并发条件下，`endIndex`并不会像想象中那样及时更新，所以要添加完日志后，再返回（本来是理所当然的一件事，但是由于一开始的时候对流程不够熟悉，后面又把这里忘了）

 多测试几次后，发现会出概率bug（卒）

为了方便测试，我直接写了一个脚本，运行命令： ` ./test_loop.sh 2B`

```cpp
#!/bin/bash  

FLAG=false
total_time=0
result() {  
    if [[ ${FLAG} == true ]]; then
        echo "FAIL"
    else
        echo "PASS ALL " ${i}
    fi
    echo "time: ${total_time}s"
    exit 1
}  
trap result SIGINT 
  
if [ -z "$1" ]; then  
    echo "第一个参数为空"  
    exit 1
fi

test=$1
i=1
for ((; i<=3000; i++))  
do  
    s_time=$(date +%s)
    echo "Running test ${test} iteration $i..."  
    filename="out/${test}output${i}.log"
    
    go test -run ${test} > ${filename} &
    wait # 等待上一个命令结束 
    e_time=$(date +%s)
    run_time=$((e_time - s_time))
    total_time=$((total_time + run_time))

    # 检查日志最后一行的第一个单词是否为 "FAIL"
    last_line=$(tail -n 1 ${filename})
    first_word=$(echo "$last_line" | awk '{print $1}')
    second_word=$(echo "$last_line" | awk '{print $2}')
    if [[ "$first_word" == "FAIL" || second_word == "FAIL" ]]; then
        FLAG=true
        echo "--- FAIL "${i}" time: ${run_time}s"
    else
        echo "PASS "${i}" time: ${run_time}s"
        rm -rf ${filename}
    fi
done

result()
```

分析日志我发现选举过程有概率split-brain（脑裂），同一任期出现两个leader。居然有人投出去两票！！！怎么会这样。

发现问题在于每次非leader更新日志的时候，把状态切成Follower的同时还会把Vote置为-1。并发条件下有些落后的信息就钻空子了。

![image-20231008142229031](picture/xuehuasu/raft2b-testloop.png)

## raft 2c

2c要求对状态持久化，实现persist和readPersist，同时使用gob对数据压缩。需要持久化的字段应该是raft图2描述的state

运行了25次，8次FAIL，70%能通过至少说明persist没问

2C的数据量很大，按照2B的调试精度直接搞出八万行，每行上百个字段，根本调不了

debug:

日志很混乱，下标相同的log，内容完全不一样，还有的server日志差了几个版本，根本就是一团乱麻。

日志梳理插件`logFileHighlighter`,只需要像下面这样简单配置，log文件的相应内容就能高亮显示，支持正则表达式。想让谁亮，就让谁亮。

![image-20231008192125771](picture/xuehuasu/log_tool.png)

2C的环境是模拟网络很差以及服务器不定期下线，要求我通过持久化来解决这些“非人为”导致的错误。

日志中0号服务器不是leader，但是却出现了0号发送的同步消息，虽然它没有成功同步，但是十分诡异的是它发送的消息中包含的任期号比它自身大一些。到底是哪里调用了发送函数，还有发送的任期号怎么会比自身还高？不只是2C中，2B中也有这种行为，有完全不属于当前进度的任期号出现，根据输出的日志可以断定心跳函数被奇怪的东西调用了，而且还发生了未定义行为(比如不属于自身的高任期)。再仔细看，有时候就连主机号都是不存在的，但是数值又比较小（只比正常的id大1或2），一开始根本没注意这个，那么现在rpc失败成了理所当然，我一开始还认为是测试程序随机让我失败。这下范围就缩减很多了，肯定是发送同步心跳那块出的问题，发送方不知道在乱发什么东西。

检查一遍之后更奇怪了，从准备到同步心跳的过程就那么几个函数，能发送心跳的只有两个途径，这两个途径全程上锁，不可能被奇怪的东西影响和调用，我懵了。

`sendMsg: args: {2 2 1 2 [{2 102} {2 103} {2 104}] 1} to 0 failed myId: 2 currentTerm: 6 votedFor: 0`

args参数是2，当前任期号已经是6了，这个args是很远之前的消息了吧。

实在是受不了了，回退到2B提交那个版本，经过多次测试这个版本的2B没有问题。那么就是2C这里弄错了东西(至于搞混了什么真的捋不清了，还好可以有版本回退)，调试的时候不出bug比我出bug还难受

原因是：leader收到往期发送的日志的回复，没有做检验，导致数据混乱了，我添加了校验，并且将超时选举时间增加了50ms，避免选举次数过多。

![image-20231010123912538](picture/xuehuasu/raft2c.png)

测试了34次全部通过，平均每次140s左右，在正常范围内。

> 一次面试中，面试官问我为什么对调试命令不太熟，我说我一般在vscode里面`ctrl + f`看日志，然后被嘲笑了......他说正经程序员还是用终端。



## raft2d

终于到这一步了，这里的难度看起来不小于前面任何一个，如果前面的基础够牢固的话会简单很多(即使我测试了很多遍，我也不敢保证真的没有错误)，2d主要要求对日志进行压缩，通过快照将服务器快速重启。

除了2d必须要添加的快照操作函数、接口、变量之外，在原来基础上只需要修改`AppendEntry`和`sendMsg`就够了(确信)，需要修改的两个函数涉及心跳同步机制，之前同步没有考虑快照，现在可能需要考虑发送快照给对方同步。

考虑了很久怎么实现快照的存储和读取，结果读文档发现已经提供好了接口，我只用把结果提交给接口就行了(流汗.jpg)

debug:

1.故事的场景是这样的：leader添加当前日志后，死机，然后立马恢复，开始重新选举，并且成功当选，于是当前leader拥有最新的、尚未同步的、尚未提交的日志，leader通过心跳同步，把这个往期日志同步给follower，然后新的日志到来，leader同步并提交这个日志，同时往期日志也会被顺利提交，一切都很正常，但是不和谐的bug出现了：

leader 2死机前： 测试文件收到并输出： server 1 apply 107 lastApplied 106

leader 2收到新日志，下标为108，然后马上死机重启，重新当选

leader 2收到新日志，下标为109，同步并提交，测试文件输出：server 1 apply 108 lastApplied 99

上面这一段的bug是 server 1作为follower，提交了107之前的所有日志，然后新leader上任后，100~107之间的似乎就白提交了。

我突然看到上层调用了`CondInstallSnapshot`函数，让两个follower切换了快照，截断了99之前的日志，终于发现bug起因了：server 0切换快照后重新提交了99~107的日志，但是server 1并没有重新提交，明明是同一份代码，为什么会有这种差错？？？不过bug位置找到了，之后就好解决了。

破案了，我傻逼的把`CondInstallSnapshot`全部返回了true，导致原本没有切换，上层却以为切换了快照。

2.小bug，新增的快照功能没有维护leader的唯一性，导致分裂了。检查一下快照相关函数就解决了。多线程下我们无法确认函数执行顺序，包括rpc函数，所以要在每次发送前检查自己是否还是leader，**或者**开启线程发送之前就先定义好参数，确保发送的数据一定是确定的状态。在rpc调用前后，有可能发生状态转变，所以也要做参数校验(代码中其实就是检查任期)，这个问题在日志同步和领导选举中也是一样的。

![image-20231014235200003](picture/xuehuasu/raft2d.png)

测试全部： `time go test -race` 

![image-20231016205517053](picture/xuehuasu/raft-runtime.png)

测试了一天：

![image-20231016204709825](picture/xuehuasu/raft-looptest.png)

最终结构：(rpc通信字段是按照figure 2 和 13设计的，就不展示了)

```go
type Raft struct {
        mu        sync.Mutex
        cond      *sync.Cond
        peers     []*labrpc.ClientEnd // 所有的RPC端点
        persister *Persister          // Object to hold this peer's persisted state 持久化状态
        me        int                 // 当前节点在peers中的索引
        dead      int32               // 用于标记当前节点是否已经被关闭

        applyCh   chan ApplyMsg
        applyCond *sync.Cond

        state        int       // 当前节点的状态
        electiontime time.Time // 选举超时时间

        currentTerm int // 当前任期
        votedFor    int // 当前任期投票给谁了
        log         Log // 日志

        commitIndex int // 已经提交的日志索引
        lastApplied int // 已经应用的日志索引

        nextIndex  []int // 每个节点的日志索引
        matchIndex []int // 每个节点已经复制的日志索引

        snapshot          []byte
        lastIncludedIndex int
        lastIncludedTerm  int
}

type Entry struct {
        Term    int
        Command interface{}
}

type Log struct {
        Entries []Entry
        Index0  int
}

```



# kv-raft

这一阶段，是在lab2的基础上写一个能够提供键值服务的应用层，我看半天实验要求好像并不需要依靠哪篇论文来实现(看Zookeeper和CR的时候可累死了)。

## 3A

client通过rpc调用连接server，server只需要调用raft中的操作，并保存一些状态就可以了。

一共三种操作：Get Put Append，需要做操作的重复检验，若是重复操作直接返回缓存的数据，重复操作指的是和上次操作成功的序列号一致的操作，而且并不能用put的数据和缓存数据是否相等作为判断，因为有：put 1, put 2, put 1类似这样的操作，可能会读取旧数据。与此同时注意Get操作也需要传递给raft层，我们必须确保get之前所有的操作都已被提交，读取缓存可能会读取到旧数据。

重启服务的测试没有通过：

原本的raft实现中我并没有重新提交已提交过的日志，但现在挂机之后server层未持久化的数据丢失，需要重新读取日志来恢复。

效率低：

在效率测试中超时严重：

`Operations completed too slowly 51.256097ms/op > 33.333333ms/op`

查看日志也没发现特别的错误，就是非常慢，那么又要回到raft层，看看情况：原因在于start之后我并没有立即同步，而是等在心跳时间，而我的心跳时间是50ms，那么破案了。刚开始写raft考虑的是每次加一个log就发送同步信息，如果信息太多了，可能会造成网络拥堵，而现在是server端必须等待它同步完成，才能退出，就导致了效率的问题，那么我就在start之后发送同步日志消息。

但是这么做之后又出现了一个问题：raft的2D一个都过不了，全被挂了，我一脸懵，不就是消息多了点吗，关键是其它的全能过，就2D有问题，2D是快照相关的部分， 为什么这会影响到我的快照？？

debug: 以前只有一个地方会向上提交日志，那时候是正常的，现在途径有交叉，导致提交日志的时候发生了多线程下的资源混乱，于是我优化了一下提交函数，并由原来的函数调用改为信号触发，再次测试发现效率也得到了很大的提升(lab2的总测试时长从平均380s降到了平均340s)

但是仍然无法通过，响应时间还是50ms

1. 以前写的时候对持久化的需求不够明确，有些地方不需要持久化，把多余的持久化都删掉之后。额，还是有点慢。。。。。

   ![image-20231030225101538](picture/xuehuasu/kv-raft3A.png)

   

![image-20231030222744891](picture/xuehuasu/kv-raft3A.png)

## 3B

这一部分主要是实现快照功能，并没有什么工作量，(如果前面的东西都写对了的话)

我自认为已经非常完备了，但就是运行速度仍然较慢，我上网查了一下解决方案，觉得我的方案并没问题，但是就是慢。把日志全注释了倒理想一点。

bug:

1.快照发送的异常频繁，结果发现日志大小从来没有减小过一直在增大，明明做了压缩呀。原因：对快照的去重做的太简陋了，应该更新的时候判断错误，导致一直不接受快照。

2.部分合法的日志被拒绝更新，导致信息丢失。server层有两处判重，一个是整体的序列号，一个是客户的上次请求的序列号，我把整体序列号的更新操作放在了客户序列的判重前面，导致出现了客户端认为合法但server认为非法的情况，把更新换下来就可以了，不过现在看来整体序列号这个字段完全可以删除掉，不记得当时是出于什么考虑加了它。

![image-20231030214215800](picture/xuehuasu/kv-raft3B)

测试了一百多遍，应该是没问题了。

这个项目写到现在已经过了一个半月，下周就去实习了，感觉这个项目最大的难点除了看论文外，就是debug，这和在学校写管理系统完全不是一个量级的，日志文件几万几万的输出，看的眼花缭乱，有时候光是定位到bug都感觉心累，很是抗拒，更别提顺着时间线去找bug产生原因了，这个过程非常的难受。现在我debug的方法是写一个grep的脚本，每次利用脚本将不同关键字信息写入不同的文件，然后再到各个文件里面查看想要的关键信息，这样找bug也挺累的，不过我也想不到更好的方法了，总比每次都手动grep强.

debug累还有一个因素是：输出日志文件不够规范，有时候日志里面有哪些信息我自己都忘了，就导致debug效率下降不少，一开始想像C++那般使用宏\_\_LINE\_\_快速定位，结果百度一下好像没有类似的功能(可能有，但我没去深究)，日志内容输出的比较随意，想找到想看的日志信息比较麻烦。

最后我统一了日志输出规范：

时间 [raft | server] 函数名 唯一标识(me) 具体内容


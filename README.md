# CMU15-445
卡内基梅隆大学2021秋数据库实验

lab0直接跳过，因为自认为有一点cpp基础，加之前面两个实验(cs144和cs143)也是cpp写的，并且并没有发现写实验需要对cpp掌握很深。

## lab1

实验1是手撸一个内存池。如果有os基础的话，相信应该非常容易上手。只需看完CMU数据库视频buffer pool相关章节即可开始。实验被分成三个子任务。


### LRU REPLACEMENT POLICY

这个子任务是写一个lru算法替代类，一开始我觉得这个任务估计是最简单的，后面才发现玄学多着。

#### PARAM OUT

cpp的这个注释我一开始是不知道的，输出参数。即虽然这个参数是传入的形参，但它是个指针变量，起到输出的作用，你需要将要输出的参数值写入此指针所指向的地址，是为输出参数。如victim
函数。起初我很疑惑，为啥这个函数删除的一个frameid，但是不返回呢？这样调用者怎么获得你删除的那个frameid呢，后面一看注释，才知道这个知识点。妙啊

#### 哈希表与链表结合

为了实现lru算法，我使用链表按到来的先后顺序存储frameid，使用哈希表存储frameid和其在链表中的指针。这样我需要对链表中指定数据的删除时可以直接做到o1时间复杂度，而我要找到victim
也只需pop链表头，从哈希表中删除即可，时间复杂度也为o1，需要添加framid也只需要将其push到链表和加入哈希表（重复则不加），时间复杂度也为o1。这样可以大大减少时间开销。一开始我是
单纯使用哈希表存储frameid加时间戳，这样寻找victim的时间复杂度是on，线上测试内存这块会超时。

#### UNPIN

这个方法需要注意，当然你看本地的测试文件时也会得出这个结论。就是unpin多次同一个frameid时，时间应该以第一次unpin时为准，而不是最后一次


### BUFFER POOL MANAGER INSTANCE

这个算这个实验的核心与重点吧，下面分别介绍一些实现的函数思路

#### NewPgImp

虽然说注释给出了特别详细的信息步骤，但是你直接按注释写会死翘翘的。这个注释有没有提到，就是如果你找的是victim，那么，你必须对其进行脏判断，若是脏判断则必须写回，并且要将脏标
记改为不脏，还有一个就是要去page_table_中取消它们的映射！！！注意到这两个关键点后，在配合注释食用，那基本差不多了。哦对了还有就是new也必须要pin。

#### FetchPgImp

这个方法与上面的方法唯一区别就是，这个方法首先会在page_table_中寻找，找到则pin，然后返回，还有这个方法不会分配pageid

#### UnpinPgImp

这个方法首先应该在page_table_中寻找pageid，找不到则直接返回，注意每次涉及到pageid和framid大大联系时必须先在page_table_中寻找！我就是一开始unpin方法没有查找就直接使用，导致
最后出现莫名其妙的bug。然后就是dirty的设置，如果isdirty为真，则需要标记其为真，如果为假则不要标记其为假！！！剩下的估计都是常规操作，按注释来即可。


### PARALLEL BUFFER POOL MANAGER

这个子任务估计是最简单的，只需要按照mod的映射关系，调用相关instance的相关接口即可。提一下NewPgImp，这个方法注意你starting_index加的时机，这里注释没有说明白，不是说你一轮寻
找完了才对starting_index加1，而是每找一个instance，都对其进行加1，直到成功，成功后再加1。

最后提一下线程安全，只需要在相关函数前后加锁即可，无脑！



## lab2

实验二是实现extendible哈希表，其实我更想写b+树的，可惜这个好像是2020年的，以后有机会写一下。

三个子任务，一开始是高估的第一个子任务的难度，其实看测试文件，只需要实现一点点即可。


### PAGE LAYOUTS

#### DIRECTORY

第一个就是directory的IncrGlobalDepth()，不仅仅是简单的加深度即可，还要注意复制前一半的pageid和深度到后一半，当然你去上层去实现这个机理也可。第二个就是GetSplitImageIndex，
这里的SplitImage概念指导书说自己自然而然会明白，我解释一下，举个栗子，xxxxx1000的SplitImage是xxxxx0000，x的位数是globaldepth-localdepth，代表0或者1，而数字的位数则是
localdepth。所以获取SplitImage是bucket_idx ^ (1 << (local_depths_[bucket_idx] - 1))。这里获取的是其中一个镜像，这里注意深度为0时调用此方法会报错！！！这里还得区分一下镜像
(SplitImage)和镜像族，镜像是镜像族的一个，我这里获取的镜像是只有第localdepth位不同，其他都相同，而镜像族则是所有localdepth-1位都相同，但是第localdepth位不同的所有index集合
(最高位限制到globaldepth位)，在分裂和合并时镜像族的概念非常重要！！！

#### BUCKET

bucket的删除，删除是位删除，只需要将readable设为0即可，occupied不需置位。最后就是0(1)长数组，这个自行google。


### HASH TABLE IMPLEMENTATION

这个是重点。我提一下fetch和new的调用时机，还有就是flite和merge的思路机理


#### FETCH AND NEW

directory_page_id_在构造方法中需要调用new方法新建一个page，将directory的内容写入其中。bucket的page在构造方法中也需要首先先new一个，将pageid号写入directory中。还有就是在分
裂flite时需要生成新的bucket，此时也需要调用new方法，其他的任何时机，只需要调用fetch方法！！！如果不这样，会导致你的数据不一致，还有很多奇奇怪怪的错误！！！还有就是每一个
fetch或者new必须要即时unpin，否则会内存溢出。

#### FLITER



#### MERGE






















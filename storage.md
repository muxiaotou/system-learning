    大多来源于”奇犽云存储“公众号，“存储技术”相关文章
    还有“小林coding”公众号里面这篇文章：https://mp.weixin.qq.com/s/qJdoXTv_XS_4ts9YuzMNIw
    对整个文件系统的IO相关都做了系统性的阐述
    
    #https://mp.weixin.qq.com/s?__biz=Mzg3NTU3OTgxOA==&mid=2247489085&idx=1&sn=1eb51ea0a00a00a7a62f232221bafaf8&chksm=cf3e06f8f8498fee063670f21bd0a756ef821e359709ed62720b48927ad18eeb0757399036a8&cur_album_id=1778430570125967361&scene=190#rd
    文件的读写核心：
    读文件：把磁盘上文件的特定位置的数据读到内存的buffer：
    写文件：把内存buffer的数据写到磁盘的文件的特定位置。

    特定位置：offset, length
    内存buffer：文件的数据直接和内存打交道的，写的时候从内存写到磁盘文件，读的时候从磁盘文件读到内存
    IO行数本质上都离不开offset、length、buffer这三个要素

    Open：open文件获得一个句柄，也就是file结构，后续对文件的读写都是基于file结构之上进行的，open文件的时候，当前偏移量默认是0
    Write、Read 分别写一个buffer到文件、从文件读一个buffer
    Seek 指定偏移量。

    #https://mp.weixin.qq.com/s?__biz=Mzg3NTU3OTgxOA==&mid=2247488282&idx=1&sn=79bdc53ac60e7f1f4a6dca8fbd4f92fd&chksm=cf3e03dff8498ac93a0bca919ad8e71eda99f05517fd8d0c44f6522a2e443c3133ede758c3ba&cur_album_id=1778430570125967361&scene=190#rd
    cp文件引发的思考：
    ls -lh 以可读方式显示列表文件size
    du  -sh  评估文件空间占用
    stat 显示文件状态信息，其中size是文件逻辑大小(多数人看到的文件大小)，单位是byte，仅仅只是inode的一个属性，实际占用的物理空间需要看blocks
    的数量，以上size和blocks分别对应文件的逻辑空间大小和物理空间大小
    blocks表示物理实际占用了多少个block(512byte)，IO Block表示这个文件占用的总大小，单位是byte

    文件系统：通俗来说就是用来存储数据的一个容器
    文件系统的存储介质是磁盘，本质是就是负责怎么使用磁盘空间，并且保证将文件按照一定的规则存储到磁盘。
    文件系统将磁盘物理空间按照一定粒度切分，一般是8个block，即8*512byte=4KB

    写文件：
    数据按照block粒度写到磁盘对应位置，然后再把block对应的磁盘位置写入元数据(即inode)
    读文件：
    先读元数据,找到各个block的位置，再读文件，构造一个完整的文件返回给用户

    一个文件系统的inode个数，是文件系统格式化的时候就确定好的，因此inode个数是有上线的

    bitmap(superblock)：
    当文件系统格式化好之后，inode和block的个数就已经确定好了，文件系统超级块有一个bitmap来维护这些inode和block哪些空闲哪些已经被使用
    inode和block个数由于格式化被确定，因此文件的个数和大小都是有上限的，bitmap是一个二进制表,本身应该是4B

    inode当中一般有个数组来存储对应的inode所占用的block列表，当然这个数据空间也是已经分配好了的，占用磁盘空间的，一个block一般对应底层4KB，而
    每个block编号一般占用4byte。
    举例ext2，存储blocks列表的数组长度是15，即最多能存15*4KB=60KB大小文件？实际是可以支持4TB文件的，那怎么来存储的呢？答案是采用间接索引：
    比如ext2：
    1）inode当中blocks列表前12个位置，为直接索引，每个索引位置记录的即是磁盘block的实际地址，可以存储的文件大小12*4KB=48KB，每个索引占用4byte，
    则可以寻址48B磁盘空间
    2）inode当中blocks列表第13个位置，为一级索引，索引位置记录的是一个一级索引的地址，一级索地址内的数据实际存储的是block地址，一个block 4KB，而一个索引
    占4byte，一级索引地址则可以存储4KB/4B=1024个block块地址，则可以寻址1024*4KB=4M的磁盘空间
    3）inode当中blocks列表第14个位置，为二级索引，即比一级索引又多了一层索引，大致计算，可以存储的磁盘寻址空间个数可以这么计算：最顶级block可以
    存储4KB/4B=1024个一级索引，而每个一级索引也可以存储4KB/4B个二级索引，因此14位置可以存储的索引个数是1024*1024，可以寻址4GB磁盘空间
    4）inode当中的blocks列表第15个位置，为三级索引，算法同上，可以存储索引的个数是1024*1024*1024(每个4KB可以存储1024个索引)，因此可以寻址
    1024*1024*1024*4KB磁盘空间(4TB)
    因此，ext2单个文件大小是可以支持到4TB，上文讲到bitmap来，bitmap应该是占4byte，32位换算成十进制是4294967296(42亿)，由此单个文件系统可以存储
    40多亿个4KB的块，大致换算是16TB

    以上，由于inode当中的blocks列表以及真实的block块地址，都是在实际有数据写写入的时候才真实分配。稀疏文件便是后分配的一种实现方式。

    再回到cp命令，cp时，有个sparse参数，默认值为auto，就是空洞位置不会拷贝，非空洞位置(包括全0)会全部拷贝到dest，可以制定sparse=always，在auto
    基础上判断是否是全0，如果是则不进行拷贝，物理blocks占用更小，sparse=never，则无论是空洞还是全0，全部拷贝，加入100GB空洞文件，拷贝完后实际
    占了100GB的空间。

    #https://mp.weixin.qq.com/s?__biz=Mzg3NTU3OTgxOA==&mid=2247489347&idx=1&sn=e897fd2f3584fe0fe0c011d4e6503274&chksm=cf3e0786f8498e903b463ac2ddaac2a0fb4cebac7c6cbf02ff02348fbc71dcd80d09a26c4257&cur_album_id=1778430570125967361&scene=190#rd
    fd：
    每个进程里面维护一个打开文件的数组，fd就是这个数组的索引(比如通过/proc/9319/fd可以查看所有打开的文件)，而每个数组值就是一个file struct，包括文件的path、inode、pos(偏移)等
    vfs  inode，实际是屏蔽掉底层各种不同的文件系统，对外提供统一的inode概念，对下定义好接口进行回调注册。
    inode是文件系统提供的，每个文件系统提供inode的内存空间，且内部会内嵌一个vfs inode结构体。同一文件系统内inode不相同，不同文件系统inode可能相同
    inode操作系统仅仅用来在不同文件系统内对文件进行索引查询
    stat出来的是vfs inode？还是文件系统inode？

    #https://mp.weixin.qq.com/s?__biz=Mzg3NTU3OTgxOA==&mid=2247490426&idx=1&sn=4227133d233ea5c86a3b58de64c4804f&chksm=cf3e0bbff84982a931325532b0dbde038d9404455a81bec8e8afed6b2543002e52bc22aa5bd2&cur_album_id=1778430570125967361&scene=190#rd
    如何保证数据安全？
    一般写数据都是write  back方式，数据到达文件系统page cahce(脏页)即返回写成功，然后文件系统按照一定策略再flush到磁盘截至。
    可以使用比如direct_IO方式；write+sync方式；mmap方式来真正写到物理磁盘介质，但是，可能物理磁盘介质也存在缓存......

    #https://mp.weixin.qq.com/s?__biz=Mzg3NTU3OTgxOA==&mid=2247490597&idx=1&sn=e0295b72c7b50e0a5fcf5cca5f967210&chksm=cf3e0ce0f84985f662a40625b2263c48cb419367d70c4614d5cc4dc95c9a57c06035f0e5380d&cur_album_id=1778430570125967361&scene=190#rd
    linux当中的三种"拷贝"：ln 、 cp 、 mv
    文件系统有三个区：超级块区、元数据区、数据区
    文件系统当中有两种文件：regular file和directory，linux stat命令可以区分的，实际是存储在元数据当中i_mode内
    普通文件inode内存储元数据，inode可以索引到block，block里面存储的是用户数据；
    目录文件inode内存储元数据，inode可以索引到block，block里面存储的是目录条目。目录条目(dirent)即是当前目录文件下的字目录或文件的文件名和inode的映射
    根据目录文件的目录条目，可以在内存当中构建除目录树结构，即所说的dentry项(目录项)

    链接文件：软链和硬链都是链接文件，都可以通过链接文件找到背后的源文件、
    软链接：软连接是一个全新的文件，有自己的inode和block，只是block里面存储的是源文件的路径而已，因此可以跨文件系统.
    过程大概是新建一个inode、block，inode存储元数据，block存储源文件的路径，并且在此文件的父目录对应的block当中新增一个目录条目(即软链文件名和inode的对应关系)
    硬链接：硬链接仅仅只是在父目录内新增了一个目录条目(dirent),使用一个新的name指向原来的文件，本身没有新建inode和block，就是说没有新建文件
    由于不同文件系统的inode是独立管理的，因此硬链接不能跨文件系统。
    软链只跟元数据相关，只涉及到目录条目的变化，起不到数据备份的作用。

    mv：同一文件系统+跨文件系统
    同一文件系统：mv命令调用rename系统调用，通常操作是删除源文件所在目录文件中的目录条目(dirent)，在目标目录文件中添加一个新的目录条目(dirent)
        inode number\inode 未变，文件没有做任何的移动；
    跨文件系统：先调用rename，由于跨文件系统，会报错，然后退化一下先调用copy，再调用remove，拷贝源文件，生成一个真正的副本。

    cp：上面以及描述过了，核心就是--sparse 稀疏  参数，默认时auto，即跳过空洞但不跳过零；always跳过稀疏和零，never是既不跳过稀疏，也不跳过零
    
    #https://mp.weixin.qq.com/s?__biz=Mzg3NTU3OTgxOA==&mid=2247489977&idx=1&sn=49c626c57839bee7f90c5fa1646e6ea3&chksm=cf3e097cf849806ae597a9e762a83d6b5d887a3bafae934a75df570f8fe10b886c63a0889c76&scene=178&cur_album_id=1778430570125967361#rd
    磁盘IO为什么总叫你对齐？
    机械硬盘IO要扇区对齐，绝大多数扇区是512byte，磁盘的读写最小单元是扇区；
    ssd盘的IO要4K对齐，ssd盘的读写单元是page，一个page大小是4K。
    机械盘update步骤：先从盘上读取数据到内存，然后update，再协会到磁盘对应位置；
    ssd盘update步骤：无法覆盖写，每次都是些新的位置，旧的位置作为垃圾等待后台GC。

    #https://mp.weixin.qq.com/s?__biz=Mzg3NTU3OTgxOA==&mid=2247490914&idx=1&sn=ba8eb5f875742815997f8b9f193e9120&chksm=cf3e0da7f84984b1823db102616e9bc83d06917f765f0796e98c63d732e2b2255f72f9083cfd&scene=178&cur_album_id=1778430570125967361#rd
    df -Tha T显示文件系统type，h以比较友好的方式
    /etc/fstab 可以指定设备名、label、UUID进行挂载
    /lib/modules/3.10.0-957.el7.x86_64/kernel/fs 内可以看到操作系统支持的文件系统
    文件系统位于内核中，vfs之下，块设备之上，的位置。对外呈现文件存储实现，对内管理裸块设备。用户态程序可以debug，但是内核态程序出现问题就宕机，
    所有现场都丢失，只能通过日志等方式来排查，且内核态的编程要比用户态谨慎的多，因此内核态程序难点在于调试和排障。
    因此要实现一个文件系统，内核开发者提供了一个fuse机制，即把IO路径导向用户态，由用户态捕获到IO，从而实现文件存储。

    #https://mp.weixin.qq.com/s?__biz=Mzg3NTU3OTgxOA==&mid=2247491049&idx=1&sn=9c2558f660c2b09333aa02b8e5127a75&chksm=cf3e0d2cf849843abb5b79480117a0e03e194c6e4a9a28ad6b856a51140230829415c8eab8f0&cur_album_id=1778430570125967361&scene=189#wechat_redirect    
    FUSE是什么？    
    FUSE是一个用来实现用户态文件系统的框架，这个框架包含三个组件：
    1）fuse.ko 内核模块，用来接受vfs传递下来的IO请求，并且把这个IO封装后发送到用户态；
    2）libfuse,用户态lib库，解析fuse内核态发送过来的协议包，拆分成常规的IO请求；【文件系统用户态程序使用，用来解析、封装、运输fuse协议包】
    3）fusermount，mount工具
    以上三个组件完成了一件事情：让IO在内核态和用户态直接来回自由穿梭
    大体流程是，用户对文件系统mountpoint的IO请求，先进入内核，经vfs传递给内核fuse文件系统模块，内核fuse模块把请求发给到用户态程序(用户自己实现的
    用户态文件系统程序)，处理完响应后再原路返回。实际内核fuse模块不是将请求发给用户态程序，而是写入/dev/fuse的虚设备文件，然后用户态程序包含的libfuse有
    个daemon程序监听/dev/fuse，看到有请求写入就立马读出来进行处理。
    那内核模块无法直接和用户态程序交互，需要一个桥梁，此桥梁就是/dev/fuse虚设备文件

    
    #https://mp.weixin.qq.com/s?__biz=Mzg3NTU3OTgxOA==&mid=2247492078&idx=1&sn=0449eb5989a01e170c624561fea5fd2d&chksm=cf3df12bf84a783dcc0b96a1e8f6061d4f46b52be0f2d4a2bcea10a4108ef3bca3548b72b3d3&scene=178&cur_album_id=1778430570125967361#rd
    分布式文件系统的一些收获：
    分布式文件系统架构一般分为client、元数据中心、server三大部分：

    client承接用户请求，然后把请求发到后端服务器，一般有3类分发策略：
    1）轮询：请求依次分发，这样能保证每一个后端 节点都能得到请求；
    2）普通哈希：考量一定的随机性，一般选定一个随机因子hash，比如时间戳、请求数、ip等，然后除以节点数，做取模运算，当节点数变化，所有落点都需要重新计算
    3）一致性哈希：选定随机因子hash，不求余，此方法节点围绕形成一个环，hash值顺时针找它的落点，当有节点down，将down的下一个节点作为新落点，原有的
    落点都不受影响。
    #https://www.jianshu.com/p/bfd1fc49f244
    以上当落点不均衡的时候，就需要均衡迁移，balance，目的就是为了每个节点的数量平衡。

    #https://blog.csdn.net/wxb880114/article/details/81604245 
    副本策略：【多个副本的意思，数据多副本的一致性】
    全写成功返回？还是部分写成功返回？
    WRAO:write all read one，多个副本都写成功，才算成功，读仅仅读一个副本而已。
    quorum:法定人数，写多份返回成功，多的时候也读多份，相比WRAO来说，非强一致性
    raft：
    副本粒度也有几种分类：文件级别、磁盘级别、主机级别
    
    纠删策略：
    一般通过quorum来提高可用性
    
    元数据中心：
    用来存放用户数据的元数据和集群本身的元数据，也可以有服务注册发现功能(集群组建的扩、缩容)，集群拓扑管理等

    服务端：
    用来存储用户数据，管理磁盘空间。
    存储引擎，空间回收，坏盘策略，磁盘管理等

    以上client、元数据、服务端，整体阐述了分布式文件系统的client的核心功能是协议解析，请求均衡转发，冗余策略，修复策略等，服务端的核心功能是存储
    引擎，磁盘管理等。

    #vim的IO存储原理
    vim编辑器使用的依旧是read、write此类系统调用，vim编辑文件时会先read一遍文件，并解码(用于显示到屏幕)，创建一个memfile（memory+.swp文件）
    swp文件用来恢复数据，里面仅仅是一些特定的元数据，当:w保存之前，用户的修改可能在内存，也可能在swp文件，当:w保存是，vim会创建一个~结尾的同文件，把
    原文件拷贝出来，将原文件truncate截断为0，然后从内存+swp拷贝数据，重新写入原文件，删除删除~结尾的备份文件。

    falllocate  预分配和释放磁盘块，能快速生产一个大文件，比dd 0快很多
    truncate 文件打洞，即仅有逻辑空间，而实际物理空间未被占用
    
    

    
    大多来源于”奇犽云存储“公众号
    
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
    stat 显示文件状态信息，其中size是文件大小(多数人看到的文件大小)，单位是byte
    blocks表示物理实际占用了多少个block(512byte)，IO Block表示这个文件占用的总大小，单位是byte

    文件系统：通俗来说就是用来存储数据的一个容器
    文件系统的存储介质是磁盘，本质是就是负责怎么使用磁盘空间，并且保证将文件按照一定的规则存储到磁盘。
    文件系统将磁盘物理空间按照一定粒度切分
    
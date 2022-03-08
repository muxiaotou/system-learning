    https://www.cnblogs.com/cslunatic/p/3683117.html
    本质上mount的过程是inode被替代的过程。
    比如，mount -t ext3 /dev/sdb /mnt/alan,这个过程需要解决的问题就是将
    /mnt/alan的dentry目录项指向的inode屏蔽掉，重新定位到/dev/sdb所表示的inode索引节点

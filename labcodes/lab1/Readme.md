# 第一步： 分析makefile

## makefile的结构：  
1～138行：定义各种变量/函数，设置参数，进行准备工作。  
140~153行：生成bin/kernel  
155~170行：生成bin/bootblock  
172~176行：生成bin/sign  
178~188行：生成bin/ucore.img  
189~201行：收尾工作/定义变量  
203~269行：定义各种make目标

## 精细的分析

<code>
V := @
</code>

变量V=@，后面大量使用了V  

@的作用是不输出后面的命令，只输出结果  

在这里修改V即可调整输出的内容  

也可以 make "V=" 来完整输出 


function.mk中定义了大量的函数。
.mk中每个函数都有注释。

<code>
include tools/function.mk
</code>

call函数：call func,变量1，变量2,...

listf函数在function.mk中定义，列出某地址（变量1）下某些类型（变量2）文件

listf_cc函数即列出某地址（变量1）下.c与.S文件

<code>
listf_cc = $(call listf,$(1),$(CTYPE))
</code>

<!--这是lab1_result编译问题的分析和解决过程-->
在用make编译lab1_result时候遇到了一下问题

<code>
...

'obj/bootblock.out' size: 620 bytes  
620 >> 510!!  
Makefile:152: recipe for target 'bin/bootblock' failed
...
</code>
然后make中止，不能继续运行，之后的make grade等操作也同样不能运行

从这两句的输出来看，是可以看出是来源在sign.c文件中定义的st的大小大于510字节导致的  
查看tools/ 目录下的sign.c文件， 可以看到如下代码

    #include <sys/stat.h>

    int
    main(int argc, char *argv[]) {
        struct stat st;
        if (argc != 3) {
            fprintf(stderr, "Usage: <input filename> <output filename>\n");
            return -1;
        }
        if (stat(argv[1], &st) != 0) {
            fprintf(stderr, "Error opening file '%s': %s\n", argv[1], strerror(errno));
            return -1;
        }
        printf("'%s' size: %lld bytes\n", argv[1], (long long)st.st_size);
        if (st.st_size > 510) {
            fprintf(stderr, "%lld >> 510!!\n", (long long)st.st_size);
            return -1;
    }
 
可以看出这里定义了一个结构提变量 st。  
我们转到这个结构体定义的地方  

    struct stat
    {
        __dev_t st_dev;		/* Device.  */
    #ifndef __x86_64__
        unsigned short int __pad1;
    #endif
    #if defined __x86_64__ || !defined __USE_FILE_OFFSET64
        __ino_t st_ino;		/* File serial number.	*/
    #else
        __ino_t __st_ino;			/* 32bit file serial number.	*/
    #endif
    #ifndef __x86_64__
        __mode_t st_mode;			/* File mode.  */
        __nlink_t st_nlink;			/* Link count.  */
    #else
        __nlink_t st_nlink;		/* Link count.  */
        __mode_t st_mode;	
        ......


我们可以看出这里的结构体定义和系统还有gcc版本有关和过程中的环境声明有关，考虑到这个lab1_result的代码在老师给的实验虚拟机上运行是ok的，所以猜测是gcc的版本问题， 我的debian系统上用的是比较新的gcc-6，而实验的环境中的gcc是4.9版本。

故这里尝试使用低版本的gcc

首先， 安装  
在/etc/apt/source.list 文件后面增加  

deb http://ftp.us.debian.org/debian/ jessie main contrib non-free
deb-src http://ftp.us.debian.org/debian/ jessie main contrib non-free

然后运行 sudo apt install gcc-4.9

将lab1工程中的Makefile 中的gcc配置进行修改  

<code>
CC		:= $(GCCPREFIX)gcc-4.9
</code>

再次运行make clean 和 make  
success!




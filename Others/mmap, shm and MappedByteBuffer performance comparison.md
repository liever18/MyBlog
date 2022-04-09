# 【共享内存】mmap、shm和MappedByteBuffer性能比较

## 引子

由于近期需要写一个java程序和一个已有的c程序（AFL）进行本地通信，使用udp发送数据感觉效率不是很高。另外afl本身使用shm共享物理内存获得待测试子程序的执行路径信息，而java程序共享内存只有文件映射内存的方式，类似与c里面的mmap。因此考虑将AFL的共享内存机制改为mmap，本文将测试linux c 中 mmap、shm和java nio MappedByteBuffer性能，来评估修改AFL共享内存方式是否会影响其本身的性能。

## 共享内存背景

### mmap与shm

内存映射函数mmap, 它把文件内容映射到一段内存上(准确说是虚拟内存上),  通过对这段内存的读取和修改,  实现对文件的读取和修改, mmap()系统调用使得进程之间可以通过映射一个普通的文件实现共享内存。普通文件映射到进程地址空间后，进程可以向访问内存的方式对文件进行访问，不需要其他系统调用(read, write)去操作。

mmap图示例：

![image-20211109102546315](https://fujx98.oss-cn-beijing.aliyuncs.com/img/202204091517043.png)



shm直接将进程空间的虚拟内存与实际物理内存做了映射。

shm图例：

![image-20211109102731301](https://fujx98.oss-cn-beijing.aliyuncs.com/img/202204091517128.png)



总结mmap和shm:

1. mmap是在磁盘上建立一个文件，每个进程地址空间中开辟出一块空间进行映射。而对于shm而言，shm每个进程最终会映射到同一块物理内存。shm保存在物理内存，这样读写的速度要比磁盘**理论上**（后面会结合实验结果分析原因）要快，但是存储量不是特别大。
2. 相对于shm来说，mmap更加简单，调用更加方便，所以这也是大家都喜欢用的原因。
3. 另外mmap有一个好处是当机器重启，因为mmap把文件保存在磁盘上，这个文件还保存了操作系统同步的映像，所以mmap不会丢失，但是shmget就会丢失。



## 测试

### 实验环境

linux服务器用kvm开的虚拟机，因此实际测试可能不会特别准确，开的其他虚拟机在24h跑另外的实验。

4核 16G内存 

### 测试项

- 随机写入1字节
- 清零共享内存

### 64KB共享内存大小 无负载

|               | gcc + mmap | gcc + shm | gcc O3 + mmap | gcc O3 + shm | java MappedByteBuffer |
| ------------- | ---------- | --------- | ------------- | ------------ | --------------------- |
| 1e9次随机写入 | 9.535s     | 9.716s    | 8.761s        | 9.548s       | 11.672s               |
| 1e7次清零     | 20.621s    | 19.952s   | 20.537s       | 20.362s      | 160.178s              |

可以看出在64K共享内存的大小下，随机写入mmap和shm在性能上没有明显差异，MappedByteBuffer会慢个将近20%。但是MappedByteBuffer在大内存清零操作上效率很低。

### 64KB共享内存大小，有25%左右的CPU负载

额外运行了一个程序，该程序用top看大概稳定在25%的cpu占用。

|               | gcc + mmap | gcc + shm | gcc O3 + mmap | gcc O3 + shm | java MappedByteBuffer |
| ------------- | ---------- | --------- | ------------- | ------------ | --------------------- |
| 1e9次随机写入 | 9.478s     | 9.616s    | 8.262s        | 9.634s       | 11.036s               |
| 1e7次清零     | 20.097s    | 20.239s   | 19.713s       | 19.252s      | 142.245s              |

该实验本来我是期望shm会比mmap效果好，因为想着mmap测试程序的内存可能会被其他程序给替换出去，后面发现这个情况的话shm测试程序也避免不了。但是总体实验结果比无负载情况还要快，还是让我觉得奇怪，只能将原因归其为虚拟机运行不能完全做到资源隔离。建议大家在自己电脑上跑一下看看结果。

### 1G共享内存大小

|               | gcc + mmap | gcc + shm | java MappedByteBuffer |
| ------------- | ---------- | --------- | --------------------- |
| 1e9次随机写入 | 758.426s   | 149.275s  | 805.912s              |

这个实验结果来看，shm的性能远超mmap，MappedByteBuffer比mmap慢了不到10%，比之前有进步，也许是JIT长时间运行慢慢优化过来了。

shm比mmap好这么多的原因我分析是由于缓存肯定远远不够1G，所以涉及到了大量的缓存替换的操作，由于shm是直接把物理内存的内容放入缓存，速度肯定比文件映射的内容放入缓存快得多。

## 总结

根据实验结果，有以下结论

1. 共享内存在映射大小不大的情况下，效率非常高，mmap和shm甚至MappedByteBuffer都有不错的效率。循环1e9次随机写入操作的时间开销大概在10s量级，假设cpu频率为2GHz，那么一次循环内，循环操作（i++， i < 1e9）、生成随机数和写入操作一共只用了20个机器周期的时间。
2. 共享内存在映射大小较大的情况下，shm效率比mmap更高，但是由于shm直接占用了物理内存，这样可能会影响其他程序的效率，因为可以使用的物理内存变少了，增加了换页的几率。但是，由于mmap映射的是虚拟内存，它可以开辟的空间上限比shm高，而且会保存到文件中，可操作性强于shm。



## 附 测试代码

```c
// test_mmap.c
#include<stdio.h>
#include<stdlib.h>
#include<string.h>
#include<unistd.h>
#include<sys/types.h>
#include<sys/time.h>
#include<sys/mman.h>
#include<fcntl.h>

#define SIZE (1 << 16) // 64KB
#define TEST_NUM 10000000 // 1e7
int main() {
    int fd = open("share-memory-file", O_RDWR | O_CREAT | O_TRUNC, 0644);
    ftruncate(fd, SIZE);
    // 以一个字节为单位
    char *p = mmap(NULL, SIZE, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0); 

    close(fd);
    
    struct timeval start, end;  
    double during_time;
    
    // test random write
    gettimeofday(&start, NULL); 
    char to_write = 0b01010101;
    for (long long i = 0; i < TEST_NUM * 100; i++) {
        int pos = rand() % SIZE;
        memcpy(p + pos, &to_write, sizeof(char));

    }
    gettimeofday(&end, NULL); 
    during_time = end.tv_sec - start.tv_sec + (double)end.tv_usec / 1000000 - (double)start.tv_usec / 1000000;
    printf("test random write cost %lf sec\n", during_time);

    // test set zero
    gettimeofday(&start, NULL); 
    for (long long i = 0; i < TEST_NUM; i++) {
        memset(p, 0, SIZE);
    }
    gettimeofday(&end, NULL); 
    during_time = end.tv_sec - start.tv_sec + (double)end.tv_usec / 1000000 - (double)start.tv_usec / 1000000;
    printf("test set zero cost %lf sec\n", during_time);

    int ret = munmap(p, SIZE);
    if(ret < 0) {
        perror("mmumap");
        exit(1);
    }

    return 0;
}
```



```c
// test_shm.c
#include<stdio.h>
#include<stdlib.h>
#include<string.h>
#include<unistd.h>
#include<sys/time.h>
#include<sys/shm.h>
#include<sys/types.h>
#include<sys/mman.h>
#include<fcntl.h>

#define SIZE (1 << 16) // 64KB
#define TEST_NUM 10000000 // 1e7

int main() {

    int shm_id = shmget(IPC_PRIVATE, SIZE, IPC_CREAT | IPC_EXCL | 0600);
    char* p = shmat(shm_id, NULL, 0);

    struct timeval start, end;  
    double during_time;
    
    // test random write
    gettimeofday(&start, NULL); 
    char to_write = 0b01010101;
    for (long long i = 0; i < TEST_NUM * 100; i++) {
        int pos = rand() % SIZE;
        memcpy(p + pos, &to_write, sizeof(char));

    }
    gettimeofday(&end, NULL); 
    during_time = end.tv_sec - start.tv_sec + (double)end.tv_usec / 1000000 - (double)start.tv_usec / 1000000;
    printf("test random write cost %lf sec\n", during_time);

    // test set zero
    gettimeofday(&start, NULL); 
    for (long long i = 0; i < TEST_NUM; i++) {
        memset(p, 0, SIZE);
    }
    gettimeofday(&end, NULL); 
    during_time = end.tv_sec - start.tv_sec + (double)end.tv_usec / 1000000 - (double)start.tv_usec / 1000000;
    printf("test set zero cost %lf sec\n", during_time);

    shmctl(shm_id, IPC_RMID, NULL);

    return 0;
}
```



```java
// ShareMemory.java
import java.io.File;
import java.io.IOException;
import java.io.RandomAccessFile;
import java.nio.ByteBuffer;
import java.nio.MappedByteBuffer;
import java.nio.channels.FileChannel;
import java.util.Random;

public class ShareMemory {
    int mapSize = 1 << 16;
    String shareFileName="mem";             //共享内存文件名
    String sharePath="sm";                  //共享内存路径
    MappedByteBuffer mapBuf = null;         //定义共享内存缓冲区
    FileChannel fc = null;                  //定义相应的文件通道
    RandomAccessFile RAFile = null;         //定义一个随机存取文件对象
    byte[] buff = new byte[1];
    public static Random rand = new Random();
    public ShareMemory() {
        File folder = new File(sharePath);
        if (!folder.exists()) {
            folder.mkdirs();
        }
        this.sharePath+=File.separator;

        try {
            File file = new File(this.sharePath + this.shareFileName+ ".sm");
            if(file.exists()){
                RAFile = new RandomAccessFile(this.sharePath + this.shareFileName + ".sm", "rw");
                fc = RAFile.getChannel();
            }else{
                RAFile = new RandomAccessFile(this.sharePath + this.shareFileName + ".sm", "rw");
                //获取相应的文件通道
                fc = RAFile.getChannel();

                byte[] bb = new byte[mapSize];
                //创建字节缓冲区
                ByteBuffer bf = ByteBuffer.wrap(bb);
                bf.clear();
                //设置此通道的文件位置。
                fc.position(0);
                //将字节序列从给定的缓冲区写入此通道。
                fc.write(bf);
                fc.force(false);

                //将此通道的文件区域直接映射到内存中。
            }
            mapBuf = fc.map(FileChannel.MapMode.READ_WRITE, 0, mapSize);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
    public void closeSMFile() {
        if (fc != null) {
            try {
                fc.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
            fc = null;
        }

        if (RAFile != null) {
            try {
                RAFile.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
            RAFile = null;
        }
        mapBuf = null;
    }
    public int reset(){
        try {
            mapBuf.position(0);
            mapBuf.put(new byte[mapSize], 0, mapSize);
            return 1;

        } catch (Exception e) {
            e.printStackTrace();
            return 0;
        }
    }
    public static void main(String[] argv) {
        ShareMemory sm = new ShareMemory();
       
        byte to_write = 0b01010101;
        long startTime;
        long endTime;
        
 		// test random write
        startTime = System.currentTimeMillis();
        for (int i = 0; i <1000000000; i++) {
            int pos =  rand.nextInt(sm.mapSize);
            sm.mapBuf.position(pos);
            sm.mapBuf.get(sm.buff, 0, 1);

            sm.buff[0] = to_write;
            sm.mapBuf.position(pos);
            sm.mapBuf.put(sm.buff, 0, 1);
        }
        endTime = System.currentTimeMillis();
        System.out.println("test random write cost: " + (endTime - startTime) + "ms");
 		// test set zero
        startTime = System.currentTimeMillis();
        for (int i = 0; i < 10000000; i++) {
            sm.reset();
        }
        endTime = System.currentTimeMillis();
        System.out.println("test set zero cost: " + (endTime - startTime) + "ms");
        
    } 
}
```

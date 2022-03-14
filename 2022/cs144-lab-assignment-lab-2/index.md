# CS144 Lab Assignment lab-2


## build
首先把`origin/lab2-startercode`合并到本地仓库，`cmake`构建时，有可能会报错：  
>CMake Error: The following variables are used in this project, but they are set to NOTFOUND.

这是因为缺少`libpcap-dev`库，安装即可：  
```shell
sudo apt-get install libpcap-dev
```

-----

## The TCP Receiver
本次实验是实现TCP的接收端。
- `segment_received()`函数接收TCP报文段`TCP segment`并交由lab0和lab1实现的`ByteStream`和`StreamReassembler`使用
- `ackno()`函数提供了TCP接收端下一个需要接收的`sequence number`
- `window_size()`函数提供了TCP接收端的接收窗口大小
- `WrappingInt32`类实现了一个32位无符号整型的封装，用来表示32位的`sequence number`
- `wrap() unwrap()`函数实现了64位`stream index`和32位`sequence number`之间的转换

-----

## Translating between 64-bit indexes and 32-bit seqnos
之前我们实现的`StreamReassembler`流重组器，它能够将乱序、重复的字节流重新组合，它为每个字节提供一个64位的字节索引，$2^{64}$的范围足够的大，可以将其看作是永不溢出。  
然而在TCP报头中，我们只有32位的空间来表示每个字节的“序列号（`sequence number`）”。  
这就导致我们需要在64位的`stream index`和32位的`sequence number`之间进行转换：
- 从`TCP segment`中读取字节数据时需要将`seqno`转换为整个字节流中的`stream index`，然后将字节索引和数据交由`StreamReassembler`处理
- 在提供`ackno`时，需要将流中的`stream index`转换为`seqno`以返回给TCP的发送端

在实现之前，我们需要知道几个要点：
- TCP中的字节流可以是任意长度，而$2^{32}$字节只有4GiB大小，`seqno`范围是0~$2^{32}-1$，$2^{32}-1$下一个`seqno`为0
- TCP序列号不是从0开始，而是从一个随机的32位初始序列号`ISN`开始，这是表示流的开始（`SYN`）的序列号，第一个数据字节从`ISN + 1`开始
- 流的逻辑起点`SYN`和逻辑终点`FIN`各占用一个`seqno`

我们考虑以0为`SYN`的序列号，称之为绝对序列号`absolute sequence number`，对理解`seqno`和`stream index`的转换有帮助。下图是文档里的解释：  
![](/post_images/posts/Networking/CS144LabAssignmentlab-2/abs_seqno.jpg "seqno-abs_seqno-stream_index")

-----

### absolute seqno(64 bit) → seqno(32 bit)
从`seqno`到`abs_seqno`只需要`abs_seqno`对$2^{32}$取模，再加上`isn`即可。  
代码：  
```cpp
WrappingInt32 wrap(uint64_t n, WrappingInt32 isn) {
    // seqno = isn + n % 2^32
    return WrappingInt32{isn + static_cast<uint32_t>(n)};
}
```
*`static_cast<uint32_t>`将`uint64_t`转换为32位，这种方式会截取低32位数据，相对于对$2^{32}$取模*

-----

###  seqno(32 bit) → absolute seqno(64 bit)
从`seqno`转换到`abs_seqno`需要考虑更多一些。  
我们需要待转换序列号`n`，初始序列号`isn`，以及一个`checkpoint`（下面会解释它的作用）。  
过程：  
1. 算出`n`到`isn`的偏移量`offset`（这里使用32位无符号减法，如`isn = 2^32 - 2` `n = 3`，`offset = n - isn = 3 - (2^32 - 2) = 5`）
2. 我们找到了`offset`之后，不能直接得到`abs_seqno = offset`，这是因为从`abs_seqno`到`seqno`需要模$2^{32}$运算，所以真实的`abs_seqno`可能是`offset`、`offset + 2^32`、`offset + 2 * 2^32*`……我们需要借助`checkpoint`来确定到底是哪个
3. *`checkpoint`可以取为已经收到的最后一个字节的`abs_seqno`，这是因为TCP接收端当前收到的`TCP seg`的`seqno`几乎不可能和`checkpoint`相差超过$2^{32}$字节*，我们将`checkpoint`表示为`A * 2^32 + B`，那么`abs_seqno`只能为：`(A - 1) * 2^32 + offset`、`A * 2^32 + offset`、`(A + 1) * 2^32 + offset`中最靠近`checkpoint`的那个，值得注意的是，如果`A == 0`那么不考虑`(A - 1) * 2^32 + offset`的情况

代码：  
```cpp
uint64_t unwrap(WrappingInt32 n, WrappingInt32 isn, uint64_t checkpoint) {
    uint64_t offset = n.raw_value() - isn.raw_value();
    // checkpoint = A * 2^32 + B
    uint64_t A = checkpoint >> 32;
    // num1 = (A - 1) * 2^32 + offset
    uint64_t num1 = ((A - 1) << 32) + offset;
    uint64_t abs1 = checkpoint - num1;
    // num2 = A * 2^32 + offset
    uint64_t num2 = (A << 32) + offset;
    uint64_t abs2 = checkpoint > num2 ? checkpoint - num2 : num2 - checkpoint;
    // num3 = (A + 1) * 2^32 + offset
    uint64_t num3 = ((A + 1) << 32) + offset;
    uint64_t abs3 = num3 - checkpoint;
    // A == 0 只需比较abs2 abs3
    if (A == 0)
        return abs2 < abs3 ? num2 : num3;
    // 比较abs1 abs2 abs3
    if (abs1 < abs2)
        return abs1 < abs3 ? num1 : num3;
    return abs2 < abs3 ? num2 : num3;
}
```

-----

### Translating测试结果
![](/post_images/posts/Networking/CS144LabAssignmentlab-2/Translating测试结果.jpg "Translating测试结果")

-----

## Implementing the TCP receiver
完成32-64位序列号转换后，我们就可以开始实现`tcp receiver`，我们要做的事情有：  
- 从发送方处接收`TCP segment`
- 使用`StreamReassembler`重新组装`ByteStream`
- 计算确认号`ack`和窗口大小`window_size`

-----

### segment_received()
在接收`TCP segment`的时候，我们首先要检查，TCP报头中的：`seqno`、`SYN`、`FIN`。这是我们这次实验要考虑的字段。  
![](/post_images/posts/Networking/CS144LabAssignmentlab-2/TCPsegment.jpg "TCPsegment")  
在这个部分，我们要做的事情有：  
- 检查`syn`，并设置初始序列号`isn`
- 检查`fin`，判断该段是否最后字节是否是流的最后字节
- 将`TCP segment`携带的数据推送到`StreamReassembler`

可以看一下文档中的图示：  
![](/post_images/posts/Networking/CS144LabAssignmentlab-2/TCPreceiver.jpg "TCPreceiver")  

代码：  
```cpp
void TCPReceiver::segment_received(const TCPSegment &seg) {
    TCPHeader header = seg.header();
    string data = seg.payload().copy();
    bool eof = false;
    // listening for syn
    if (!_syn_flag && !header.syn)
        return;
    // recive first syn
    if (!_syn_flag && header.syn) {
        _syn_flag = true;
        _isn = header.seqno;
        if (header.fin) { // 第一次握手 FIN也可能为true
            _fin_flag = true;
            eof = true;
        }
        _reassembler.push_substring(data, 0, eof); // 握手时也可能携带数据
        return;
    }
    // 如果接收的seg FIN为true
    if (header.fin) {
        _fin_flag = true;
        eof = true;
    }
    // 要将seg header.seqno 转换为 stream index
    // chekpoint 是一个 abs_seqno 这里赋值为 已经收到的最后一个字节的abs_seqno（stream index + 1）
    uint64_t checkpoint = stream_out().bytes_written();
    uint64_t abs_seqno = unwrap(header.seqno, _isn, checkpoint);
    uint64_t stream_index = abs_seqno - 1;
    _reassembler.push_substring(data, stream_index, eof);
}
```

-----

### ackno()
在收到发送方的`TCP segment`后，接收方需要返回一个确认号`ackno`来告诉发送方需要的下一个`seg`的序列号，即是下一个要接收的字节的`seqno`。我们要做的如下：  
1. 如果没有同步，`syn flag == false`，则返回空
2. 否则，通过`StreamReassembler`已经接收的字节数确定`abs_seqno`，然后根据`isn`将其转换为`seqno`返回

*这里我们要注意`syn`和`fin`是要占用一个`seqno`的*
代码：  
```cpp
optional<WrappingInt32> TCPReceiver::ackno() const { 
    if (!_syn_flag)
        return nullopt;
    // abs_seqno是已经收到的字节数和SYN（SYN占用0号abs_seqno）
    uint64_t abs_seqno = stream_out().bytes_written();
    // 如果收到FIN且ByteStream关闭则FIN占用一个abs_seqno
    if (_fin_flag && stream_out().input_ended())
        ++ abs_seqno;
    return WrappingInt32(wrap(abs_seqno + 1, _isn)); 
}
```

-----

### window_size()
允许发送方发送的窗口大小，即为`StreamReassembler`中`ByteStream`的缓冲区剩余大小。  
代码：  
```cpp
size_t TCPReceiver::window_size() const { return stream_out().remaining_capacity(); }
```

-----

## lab2测试结果
![](/post_images/posts/Networking/CS144LabAssignmentlab-2/lab2测试结果.jpg "lab2测试结果")  

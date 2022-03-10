---
title: CS144 Lab Assignment lab-1
date: 2022-03-10
tags: [network, C++]
categories: [Networking]
---

## stitching substrings into a byte stream
本次实验是要实现一个流重组器，能够将`TCPReceiver`接收到的乱序的、重复的`TCP段`重新组装成一个正确的、按序的、无重复的字节流`ByteStream`。  
我们需要完成`StreamReassembler`类的编写，主要功能在`push_substring`中实现，该函数接收一个子字符串、该子串第一个字节在整个流中的字节索引（从0开始）和一个`eof`标志（用来标记该子串的最后一个字节是整个流的结尾）。

该实验看上去简单，但在细节部分十分繁琐，需要认真阅读文档，搞清楚所有细节问题在动手编写代码。

## capacity
lab1中最重要的概念就是`capacity`，必须正确理解其含义。  
下图详细展示了`capacity`的含义，  
- stream start 到 first unread 的蓝色部分代表已经被`ByteStream`读取的部分（已经不在`ByteStream`中）
- first unread 到 first unassembled 的绿色部分代表保存在`ByteStream`的缓冲区中还未被读取的部分
- first unassembled 到 first unacceptable 部分，是`StreamReassembler`中能够保存的部分，红色片段代表在整个流中的该片段已经保存到`StreamReassembler`中，空白部分代表那些还没收到的部分
- 超过first unacceptable的字节应该被丢弃，而不继续保存在`StreamReassembler`中，这是为了避免在极限条件下的内存占用问题

经过上述的分析，我们可以弄清楚`capacity`到底是什么，显然，`capacity`规定了`ByteStream`的大小，它的缓冲区不能超过这个大小，这就意味着，`StreamReassembler`中保存的字节数，不能超过`ByteStream`中剩余的缓冲区大小。  
![](/post_images/posts/Networking/CS144LabAssignmentlab-1/capacity.jpg "capacity")

## StreamReassembler实现
### 数据结构
很明显，我们需要一种数据结构来记录到目前为止，我们收到的`substring`。  
我们可以使用`map<size_t, string>`来记录字节索引和子串的键值对，并且map容器底层由红黑树实现，可以排序方便插入删除，但是接收到的字符串彼此之间可能重叠、覆盖，使用这种结构会涉及许多字符串截取合并的操作，非常繁琐QAQ。  
我使用的方法是：用两个双端队列`deque`分别记录`StreamReassembler`中接收的字符（char）以及字符的字节索引。
```cpp
std::deque<char> _unassembled_chars{};
std::deque<bool> _unassembled_flags{};
```
### 边界处理
我们只需要留下在允许范围内的字节，小于下界或是大于上界的字节直接丢弃即可。
### eof
在允许范围内的字节需要考虑eof标记，小于下界的eof表示已经到结尾，而大于上界的eof无需考虑。

### 代码
代码：  
```cpp
#include "byte_stream.hh"

#include <cstdint>
#include <string>
#include <deque>

//! \brief A class that assembles a series of excerpts from a byte stream (possibly out of order,
//! possibly overlapping) into an in-order byte stream.
class StreamReassembler {
  private:
    std::deque<char> _unassembled_chars{};
    std::deque<bool> _unassembled_flags{};
    size_t _next_assemble_index{0};
    size_t _unassembled_bytes{0};
    bool _eof_flag{false};
    
    ByteStream _output{0};  //!< The reassembled in-order byte stream
    size_t _capacity{0};    //!< The maximum number of bytes
...
	
	
	
// 下面是 stream_reassembler.cc 内容
	
#include "stream_reassembler.hh"

using namespace std;

StreamReassembler::StreamReassembler(const size_t capacity)
    : _unassembled_chars(capacity, '\0'), // 初始化为空字符
    _unassembled_flags(capacity, false),  // 初始化为false
    _output(capacity), 
    _capacity(capacity) {}

void StreamReassembler::push_substring(const string &data, const size_t index, const bool eof) {
    // 我们在_unassembled_chars中可以容纳的字节数，不能多于ByteStream中剩余缓冲区的容量
    // 如果当前串的起始位置，超过容纳范围，直接丢弃该串
    if (index >= _next_assemble_index + _output.remaining_capacity())
        return;
    // 如果当前串所有字节，都已经被写入ByteStream，则丢弃该串
    if (index + data.size() <= _next_assemble_index) {
        if (eof && empty()) // 如果eof标记为true，表明后续已无字节需要写入，关闭ByteStream输入
            _output.end_input();
        return;
    }
    // 只有当前串的最后一个字节落在_unassembled_chars的范围内，且eof==true时，才将_eof_flag置为true
    // 标记当前串的最后一个字节是否在_unassembled_chars的范围内
    bool flag_in_bounds = true;
    if (index + data.size() > _next_assemble_index + _output.remaining_capacity())
        flag_in_bounds = false;
    // 遍历当前串，对在_unassembled_chars范围内的字节进行判断
    for (size_t i = index; i < index + data.size() && i < _next_assemble_index + _output.remaining_capacity(); ++ i) {
        if (i >= _next_assemble_index) {
            if (_unassembled_flags[i - _next_assemble_index] == false) { // 如果该位置为空，则写入字节
                _unassembled_chars[i - _next_assemble_index] = data[i - index];
                _unassembled_flags[i - _next_assemble_index] = true;
                ++ _unassembled_bytes;
            }
        }
    }
    // 把_unassembled_chars中开头的字节写入ByteStream
    // 因为整个_unassembled_chars的大小不大于ByteStream缓冲区的剩余容量，所以不用担心写入的字节被丢弃
    string write_string{};
    for (size_t i = _next_assemble_index; _unassembled_flags[i - _next_assemble_index] == true; ++ i) {
        write_string += _unassembled_chars[i - _next_assemble_index];
        _unassembled_chars.pop_front();
        _unassembled_chars.push_back('\0');
        _unassembled_flags.pop_front();
        _unassembled_flags.push_back(false);
        ++ _next_assemble_index;
        -- _unassembled_bytes;
    }
    _output.write(write_string);
    // 判断是否需要置位_eof_flag以及是否需要关闭ByteStream输入
    if (flag_in_bounds && eof)
        _eof_flag = true;
    if (_eof_flag && empty())
        _output.end_input();
}

size_t StreamReassembler::unassembled_bytes() const { return _unassembled_bytes; }

bool StreamReassembler::empty() const { return _unassembled_bytes == 0; }
```

### 测试结果
（我还优化了一下lab0的代码，这下测试用时很少了）  
![](/post_images/posts/Networking/CS144LabAssignmentlab-1/lab1测试结果.jpg "lab1测试结果")
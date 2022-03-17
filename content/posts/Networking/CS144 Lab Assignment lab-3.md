---
title: CS144 Lab Assignment lab-3
date: 2022-03-17
tags: [network, C++]
categories: [Networking]
---

## the TCP sender
本次实验是TCP的sender，需要完成的主要功能有：  
- 跟踪接收方的接收窗口，正确处理接收方返回的确认号`ackno`和窗口大小`window_size`
- 发送方从字节流`ByteStream`中读取字节，并包装成`TCPSegment`，用以填充接收窗口，发送方应该保持持续发送，直到接收窗口大小为0，或字节流为空
- 将发送过但未收到确认的`segment`缓存下来，直到收到接收方返回的确认
- 为发送过的TCP段进行计时，在一段时间后还没有收到确认，则认为该段在网络中丢失，此时应该重新向接收方发送该段

需要完成的接口如下：  
- `fill_window()`：向接收方发送组装的`TCPSegment`
- `ack_received()`：接收来自接收方的确认号和窗口大小，移除缓存区中已经确认的`segment`，并调用`fill_window()`填充接收窗口
- `tick()`：该函数是本次实验中唯一的获得时间流逝的函数，用来判断`segment`是否超时需要重发
- `send_empty_segment()`：发送一个空的`segment`

-----

## How does the TCPSender know if a segment was lost?
TCPSender怎么知道一个`segment`在网络中丢失了呢？  
TCP协议使用**超时重传**机制，简单来说，对于每个发送出去的`segment`，都将其保存在缓存区中，直到收到该`segment`的**完全确认**，再把它从缓存区删除。发送时，对其进行计时，如果当超过某个超时时间时，我们对其进行重发操作。  
需要注意的是：在实验中我们使用**完全确认**，也就是说，对于每个等待确认的`segment`，我们不会确认它的部分，而是只有在收到的确认号完全覆盖该`segment`时，才确认它。

超时重传机制的实现比较复杂，下面是几个关键点：  
- 每隔几个毫秒，`TCPSender`的`tick()`函数就会被调用（*被谁调用？被该TCP实现的更高层调用*），它会传回一个参数，代表从上次调用`tick()`之后经过了多少个毫秒，这是我们用来记录时间流逝并判断超时的方法。除此之外，我们不能调用系统提供的其他`time`或`clock`相关的接口
- 在构造`TCPSender`时，会从参数中得知**超时重传时间RTO**的初始值，RTO会随网络情况的改变而变化，但是初始值始终不变
- 需要设计一个**重传定时器**，它需要在RTO超时时处理一些事情，我们需要在一些不同事件发生时启动重传定时器
- 当重传定时器超时时，我们需要做一些事情：
	- 重新发送之前最先发送的且没有收到确认的`segment`
	- 如果接收方的窗口大小 > 0：
		- 启用**二进制指数退避**，将RTO翻倍
		- 超时重发计数增加1，这个量是用来给上层判断连接是否可达的，如果超时重发次数过多，则认为连接已不可达
	- 重置定时器
- 当`TCPSender`收到来自接收方的`ackno`时（要比之前收到的`ackno`大）
	- RTO设为初始值
	- 发送缓存区存在未确认的数据，重启定时器
	- 连续超时重传计数归零（因为收到ack证明网络没有中断）

-----

##  Implementing the TCP sender
在`tcp_sender.hh`和`tcp_sender.cc`中完成接口即可。

注释已经足够详细，就不再赘述：  
```cpp
class TCPSender {
  private:
    //! our initial sequence number, the number for our SYN.
    WrappingInt32 _isn;

    //! outbound queue of segments that the TCPSender wants sent
    std::queue<TCPSegment> _segments_out{};

    //! retransmission timer for the connection
    unsigned int _initial_retransmission_timeout;

    //! outgoing stream of bytes that have not yet been sent
    ByteStream _stream;

    //! the (absolute) sequence number for the next byte to be sent
    uint64_t _next_seqno{0};


    // 定时器
    struct retrans_timer {
      size_t timecount{0}; // 计时
      size_t timeout;      // 当前的RTO
      size_t consecutive_retransmissions_count{0}; // 连续超时重传次数
      retrans_timer(size_t retx_timeout) : timeout(retx_timeout) {}
    } _retrans_timer;

    // 已发送未确认的TCP段
    std::queue<TCPSegment> _outstanding_queue{};
    size_t _outstanding_bytes{0};

    // 接收方窗口大小
    size_t _window_size{1};
    // syn fin 是否发送
    bool _syn_flag{false};
    bool _fin_flag{false};
...
	
	
TCPSender::TCPSender(const size_t capacity, const uint16_t retx_timeout, const std::optional<WrappingInt32> fixed_isn)
    : _isn(fixed_isn.value_or(WrappingInt32{random_device()()}))
    , _initial_retransmission_timeout{retx_timeout}
    , _stream(capacity)
    , _retrans_timer(retx_timeout) {}

uint64_t TCPSender::bytes_in_flight() const { return _outstanding_bytes; }

void TCPSender::fill_window() {
    // 尽管接收方的窗口大小为0 还是要发送报文段以期得到接收方返回的最新窗口大小
    // 我们需要用另一个变量来表示_window_size
    // 因为如果我们将_window_size 视作 1 实际上是超出接收方的接受范围
    // 直接令其为1 在调用tick()时 无法判断超时的原因是网络问题还是接收方窗口为0
    // 会导致错误的二进制指数退避
    size_t win_size = _window_size == 0 ? 1 : _window_size;
    // 循环填充窗口 窗口大小大于以发送的字节数
    while (win_size > _outstanding_bytes) {
        // 构造新报文段
        TCPSegment segment;
        // 如果没有握手 立即发送syn信号
        if (!_syn_flag) {
            segment.header().syn = true;
            _syn_flag = true;
        }
        // 装入序列号
        segment.header().seqno = next_seqno();
        // 装入负载  注意syn会占用一个payload_size
        size_t payload_size = min(TCPConfig::MAX_PAYLOAD_SIZE, win_size - _outstanding_bytes) - segment.header().syn;
        // 从字节流中读取字节
        string payload = _stream.read(payload_size);
        // 判断是否装入fin 装入fin的条件为
        // 1.没发送过fin信号 2.字节流已经读取完毕并关闭
        // 3.接收方能接收的字节数 > 装入负载数 + syn(算一个)
        // 这样才能将fin装入seg
        // 若接收方当前能够接收的字节数 == 装入负载数 + syn
        // 那么fin信号应装在下一个空负载的seg中
        if (!_fin_flag && _stream.eof() && payload.size() + segment.header().syn < win_size - _outstanding_bytes)
            _fin_flag = segment.header().fin = true;
        // 装入负载
        segment.payload() = Buffer(move(payload));
        // 发送数据包的条件是 该数据包有占用seqno (包括 syn fin)
        if (segment.length_in_sequence_space() == 0)
            break;
        // 如果缓存区没有等待确认的seg 我们应为这个新的seg开启定时器
        if (_outstanding_queue.empty()) {
            _retrans_timer.timeout = _initial_retransmission_timeout;
            _retrans_timer.timecount = 0;
        }
        // 发送
        _segments_out.push(segment);
        // 缓存新的seg
        _outstanding_bytes += segment.length_in_sequence_space();
        _outstanding_queue.push(segment);
        // 更新 _next_seqno
        _next_seqno += segment.length_in_sequence_space();
        // 如果发送完毕立即退出fill
        if (segment.header().fin)
            break;
    }
}

//! \param ackno The remote receiver's ackno (acknowledgment number)
//! \param window_size The remote receiver's advertised window size
void TCPSender::ack_received(const WrappingInt32 ackno, const uint16_t window_size) {
    // 获得绝对ack的seqno
    size_t abs_seqno = unwrap(ackno, _isn, _next_seqno);
    // abs_seqno表示该字节之前的所有字节已收到
    // 确认号不能大于待发送字节
    if (abs_seqno > _next_seqno)
        return;
    // 把已经收到确认的seg弹出缓存区
    while (!_outstanding_queue.empty()) {
        const TCPSegment &seg = _outstanding_queue.front();
        // 如果队头seg（最先发送的）完全确认
        if (unwrap(seg.header().seqno, _isn, abs_seqno) + seg.length_in_sequence_space() <= abs_seqno) {
            // 弹出队列
            _outstanding_bytes -= seg.length_in_sequence_space();
            _outstanding_queue.pop();
            // 如果有新的数据包被成功接收 RTO回归初始值 定时器重新计时
            _retrans_timer.timeout = _initial_retransmission_timeout;
            _retrans_timer.timecount = 0;
        }
        // 如果seg没有被完全确认 说明后面的seg也一样
        else
            break;
    }
    // 只要收到了ack 则说明网络并没有中断
    // 那么连续重传计数也应该归零
    _retrans_timer.consecutive_retransmissions_count = 0;
    // 更新窗口大小
    _window_size = window_size;
    // 填充发送窗口
    fill_window();
}

//! \param[in] ms_since_last_tick the number of milliseconds since the last call to this method
void TCPSender::tick(const size_t ms_since_last_tick) {
    // 统计经过的时间
    _retrans_timer.timecount += ms_since_last_tick;
    // 如果存在未确认的seg 且 超时 则重传最先发送的seg
    if (_retrans_timer.timecount >= _retrans_timer.timeout && !_outstanding_queue.empty()) {
        // _window_size > 0 说明接收方还在接收
        // 如果超时则是网络拥堵导致
        // 启动二进制指数退避
        if (_window_size > 0) {
            _retrans_timer.timeout *= 2;
            // 超时则将超时重发计数加一
            ++_retrans_timer.consecutive_retransmissions_count;
        }
        // 重新计时
        _retrans_timer.timecount = 0;
        // 重发
        _segments_out.push(_outstanding_queue.front());
        
    }
}

unsigned int TCPSender::consecutive_retransmissions() const { return _retrans_timer.consecutive_retransmissions_count; }

void TCPSender::send_empty_segment() {
    TCPSegment segment;
    segment.header().seqno = next_seqno();
    _segments_out.push(segment);
}
	
```

-----

## lab3测试结果
![](/post_images/posts/Networking/CS144LabAssignmentlab-3/lab3测试结果.jpg "lab3测试结果")  
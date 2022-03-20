# CS144 Lab Assignment lab-4


## 前言
前后经过半个多月终于把关于TCP的几个lab写完了，*★,°*:.☆(￣▽￣)/$:*.°★* 。  
lab4的难点在于对于TCP各个状态的理解、如何处理各种状态下收到的`segment`、如何进行`clean_shutdown`正确结束TCP连接。

-----

## The TCP connection
本次实验的任务是，将`TCPSender`和`TCPReceiver`整合起来，`TCPConnection`负责TCP的整体连接处理。需要完成的功能如下：  
- 接收`segment`：当TCPConnection调用`segment_received()`函数时，从网络中（CS144封装的SOCKET）接收`TCPSegment`，并检查这个段：
	-  如果设置了`RST(reset)`标志，则将入站流和出站流都设置为错误状态，并永久终止连接
	-  将这个段交由`TCPReceiver`处理，获取`seqno`、`syn`、`payload`和`fin`信息
	-  如果设置了`ACK`标志，则将`ackno`和`win`告知`TCPSender`以更新发送窗口
	-  如果传入的`segment`占用了至少一个`seqno`（如果发来的时空段，则不需要发送ack），则发送至少一个段来告知对方`ackno`和`window_size`的更新
- 发送`segment`：TCPConnection将待发送的`segment`推入发送队列，等待上层调用以传入网络中
	- TCPConnection将`sender.segment_out`队列中的seg推入发送队列时，需要为`segment`添加`ack`、`ackno`和`window_size`
- 当时间流逝：TCPConnection的`tick()`函数将被上层调用，传入的参数将告知TCPConnection自上次`tick()`被调用以来经过了多少毫秒的时间
	- TCPConnection将告知`TCPSender`经过的时间
	- 如果sender超时重传的次数超过了限定值， 远端peer将会被认为不可达，此时应该向远端peer发送一个seg并带有`RST`标志
	- 如果时机合适，正常地断开连接（`clean_shutdown`）

-----

## 重中之重
要完全理解TCP协议的状态信息，和讲义中的内容，`segment_received()`和`clean_shutdown()`有很多对TCP连接状态进行判断的地方，也是我认为的本次实验的难点，其他的部分基本就比较简单了。  
再次回顾一下TCP的连接状态：（图是偷来的^^）  
![](/post_images/posts/Networking/CS144LabAssignmentlab-4/TCP建立连接.jpg "TCP建立连接")  
![](/post_images/posts/Networking/CS144LabAssignmentlab-4/TCP断开连接.jpg "TCP断开连接")  
![](/post_images/posts/Networking/CS144LabAssignmentlab-4/EvolutionofTCPreceiver.jpg "Evolution of the TCP receiver")  
![](/post_images/posts/Networking/CS144LabAssignmentlab-4/EvolutionofTCPsender.jpg "Evolution of the TCP sender")  

-----

## Implementing the TCP Connection
注释足够详细，不在赘述。  
代码：  
```cpp
class TCPConnection {
  private:
    TCPConfig _cfg;
    TCPReceiver _receiver{_cfg.recv_capacity};
    TCPSender _sender{_cfg.send_capacity, _cfg.rt_timeout, _cfg.fixed_isn};

    //! outbound queue of segments that the TCPConnection wants sent
    std::queue<TCPSegment> _segments_out{};

    //! Should the TCPConnection stay active (and keep ACKing)
    //! for 10 * _cfg.rt_timeout milliseconds after both streams have ended,
    //! in case the remote TCPConnection doesn't know we've received its whole stream?
    bool _linger_after_streams_finish{true};

	// 我新增的私有成员
    size_t _time_since_last_segment_received{0};
    bool _active_flag{true};
    void send_segments();
    void try_clean_shutdown();
    void unclean_shutdown();
...
	
	
size_t TCPConnection::remaining_outbound_capacity() const { return _sender.stream_in().remaining_capacity(); }

size_t TCPConnection::bytes_in_flight() const { return _sender.bytes_in_flight(); }

size_t TCPConnection::unassembled_bytes() const { return _receiver.unassembled_bytes(); }

size_t TCPConnection::time_since_last_segment_received() const { return _time_since_last_segment_received; }

void TCPConnection::segment_received(const TCPSegment &seg) { 
    if (!active())
        return;
    // 收到seg更新定时器
    _time_since_last_segment_received = 0;
    // 收到rst消息 需要立即关闭连接 LISTEN 期间收到的rst应该被忽略
    if (seg.header().rst) {
        // LISTEN == 未收到syn && 未发送syn
        if (!_receiver.ackno().has_value() && _sender.next_seqno_absolute() == 0)
            return;
        // 收到rst信号 不需要进行ack 断开连接即可
        _sender.stream_in().set_error();
        _receiver.stream_out().set_error();
        _active_flag =false;
        //_sender.send_empty_segment();
        //unclean_shutdown();
        return;
    }
    // SYN-SENT && LISTEN 期间 收到的seg没有syn 丢弃
    if (!seg.header().syn&& !_receiver.ackno().has_value())
        return;
    // 如果seg里有ack信息 则将确认信息告知sender
    if (seg.header().ack)
        _sender.ack_received(seg.header().ackno, seg.header().win);
    // 接收的seg交由receiver处理
    _receiver.segment_received(seg);
    // 收到seg我们需要对其进行确认 确认信息由发送的seg捎带 所以要填充窗口
    // 更新了确认信息后 应该马上填充窗口  syn会在这里添加
    _sender.fill_window();
    // 如果要对seg进行ack 如果发送队列里没有数据 我们发送一个空seg来确认
    // 还要注意如果接收的seg本身就是空的 那我们不能对ack进行ack 跳过
    if (seg.length_in_sequence_space() > 0 && _sender.segments_out().empty())
        _sender.send_empty_segment();
    // ack ackno win 会在这里添加
    send_segments();
}

bool TCPConnection::active() const { return _active_flag; }

size_t TCPConnection::write(const string &data) {
    if (!active())
        return 0;
    size_t written_bytes = _sender.stream_in().write(data);
    // 向输出流中写入数据 需要立即填充窗口 并发送
    _sender.fill_window();
    send_segments();
    return written_bytes;
}

//! \param[in] ms_since_last_tick number of milliseconds since the last call to this method
void TCPConnection::tick(const size_t ms_since_last_tick) { 
    if (!active())
        return;
    // 定时器记录时间流逝
    _time_since_last_segment_received += ms_since_last_tick;
    // 告知sender时间
    _sender.tick(ms_since_last_tick);
    // 如果超时重传次数过多 立即断开连接
    if (_sender.consecutive_retransmissions() > _cfg.MAX_RETX_ATTEMPTS) {
        // 此时主动断开连接 需要发送rst信号
        _sender.send_empty_segment();
        unclean_shutdown();
    }
    else // 连接没有断开则发送数据
        send_segments();
}

void TCPConnection::end_input_stream() {
    // 输出流结束（总之更新数据流时都需要立即填充窗口 并发送）
    _sender.stream_in().end_input();
    _sender.fill_window();
    send_segments();
}

void TCPConnection::connect() {
    // 第一次fill 如果输出流中有数据 就会发送syn
    _sender.fill_window();
    send_segments();
}

TCPConnection::~TCPConnection() {
    try {
        if (active()) {
            cerr << "Warning: Unclean shutdown of TCPConnection\n";
            // Your code here: need to send a RST segment to the peer
            // 执行析构函数时 需要告知peer断开连接 发送rst
            _sender.send_empty_segment();
            unclean_shutdown();
        }
    } catch (const exception &e) {
        std::cerr << "Exception destructing TCP FSM: " << e.what() << std::endl;
    }
}

// 该函数将_sender.segment_out()中seg全部发送出去
void TCPConnection::send_segments() {
    TCPSegment seg;
    while (!_sender.segments_out().empty()) {
        seg = _sender.segments_out().front();
        _sender.segments_out().pop();
        // 如果需要进行确认 捎带确认号
        if (_receiver.ackno().has_value()) {
            seg.header().ack = true;
            seg.header().ackno = _receiver.ackno().value();
            seg.header().win = _receiver.window_size();  // ??
        }
        _segments_out.push(seg);
    }
    // 尝试进行clean_shutdown
    try_clean_shutdown();
}

// 尝试进行clean_shutdown
void TCPConnection::try_clean_shutdown() {
    // _linger_after_streams_finish 含义是 
    // 本地的TCPConnection 是否需要在连接结束后等待一段时间 来保证没有任何需要重传的数据

    // 如果本地 已经完全接收 远端peer 发送的信息
    // 而本地还有信息没有发送完
    // 那么 本地 应看作是 server 的角色
    // 当本地 发送完数据 并收到 所有的确认
    // 就可以立即断开连接 而不需要等待远端peer是否有数据需要重传
    if (_receiver.stream_out().input_ended() && !(_sender.stream_in().eof())) {
        // 这种情况下 不需要重传 标记为false
        _linger_after_streams_finish = false;
    }
    
    // 如果 本地 已经完全接收了远端peer的数据 并且 本地的数据完全发送 并且 完全被对面接收并确认
    // 这种情况下 
    // 1. 本地无需等待（server） 可以直接断开连接
    // 2. 本地需要等待（client） 本地的数据先发送完成 而远端peer的数据发送完成后 
    //                         本地返回ack 需要等待一段时间以确认 远端peer收到ack
    //                         在等待的时间内如果收到了远端peer的重传 说明 远端peer没收到确认
    //                         本地需要重新确认 再尝试等待一段时间来关闭连接
    //                         如果本地等待时间超过设定值 说明远端peer已经收到 ack并关闭 那么本地也立即关闭 
    if (_receiver.stream_out().input_ended() && _sender.stream_in().eof() && _sender.bytes_in_flight() == 0)
        if (!_linger_after_streams_finish || time_since_last_segment_received() >= 10 * _cfg.rt_timeout)
            _active_flag = false;
}

// 该函数将输入输出流设为错误 关闭连接 并向peer发送rst信号
void TCPConnection::unclean_shutdown() {
    // unclean_shutdown 需要发送队列中有seg以rst捎带
    // 关闭输入输出流
    _sender.stream_in().set_error();
    _receiver.stream_out().set_error();
    // 关闭active状态
    _active_flag = false;
    // 构建 rst 消息 将其附带在发送队列中的第一个seg上
    TCPSegment seg = _sender.segments_out().front();
    _sender.segments_out().pop();
    seg.header().rst = true;
    seg.header().ack = true;
    if (_receiver.ackno().has_value())
        seg.header().ackno = _receiver.ackno().value();
    seg.header().win = _receiver.window_size();
    _segments_out.push(seg);
}
```

-----

## lab4测试结果
PS：某些测试可能因为网络原因导致Timeout
![](/post_images/posts/Networking/CS144LabAssignmentlab-4/lab4测试结果.jpg "lab4测试结果")  
性能测试结果：（虽然没有很高速度，但好歹超过了0.1G/s的底线哈哈）  
![](/post_images/posts/Networking/CS144LabAssignmentlab-4/性能测试结果.jpg "性能测试结果")  

-----

## webget
lab0中我们用系统的socket写了一个`get_URL()`，现在用我们自己的实现来试试。  
代码：  
```cpp
// #include "socket.hh"
#include "tcp_sponge_socket.hh"
#include "util.hh"

#include <cstdlib>
#include <iostream>

using namespace std;

void get_URL(const string &host, const string &path) {
    //TCPSocket sock{};
    CS144TCPSocket sock{};
    sock.connect(Address(host, "http"));
    sock.write("GET " + path + " HTTP/1.1\r\nHost: " + host + "\r\nConnection:close" + "\r\n\r\n");
    sock.shutdown(SHUT_WR);
    while (!sock.eof()) {
        cout << sock.read();
    }
    //sock.close();
    sock.wait_until_closed();
    return;
}
```

测试结果：  
![](/post_images/posts/Networking/CS144LabAssignmentlab-4/webget测试结果.jpg "webget测试结果")  

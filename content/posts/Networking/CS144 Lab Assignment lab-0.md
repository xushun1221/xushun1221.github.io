---
title: CS144 Lab Assignment lab-0
date: 2022-02-27
tags: [network, C++]
categories: [Networking]
---

## 前言
最近重新学习了一下计算机网络的知识，想着搞点东西做一做，然后就看到了这个Stanford的CS144网络课，它有一系列的网络实验，主要是自己动手实现一个TCP协议。开搞。  
[CS144 startercode](https://github.com/cs144/sponge)  
[CS144课程主页](https://cs144.github.io/)  
*该课程在Linux系统下进行，相关问题可以看本博客的其他博文*

## lab0
## Fetch a Web page
这个部分是让你手动获取一个web页面，通过HTTP请求报文的方式。  
使用`telnet`程序和服务器的某个服务建立一个可靠的字节流（这里是HTTP服务）：  
```shell
telnet cs144.keithw.org http
```
建立连接之后就要手动输入HTTP的请求报文。*注意要快点，不然连接超时了*  
告诉服务器要请求的URL的path和host，并且告知服务器响应之后就关闭连接，然后就会收到响应报文。  
![](/post_images/posts/Networking/CS144 Lab Assignment lab-0/手动getURL.jpg "手动getURL")  

-----

## Send yourself an email
这个部分是让你手动发送一封邮件，请求邮件服务器的SMTP服务，发送邮件。  
这里我用自己的qq邮箱给outlook邮箱发个邮件，用`telnet`程序请求`smtp.qq.com`的smtp服务，登录名和密码需要用Base64编码，密码这里要用qq邮箱的授权码。  
![](/post_images/posts/Networking/CS144 Lab Assignment lab-0/手动smtp发邮件.jpg "手动smtp发邮件")

-----

## Listening and connecting
这里让你用`natcat`程序运行一个服务器进行监听。  
![](/post_images/posts/Networking/CS144 Lab Assignment lab-0/监听和连接.jpg "监听和连接")

-----

## Writing a network program using an OS stream socket：Writing webget
这里是让你用操作系统的stream socket来获取一个网页，和手动的过程一样，只是写成代码。  
由于Internet只能提供尽最大努力交付的数据报服务，因此这些数据报可能会：丢失、乱序、内容更改、重复，所以通常OS会把Internet的这种抽象转为可靠的双向字节流，以便应用层软件使用。OS一般使用socket来完成这种转变并向程序员提供接口，socket和文件描述符类似，一旦建立连接就能进行可靠的通信。  
我们只需要在`../apps/webget.cc`源文件中完成`get_URL()`函数即可。  
注意：HTTP行要以`\r\n`为结尾；要使用`Connection:close`行来告诉服务器不用等待来自客户端的更多请求；在达到`EOF`前输出所有来自服务器的字符；  
```cpp
void get_URL(const string &host, const string &path) {
    TCPSocket sock{};
    sock.connect(Address(host, "http"));
    sock.write("GET " + path + " HTTP/1.1\r\nHost: " + host + "\r\nConnection:close" + "\r\n\r\n");
    sock.shutdown(SHUT_WR);
    while (!sock.eof()) {
        cout << sock.read();
    }
    sock.close();
    return;
}
```
![](/post_images/posts/Networking/CS144 Lab Assignment lab-0/测试webget.jpg "测试webget")  

-----

## An in-memory reliable byte stream
这个部分要求我们在单机上实现一个内存中的可靠字节流。字节可以被输入端写入，被输出端读取，字节流是有限的，写入器可以结束输入，然后不再写入字符，当读取器读到流的末尾时，它会达到`EOF`，不能再读取字节。  
我们需要完成`writer`和`reader`的接口，的定义和实现在`libsponge/byte_stream.hh`和`libsponge/byte_stream.cc`中。  
我们在`ByteStream`类中使用一个`deque<char>`双端队列来实现buffer，分别用三个整型来记录，容量、写字节数、读字节数，用两个布尔量标记是否输入终止和是否出错。  
`write`函数输入参数长度如果比剩余容量大，则丢弃超长的部分。，并记录写入字节数。  
`peek_output`函数要求从buffer的前部读取指定长度的字节，这个功能也是我们使用`deque`的原因，如果指定长度大于buffer中字符长度，则返回buffer长度的字节。
`pop_output`函数和peek逻辑相同，把前端的字节移除，并且记录读取字节数。
`read`函数通过调用`peek_output`和`pop_output`来实现。  
代码：  
```cpp
#include <deque>
using std::deque;

class ByteStream {
  private:
    deque<char> _buffer{};
    size_t _capacity{0};
    size_t _written_count{0};
    size_t _read_count{0};
    bool _input_ended{false};
    bool _error{false};
...
};

ByteStream::ByteStream(const size_t capacity):_capacity(capacity) {}

size_t ByteStream::write(const string &data) {
    size_t len = data.length();
    len = len > remaining_capacity() ? remaining_capacity() : len;
    for (size_t i = 0; i < len; ++ i)
        _buffer.push_back(data[i]);
    _written_count += len;
    return len;
}

string ByteStream::peek_output(const size_t len) const {
    size_t length = len > _buffer.size() ? _buffer.size() : len;
    return string().assign(_buffer.begin(), _buffer.begin() + length);
}

void ByteStream::pop_output(const size_t len) {
    size_t length = len > _buffer.size() ? _buffer.size() : len;
    for (size_t i = 0; i < length; ++ i)
        _buffer.pop_front();
    _read_count += length;
}

std::string ByteStream::read(const size_t len) {
    string ret = peek_output(len);
    pop_output(len);
    return ret;
}

void ByteStream::end_input() {_input_ended = true;}

bool ByteStream::input_ended() const { return _input_ended; }

size_t ByteStream::buffer_size() const { return _buffer.size(); }

bool ByteStream::buffer_empty() const { return _buffer.size() == 0; }

bool ByteStream::eof() const { return input_ended() && buffer_empty(); }

size_t ByteStream::bytes_written() const { return _written_count; }

size_t ByteStream::bytes_read() const { return _read_count; }

size_t ByteStream::remaining_capacity() const { return _capacity - _buffer.size(); }
```
![](/post_images/posts/Networking/CS144 Lab Assignment lab-0/测试bytestream.jpg "测试bytestream")  
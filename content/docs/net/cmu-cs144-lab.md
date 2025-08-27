---
title: "CMU CS144 Lab 记录"
weight: 1
date: 2025-08-22
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

# CMU CS144 Lab 记录

温习一下计算机网络.

课程地址: https://cs144.github.io

用的是 Winter 2025 的 Lab.

## Lab Checkpoint 0: networking warmup

### Writing `webget`

本来以为是简单的发送 Get 然后打印, 没想到测试用例竟然带有 `Transfer-Encoding: chunked`. 输出如下:

```
HTTP/1.1 200 OK
Content-Type: text/plain
Date: Fri, 22 Aug 2025 15:43:36 GMT
Connection: close
Transfer-Encoding: chunked

1
7
1
S
1
m
1
...
```

这样肯定是过不了测试样例的, 上网查了下, `Transfer-Encoding: chunked` 需要手动解析, 内容里面奇数行代表数据长度, 偶数行代表数据, 因此需要把数据手动拼接一下.

### An in-memory reliable byte stream

模拟一个字节流, 有一个 Writer 不断写入, 还有一个 Reader 不断读取. 由于里面的接口用了 `std::string_view`, 要求数据需要连续存储. 因此 buffer 直接开两倍大小模拟循环数组. 剩下的就是一些边界条件的判断.

{{< details "运行结果" close >}}
```
cs144$ cmake --build build --target check0
Test project /home/jinbridge/cs144/build
      Start  1: compile with bug-checkers
 1/11 Test  #1: compile with bug-checkers ........   Passed    3.15 sec
      Start  2: t_webget
 2/11 Test  #2: t_webget .........................   Passed    1.04 sec
      Start  3: byte_stream_basics
 3/11 Test  #3: byte_stream_basics ...............   Passed    0.03 sec
      Start  4: byte_stream_capacity
 4/11 Test  #4: byte_stream_capacity .............   Passed    0.03 sec
      Start  5: byte_stream_one_write
 5/11 Test  #5: byte_stream_one_write ............   Passed    0.03 sec
      Start  6: byte_stream_two_writes
 6/11 Test  #6: byte_stream_two_writes ...........   Passed    0.03 sec
      Start  7: byte_stream_many_writes
 7/11 Test  #7: byte_stream_many_writes ..........   Passed    0.09 sec
      Start  8: byte_stream_stress_test
 8/11 Test  #8: byte_stream_stress_test ..........   Passed    0.04 sec
      Start 37: no_skip
 9/11 Test #37: no_skip ..........................   Passed    0.02 sec
      Start 38: compile with optimization
10/11 Test #38: compile with optimization ........   Passed    5.05 sec
      Start 39: byte_stream_speed_test
        ByteStream throughput (pop length 4096):  7.61 Gbit/s
        ByteStream throughput (pop length 128):   7.48 Gbit/s
        ByteStream throughput (pop length 32):    6.30 Gbit/s
11/11 Test #39: byte_stream_speed_test ...........   Passed    0.16 sec

100% tests passed, 0 tests failed out of 11

Total Test time (real) =   9.71 sec
Built target check0
```
{{< /details >}}


## Lab Checkpoint 1: stitching substrings into a byte stream

### Hands-on component: a private network for the class

这一部分是尝试手动发送数据报, 奇怪的是我尝试使用给定的代码 (ip_raw.cc) 但是怎么也抓不到发送的包.

{{< details "ip_raw.cc" close >}}
```cpp
#include "socket.hh"

#include <cstdlib>
#include <iostream>

using namespace std;

// NOLINTBEGIN(*-casting)

class RawSocket : public DatagramSocket
{
public:
  RawSocket() : DatagramSocket( AF_INET, SOCK_RAW, IPPROTO_RAW ) {}
};

auto zeroes( auto n )
{
  return string( n, 0 );
}

void send_internet_datagram( const string& payload )
{
  RawSocket sock;

  string datagram;
  datagram += char( 0b0100'0101 ); // v4, and headerlength 5 words
  datagram += zeroes( 7 );

  datagram += char( 64 );  // TTL
  datagram += char( 1 );   // protocol
  datagram += zeroes( 6 ); // checksum + src address

  datagram += char( 34 );
  datagram += char( 93 );
  datagram += char( 94 );
  datagram += char( 131 );

  datagram += payload;

  sock.sendto( Address { "1" }, datagram );
}

void send_icmp_message( const string& payload )
{
  send_internet_datagram( "\x08" + payload );
}

void program_body()
{
  string payload;
  while ( cin.good() ) {
	getline( cin, payload );
	send_icmp_message( payload + "\n" );
  }
}

int main()
{
  try {
	program_body();
  } catch ( const exception& e ) {
	cerr << e.what() << "\n";
	return EXIT_FAILURE;
  }

  return EXIT_SUCCESS;
}

// NOLINTEND(*-casting)
```
{{< /details >}}

但是我用 ping 命令测试就可以抓到 ICMP 的包. 但是奇怪的是我在 orb 跑的是 `ping 192.168.1.5` 但是却在 host 的 lo 上抓到了 ICMP 的包. 并且 src 与 dst 均为 192.168.1.5. 不知道 orb 做了什么神奇的操作.

### Implementation: putting substrings in sequence

这一部分的任务是进行字节流的重组, 实现 `Reassembler` 类. 由于 TCP 的报文可能是乱序抵达的, 因此需要进行重排然后写进字节流里面.

`Reassembler` 需要实现:
- 缓存发来的数据
- 如果目前缓存里有可以输入到字符流里面的, 就输入.
- 如果有间隔, 那就继续等.

{{< details "运行结果" close >}}
```
cs144$ cmake --build build --target check1
Test project /home/jinbridge/cs144/build
      Start  1: compile with bug-checkers
 1/18 Test  #1: compile with bug-checkers ........   Passed   10.78 sec
      Start  3: byte_stream_basics
 2/18 Test  #3: byte_stream_basics ...............   Passed    0.04 sec
      Start  4: byte_stream_capacity
 3/18 Test  #4: byte_stream_capacity .............   Passed    0.04 sec
      Start  5: byte_stream_one_write
 4/18 Test  #5: byte_stream_one_write ............   Passed    0.03 sec
      Start  6: byte_stream_two_writes
 5/18 Test  #6: byte_stream_two_writes ...........   Passed    0.03 sec
      Start  7: byte_stream_many_writes
 6/18 Test  #7: byte_stream_many_writes ..........   Passed    0.09 sec
      Start  8: byte_stream_stress_test
 7/18 Test  #8: byte_stream_stress_test ..........   Passed    0.04 sec
      Start  9: reassembler_single
 8/18 Test  #9: reassembler_single ...............   Passed    0.04 sec
      Start 10: reassembler_cap
 9/18 Test #10: reassembler_cap ..................   Passed    0.04 sec
      Start 11: reassembler_seq
10/18 Test #11: reassembler_seq ..................   Passed    0.04 sec
      Start 12: reassembler_dup
11/18 Test #12: reassembler_dup ..................   Passed    0.06 sec
      Start 13: reassembler_holes
12/18 Test #13: reassembler_holes ................   Passed    0.04 sec
      Start 14: reassembler_overlapping
13/18 Test #14: reassembler_overlapping ..........   Passed    0.04 sec
      Start 15: reassembler_win
14/18 Test #15: reassembler_win ..................   Passed    1.20 sec
      Start 37: no_skip
15/18 Test #37: no_skip ..........................   Passed    0.02 sec
      Start 38: compile with optimization
16/18 Test #38: compile with optimization ........   Passed    3.82 sec
      Start 39: byte_stream_speed_test
        ByteStream throughput (pop length 4096):  2.60 Gbit/s
        ByteStream throughput (pop length 128):   2.54 Gbit/s
        ByteStream throughput (pop length 32):    2.38 Gbit/s
17/18 Test #39: byte_stream_speed_test ...........   Passed    0.22 sec
      Start 40: reassembler_speed_test
        Reassembler throughput (no overlap):  36.37 Gbit/s
        Reassembler throughput (10x overlap):  6.15 Gbit/s
18/18 Test #40: reassembler_speed_test ...........   Passed    0.10 sec

100% tests passed, 0 tests failed out of 18

Total Test time (real) =  16.70 sec
Built target check1
```
{{< /details >}}

## Lab Checkpoint 2: the TCP receiver

### Translating between 64-bit indexes and 32-bit seqnos

这一部分的任务是在 seqno 与 absolute seqno 之间做转换. 可以这样理解: 

- seqno 本来应该是 64 位的, 但是为了节省空间, 实际只会发送低 32 位.
- seqno 会带一个偏移 (ISN).
- absolute seqno 就是去掉 offset 并还原到 64 位的 seqno.

从 absolute seqno 转换为 seqno 很简单, 直接加上 ISN 取低 32 位即可.

从 seqno 还原出 absolute seqno 就有点麻烦了. 因为一个 seqno 会对应多个 absolute seqno, 因此需要一个 checkpoint 来指定是哪一个 absolute seqno. 转换结果取最接近 checkpoint 的 absolute seqno.

对应的思路是, 先计算出 checkpoint 左边第一个 absolute seqno. 也就是位于 $[\texttt{checkpoint} - 2^{32}, \texttt{checkpoint}]$ 的 absolute seqno. 之后不断向右加 $2^{32}$ 检查是不是更近了. 如果变远了, 那么当前值就是要找的那个 absolute seqno.

### Implementing the TCP receiver

接下来就是实现 TCP receiver. 要实现的功能包括:

- 接收一个 `TCPSenderMessage`, 把里面的数据填到 `Reassembler` 中.
- 根据 `Reassembler` 的状态返回相应的 `TCPReceiverMessage`.

对于 `TCPSenderMessage`, 消息的 `first_index` 可以认为是 absolute seqno - 1. 这里比较麻烦的一点就是 SYN 是占用一个 seqno 的, 因此需要特判一下消息有没有 SYN, 有的话 `first_index` 就需要额外 + 1 来避开.

其他的一些边界条件也需要注意, 比如 window size 最大是 65535.


{{< details "运行结果" close >}}
```
cs144$ cmake --build build --target check2
Test project /home/jinbridge/cs144/build
      Start  1: compile with bug-checkers
 1/30 Test  #1: compile with bug-checkers ........   Passed    5.98 sec
      Start  3: byte_stream_basics
 2/30 Test  #3: byte_stream_basics ...............   Passed    0.04 sec
      Start  4: byte_stream_capacity
 3/30 Test  #4: byte_stream_capacity .............   Passed    0.03 sec
      Start  5: byte_stream_one_write
 4/30 Test  #5: byte_stream_one_write ............   Passed    0.03 sec
      Start  6: byte_stream_two_writes
 5/30 Test  #6: byte_stream_two_writes ...........   Passed    0.03 sec
      Start  7: byte_stream_many_writes
 6/30 Test  #7: byte_stream_many_writes ..........   Passed    0.09 sec
      Start  8: byte_stream_stress_test
 7/30 Test  #8: byte_stream_stress_test ..........   Passed    0.04 sec
      Start  9: reassembler_single
 8/30 Test  #9: reassembler_single ...............   Passed    0.04 sec
      Start 10: reassembler_cap
 9/30 Test #10: reassembler_cap ..................   Passed    0.04 sec
      Start 11: reassembler_seq
10/30 Test #11: reassembler_seq ..................   Passed    0.04 sec
      Start 12: reassembler_dup
11/30 Test #12: reassembler_dup ..................   Passed    0.06 sec
      Start 13: reassembler_holes
12/30 Test #13: reassembler_holes ................   Passed    0.04 sec
      Start 14: reassembler_overlapping
13/30 Test #14: reassembler_overlapping ..........   Passed    0.04 sec
      Start 15: reassembler_win
14/30 Test #15: reassembler_win ..................   Passed    1.17 sec
      Start 16: wrapping_integers_cmp
15/30 Test #16: wrapping_integers_cmp ............   Passed    0.03 sec
      Start 17: wrapping_integers_wrap
16/30 Test #17: wrapping_integers_wrap ...........   Passed    0.02 sec
      Start 18: wrapping_integers_unwrap
17/30 Test #18: wrapping_integers_unwrap .........   Passed    0.02 sec
      Start 19: wrapping_integers_roundtrip
18/30 Test #19: wrapping_integers_roundtrip ......   Passed    0.30 sec
      Start 20: wrapping_integers_extra
19/30 Test #20: wrapping_integers_extra ..........   Passed    0.25 sec
      Start 21: recv_connect
20/30 Test #21: recv_connect .....................   Passed    0.07 sec
      Start 22: recv_transmit
21/30 Test #22: recv_transmit ....................   Passed    0.38 sec
      Start 23: recv_window
22/30 Test #23: recv_window ......................   Passed    0.04 sec
      Start 24: recv_reorder
23/30 Test #24: recv_reorder .....................   Passed    0.04 sec
      Start 25: recv_reorder_more
24/30 Test #25: recv_reorder_more ................   Passed    2.66 sec
      Start 26: recv_close
25/30 Test #26: recv_close .......................   Passed    0.04 sec
      Start 27: recv_special
26/30 Test #27: recv_special .....................   Passed    0.07 sec
      Start 37: no_skip
27/30 Test #37: no_skip ..........................   Passed    0.02 sec
      Start 38: compile with optimization
28/30 Test #38: compile with optimization ........   Passed    2.17 sec
      Start 39: byte_stream_speed_test
        ByteStream throughput (pop length 4096):  2.61 Gbit/s
        ByteStream throughput (pop length 128):   2.57 Gbit/s
        ByteStream throughput (pop length 32):    2.39 Gbit/s
29/30 Test #39: byte_stream_speed_test ...........   Passed    0.22 sec
      Start 40: reassembler_speed_test
        Reassembler throughput (no overlap):  36.29 Gbit/s
        Reassembler throughput (10x overlap):  4.23 Gbit/s
30/30 Test #40: reassembler_speed_test ...........   Passed    0.15 sec

100% tests passed, 0 tests failed out of 30

Total Test time (real) =  14.20 sec
Built target check2
```
{{< /details >}}


## Lab Checkpoint 3: the TCP sender

### Implementing the TCP sender

这一部分的任务是实现 TCP sender. 相对要复杂一点.

TCP sender 核心的函数包括三个:
- `send`: 根据字节流的内容与当前连接的状态尝试发送对应的 segment.
- `receive`: 根据收到的回复更新自己的状态.
- `tick`: 重传超时的 segment.

下面拆解下对应的逻辑:

`send` 的任务是根据字节流的内容与当前连接的状态尝试发送对应的 segment. 那么首先要做的就是判断要不要发 segment. 可以讨论下面几种情况:
- 1. sender 已经把所有的东西都发完了, 包括最后的 FIN. → 不发.
- 2. sender 还没有发完所有的东西, 至少 FIN 还没发
  - 2.1. 上次 receive 确认到的 `window_size` 都发满了 → 不发, 因为对面还没来得及处理
  - 2.2. 上次 receive 确认到的 `window_size` 还没发满
    - 2.2.1. 字节流里面还有东西 → 肯定要发
    - 2.2.2. 字节流里面没东西
      - 2.2.2.1. 还没发 SYN → 那么应该发一个 SYN.
      - 2.2.2.2. 如果 SYN 已经发了 → 那么就没东西可以发, 不发.
    - 2.2.3. 字节流里面的东西发完了 → 补一个 FIN. (根据上面的条件, 此时 FIN 还没发)

假设有这些变量
- `finished_`: 是否已经发送 FIN.
- `send_next_abs_seqno`: sender 下一个要发送的数据的 absolute seqno.
- `recv_next_abs_seqno`: 对面需要的第一个数据的 absolute seqno.
- `push_window_size`: 窗口大小

根据上面的讨论, 可以得出下面的布尔表达式:

```cpp
!finished_ // not case 1
&& send_next_abs_seqno_ < recv_next_abs_seqno_ + push_window_size_ // not case 2.1
&& ( input_.reader().bytes_buffered()  // case 2.2.1
     || ( input_.reader().bytes_buffered() == 0
          && send_next_abs_seqno_ == 0 )  // case 2.2.2.1
     || ( input_.reader().is_finished() ) // case 2.2.3
   )
```

接下来的任务就是构造 msg. 在构造 msg 之前, 需要确认 msg 的最大长度 `max_sequence_length`. 这个值应当是 sender 下一个要发送的 index 到对面能塞进 window 里面的最后一个 index 的长度. 也就是 `recv_next_abs_seqno_ + push_window_size_ - send_next_abs_seqno_`

根据上面的情况 2.1, 可以确认 `max_sequence_length` 一定是大于 0 的.

接下来要判断需不需要带上 SYN. 如果下一个要发送的 index 是 0, 那就说明这是第一个 segment, 因此需要设置 `msg.SYN` 为 true.

之后就是判断 payload 的最大长度 `max_payload_size`. 取两个中的最小值:
- `max_sequence_length - msg.SYN`
- `TCPConfig::MAX_PAYLOAD_SIZE`

在确认完最大长度以后再确认实际要传输的长度 `payload_size`, 取两个中的最小值:
- `input_.reader().bytes_buffered()`
- `max_payload_size`

之后我们要判断需不需要塞 FIN:

- 如果字节流已经关闭并清空, 那我们就需要塞一个 FIN.
  - 但是能不能塞进去还要看长度
  - 如果 `payload_size` 加上 `SYN` 刚好达到 msg 的最大长度
    - 那我们就塞不进去了
  - 不然的话就塞一个 FIN 进去, 并且更新 `finished_` 为 true.

最后给 msg 加上 seqno 就可以发送了, 发送的同时也存到自己的等待队列里.

接下来就是 `receive` 的逻辑, `receive` 的任务是根据收到的回复更新自己的状态. 同样的, 讨论接收到的 msg 的几种情况:
- 1. msg 的 ackno 未知: 这种情况只能是在 sender 发 SYN 之前收到的.
  - 1.1. msg 的 window_size 是 0 → 还没发 SYN 对面就满了, 连 SYN 都没办法发出去, 说明连接出问题了, 设置 RST.
  - 1.2. msg 的 window_size 不是 0 → 更新自己的 window_size.
- 2. msg 的 ackno 对应的 absolute seqno 前面包含还没发出去的东西 → 对面未卜先知, 消息有问题, 不予理睬
- 3. msg 的 ackno 对应的 absolute seqno 已经被接收到了 → 过时的消息, 不予理睬
- 4. msg 的 ackno 对应的 absolute seqno 是我们需要的
  - 更新 `recv_next_abs_seqno_`
  - 重置传输失败计数与 RTO.
  - 根据这个看看等待队列里面有没有确认收到的, 如果有就从队列里面移出去.
  - 重置计时器.

对于计时器的设计, 我是将等待队列设计为 `pair<TCPSenderMessage, uint64_t>` 内容, 前面放 msg, 后面放这个 msg 的 expire time. 每次取队首的 expire time 与当前的时间戳比较就可以确认有没有超时.

最后就是 `tick`, 重传超时的 segment. 这个的逻辑相对比较简单:
- 更新当前的时间戳
- 如果等待队列队首的 msg 超时了, 并且
  - 对面的 window_size 不是 0, 说明丢包了
    - 翻倍 RTO (指数避让) 并增加失败计数
    - 重传 + 重置计时器
  - 对面的 window_size 是 0, 说明可能是没空放这个 msg
    - 只需要重传 + 重置计时器就可以

最麻烦的地方还是一些边界条件的判断.

{{< details "运行结果" close >}}
```
cs144$ cmake --build build --target check3
Test project /home/jinbridge/cs144/build
      Start  1: compile with bug-checkers
 1/37 Test  #1: compile with bug-checkers ........   Passed    2.66 sec
      Start  3: byte_stream_basics
 2/37 Test  #3: byte_stream_basics ...............   Passed    0.06 sec
      Start  4: byte_stream_capacity
 3/37 Test  #4: byte_stream_capacity .............   Passed    0.04 sec
      Start  5: byte_stream_one_write
 4/37 Test  #5: byte_stream_one_write ............   Passed    0.04 sec
      Start  6: byte_stream_two_writes
 5/37 Test  #6: byte_stream_two_writes ...........   Passed    0.04 sec
      Start  7: byte_stream_many_writes
 6/37 Test  #7: byte_stream_many_writes ..........   Passed    0.09 sec
      Start  8: byte_stream_stress_test
 7/37 Test  #8: byte_stream_stress_test ..........   Passed    0.05 sec
      Start  9: reassembler_single
 8/37 Test  #9: reassembler_single ...............   Passed    0.04 sec
      Start 10: reassembler_cap
 9/37 Test #10: reassembler_cap ..................   Passed    0.06 sec
      Start 11: reassembler_seq
10/37 Test #11: reassembler_seq ..................   Passed    0.06 sec
      Start 12: reassembler_dup
11/37 Test #12: reassembler_dup ..................   Passed    0.07 sec
      Start 13: reassembler_holes
12/37 Test #13: reassembler_holes ................   Passed    0.05 sec
      Start 14: reassembler_overlapping
13/37 Test #14: reassembler_overlapping ..........   Passed    0.04 sec
      Start 15: reassembler_win
14/37 Test #15: reassembler_win ..................   Passed    1.14 sec
      Start 16: wrapping_integers_cmp
15/37 Test #16: wrapping_integers_cmp ............   Passed    0.03 sec
      Start 17: wrapping_integers_wrap
16/37 Test #17: wrapping_integers_wrap ...........   Passed    0.02 sec
      Start 18: wrapping_integers_unwrap
17/37 Test #18: wrapping_integers_unwrap .........   Passed    0.03 sec
      Start 19: wrapping_integers_roundtrip
18/37 Test #19: wrapping_integers_roundtrip ......   Passed    0.30 sec
      Start 20: wrapping_integers_extra
19/37 Test #20: wrapping_integers_extra ..........   Passed    0.25 sec
      Start 21: recv_connect
20/37 Test #21: recv_connect .....................   Passed    0.07 sec
      Start 22: recv_transmit
21/37 Test #22: recv_transmit ....................   Passed    0.38 sec
      Start 23: recv_window
22/37 Test #23: recv_window ......................   Passed    0.04 sec
      Start 24: recv_reorder
23/37 Test #24: recv_reorder .....................   Passed    0.05 sec
      Start 25: recv_reorder_more
24/37 Test #25: recv_reorder_more ................   Passed    2.65 sec
      Start 26: recv_close
25/37 Test #26: recv_close .......................   Passed    0.05 sec
      Start 27: recv_special
26/37 Test #27: recv_special .....................   Passed    0.08 sec
      Start 28: send_connect
27/37 Test #28: send_connect .....................   Passed    0.05 sec
      Start 29: send_transmit
28/37 Test #29: send_transmit ....................   Passed    0.62 sec
      Start 30: send_window
29/37 Test #30: send_window ......................   Passed    0.25 sec
      Start 31: send_ack
30/37 Test #31: send_ack .........................   Passed    0.05 sec
      Start 32: send_close
31/37 Test #32: send_close .......................   Passed    0.05 sec
      Start 33: send_retx
32/37 Test #33: send_retx ........................   Passed    0.05 sec
      Start 34: send_extra
33/37 Test #34: send_extra .......................   Passed    0.10 sec
      Start 37: no_skip
34/37 Test #37: no_skip ..........................   Passed    0.02 sec
      Start 38: compile with optimization
35/37 Test #38: compile with optimization ........   Passed    0.74 sec
      Start 39: byte_stream_speed_test
        ByteStream throughput (pop length 4096):  2.59 Gbit/s
        ByteStream throughput (pop length 128):   2.59 Gbit/s
        ByteStream throughput (pop length 32):    2.47 Gbit/s
36/37 Test #39: byte_stream_speed_test ...........   Passed    0.22 sec
      Start 40: reassembler_speed_test
        Reassembler throughput (no overlap):  36.58 Gbit/s
        Reassembler throughput (10x overlap):  4.72 Gbit/s
37/37 Test #40: reassembler_speed_test ...........   Passed    0.14 sec

100% tests passed, 0 tests failed out of 37

Total Test time (real) =  10.73 sec
Built target check3
```
{{< /details >}}

### Hands-on activity

这一部分是试着将自己写的 sender 跟 kernel 的 TCP 协议栈接起来.

one megabyte challenge 的输出:

```
cs144$ dd if=/dev/urandom bs=1000000 count=1 of=/tmp/big.txt

cs144$ ./build/apps/tcp_native -l 0 9090 < /tmp/big.txt
DEBUG: Listening for incoming connection...
DEBUG: New connection from 169.254.144.9:60649.
DEBUG: Inbound stream from 169.254.144.9:60649 finished.
DEBUG: Outbound stream to 169.254.144.9:60649 finished.

cs144$ </dev/null ./build/apps/tcp_ipv4 169.254.144.1 9090 > /tmp/big-received.txt
DEBUG: minnow connecting to 169.254.144.1:9090...
DEBUG: minnow successfully connected to 169.254.144.1:9090.
DEBUG: Outbound stream to 169.254.144.1:9090 finished.
DEBUG: minnow outbound stream to 169.254.144.1:9090 finished (0 seqnos still in flight).
DEBUG: minnow outbound stream to 169.254.144.1:9090 has been fully acknowledged.
DEBUG: minnow inbound stream from 169.254.144.1:9090 finished cleanly.
DEBUG: Inbound stream from 169.254.144.1:9090 finished.
DEBUG: minnow waiting for clean shutdown... DEBUG: minnow TCP connection finished cleanly.
done.

cs144$ sha256sum /tmp/big.txt
98dea439eabc1c64a1d9a072df1a3481e7d8302a7f28cb5bf32822a9edbc71d2  /tmp/big.txt
cs144$ sha256sum /tmp/big-received.txt
98dea439eabc1c64a1d9a072df1a3481e7d8302a7f28cb5bf32822a9edbc71d2  /tmp/big-received.txt
```

## Lab Checkpoint 4: measuring the real world

简单的网络测量.

- 测量点: 广东深圳
- 测量命令: `ping -D -n -i 0.2 www.canterbury.ac.nz | tee data.txt`
- 收集了 1 小时的数据, 大约 18000 次请求.

对于问题 10, 执行的测量命令为 `ping -s 1400 -i 0.002 -c 1000 www.canterbury.ac.nz`, 由于 ping 命令禁止发送 `-i` 小于 0.002 的值, 因此对于 `-i` 参数取了 0.002, 0.004, 0.006, 0.008, 0.010 五个值. 结果没有显著测出吞吐量的限制.

结果:
```
[Q1] delivery_rate = 0.985845  (success=17830, sent=18086)
[Q2] longest success streak = 718
[Q3] longest loss streak = 2
[Q4] Wrote autocorrelation plots: autocorr_success_given_success.png, autocorr_loss_given_loss.png
[Q5] Min RTT = 237.000 ms
[Q6] Max RTT = 525.000 ms

[Q4 extra] Conditional vs Unconditional delivery rate:
Unconditional delivery rate (Q1): 0.985845
k=-10:  P(succ|succ)=0.985859   (1 - P(loss|loss))=0.984314
k= -5:  P(succ|succ)=0.986087   (1 - P(loss|loss))=0.968750
k= -1:  P(succ|succ)=0.985810   (1 - P(loss|loss))=0.988281
k= +0:  P(succ|succ)=1.000000   (1 - P(loss|loss))=0.000000
k= +1:  P(succ|succ)=0.985810   (1 - P(loss|loss))=0.988281
k= +5:  P(succ|succ)=0.986087   (1 - P(loss|loss))=0.968750
k=+10:  P(succ|succ)=0.985915   (1 - P(loss|loss))=0.984375
[Q7] Saved RTT vs Time as rtt_vs_time.png
[Q8] Saved CDF as rtt_cdf.png
[Q9] RTT(N) vs RTT(N+1) correlation coefficient = 0.343
[Q9] Saved scatter plot as rtt_correlation.png
[Q10] Saved throughput graph as throughput.png
```

<div style="display: flex; justify-content: center;">
<div style="width: 49%" align="center">
<img src="/image/net/cmu-cs144-lab/autocorr_success_given_success.png" width="100%">
    <br>
    <div style="display: inline-block; color: #999; padding: 2px;">
    autocorr_success_given_success
    </div>
</div>

<div style="width: 2%" align="center">
</div>


<div style="width: 49%" align="center">
<img src="/image/net/cmu-cs144-lab/autocorr_loss_given_loss.png" width="100%">
    <br>
    <div style="display: inline-block; color: #999; padding: 2px;">
    autocorr_loss_given_loss
    </div>
</div>
</div>



<div style="display: flex; justify-content: center;">
<div style="width: 49%" align="center">
<img src="/image/net/cmu-cs144-lab/rtt_vs_time.png" width="100%">
    <br>
    <div style="display: inline-block; color: #999; padding: 2px;">
    rtt_vs_time
    </div>
</div>

<div style="width: 2%" align="center">
</div>


<div style="width: 49%" align="center">
<img src="/image/net/cmu-cs144-lab/rtt_cdf.png" width="100%">
    <br>
    <div style="display: inline-block; color: #999; padding: 2px;">
    rtt_cdf
    </div>
</div>
</div>


<div style="display: flex; justify-content: center;">
<div style="width: 49%" align="center">
<img src="/image/net/cmu-cs144-lab/rtt_correlation.png" width="100%">
    <br>
    <div style="display: inline-block; color: #999; padding: 2px;">
    rtt_correlation
    </div>
</div>

<div style="width: 2%" align="center">
</div>


<div style="width: 49%" align="center">
<img src="/image/net/cmu-cs144-lab/throughput.png" width="100%">
    <br>
    <div style="display: inline-block; color: #999; padding: 2px;">
    throughput
    </div>
</div>
</div>


## Lab Checkpoint 5: down the stack (the network interface)

### The Address Resolution Protocol

这一部分的任务是实现链路层封装以及 ARP 协议.

链路层的任务主要包括:
- 将网络层报文封装为链路层报文
- 将链路层报文解析为网络层报文
- 处理 ARP 协议

有一个需要注意的就是当 ARP 请求发出去, 但直到超时都没得到响应的时候, 链路层应该重发 ARP 请求**并丢弃发送队列里面这个 ip 的报文**

{{< details "运行结果" close >}}
```
cs144$ cmake --build build --target check5
Test project /home/jinbridge/cs144/build
    Start  1: compile with bug-checkers
1/3 Test  #1: compile with bug-checkers ........   Passed    8.37 sec
    Start 35: net_interface
2/3 Test #35: net_interface ....................   Passed    0.09 sec
    Start 37: no_skip
3/3 Test #37: no_skip ..........................   Passed    0.02 sec

100% tests passed, 0 tests failed out of 3

Total Test time (real) =   8.49 sec
Built target check5
```
{{< /details >}}

## Lab Checkpoint 6: building an IP router

### Implementing the Router

这一部分的任务是实现路由器. 路由器运行在网络层, 因此会查看网络层报文. 根据报文的目的 IP 进行转发.

如果 `dst[0:entry.prefix_length] == entry.prefix[0:entry.prefix_length]`, 那么 `dst` 匹配上了 `entry` 这一项, 匹配长度为 `prefix_length`. 转发时会选择最长的那一项进行转发. 并且给 TTL 减 1. 如果 TTL 减去 1 以后为 0, 那么就丢掉这个包.

这里实现的时候踩了一个坑, debug 输出一直报 bad IPv4 datagram, 后来才发现是修改 TTL 之后没有重新计算 checksum. 修改以后就可以正常跑了.

顺便一提, 在 macOS 上使用 OrbStack 虚拟机进行 debug 的时候, 是无法正常使用 gdb 的. 根据 [OrbStack 的官方文档](https://docs.orbstack.dev/machines/#debugging-with-gdb-lldb), 需要使用 qemu 运行. 并使用 gdb 远程连接.

{{< details "运行结果" close >}}
```
cs144$ cmake --build build --target check6
Test project /home/jinbridge/cs144/build
    Start  1: compile with bug-checkers
1/4 Test  #1: compile with bug-checkers ........   Passed    8.50 sec
    Start 35: net_interface
2/4 Test #35: net_interface ....................   Passed    0.09 sec
    Start 36: router
3/4 Test #36: router ...........................   Passed    0.08 sec
    Start 37: no_skip
4/4 Test #37: no_skip ..........................   Passed    0.02 sec

100% tests passed, 0 tests failed out of 4

Total Test time (real) =   8.71 sec
Built target check6
```
{{< /details >}}

## Lab Checkpoint 7: putting it all together

### Sending a file

实测网络连接速度会有点慢, 在比对 checksum 之前可以先用 `ls -l` 检查一下大小是否一致, 不一致的话大概率还在传输中.

```
cs144$ ./build/apps/endtoend server cs144.keithw.org 62844 < /tmp/big.txt 
DEBUG: Network interface has Ethernet address 02:00:00:3b:18:dd and IP address 172.16.0.1
DEBUG: Network interface has Ethernet address 02:00:00:8f:ee:40 and IP address 10.0.0.172
DEBUG: Network interface has Ethernet address 1a:1f:25:18:9a:8f and IP address 172.16.0.100
DEBUG: minnow listening for incoming connection...
DEBUG: minnow new connection from 192.168.0.50:25704.
DEBUG: minnow inbound stream from 192.168.0.50:25704 finished cleanly.
DEBUG: Inbound stream from 172.16.0.100 finished.
DEBUG: Outbound stream to 172.16.0.100 finished.
DEBUG: minnow waiting for clean shutdown... DEBUG: minnow outbound stream to 192.168.0.50:25704 finished (64000 seqnos still in flight).
DEBUG: minnow outbound stream to 192.168.0.50:25704 has been fully acknowledged.
DEBUG: minnow TCP connection finished cleanly.
done.
Exiting... done.

cs144$ </dev/null ./build/apps/endtoend client cs144.keithw.org 62845 > /tmp/big-received.txt
DEBUG: Network interface has Ethernet address 02:00:00:57:ce:b9 and IP address 192.168.0.1
DEBUG: Network interface has Ethernet address 02:00:00:ad:38:e3 and IP address 10.0.0.192
DEBUG: Network interface has Ethernet address da:1e:79:62:63:71 and IP address 192.168.0.50
DEBUG: Connecting from 192.168.0.50:25704...
DEBUG: minnow connecting to 172.16.0.100:1234...
DEBUG: minnow successfully connected to 172.16.0.100:1234.
DEBUG: Outbound stream to 172.16.0.100 finished.
DEBUG: minnow outbound stream to 172.16.0.100:1234 finished (0 seqnos still in flight).
DEBUG: minnow outbound stream to 172.16.0.100:1234 has been fully acknowledged.
DEBUG: minnow inbound stream from 172.16.0.100:1234 finished cleanly.
DEBUG: Inbound stream from 172.16.0.100 finished.
DEBUG: minnow waiting for clean shutdown... DEBUG: minnow TCP connection finished cleanly.
done.
Exiting... done.

cs144$ sha256sum /tmp/big.txt 
793f4c489e8f2a78f9a8bfbcc4059d7baab65ec846286ea812ef7c003d72884a  /tmp/big.txt
cs144$ sha256sum /tmp/big-received.txt 
793f4c489e8f2a78f9a8bfbcc4059d7baab65ec846286ea812ef7c003d72884a  /tmp/big-received.txt
```

---
title: LA 挑战赛：双 Cache 设计
date: 2025-07-15 13:03:13
tags:
    - cache
    - CPU
categories: 体系结构
---

## 前言
cache 的设计如果能配合良好的 debug 系统，那么就不是太难。由于 icache 只是 dcache 的一个子集功能，因此只要设计好了 dcache，icache 自然就能正常运作。笔者设计好 cache 并成功启动 linux_kernel 后，认为 cache 设计可以问分为两个部分：可缓存的访问和不可缓存的访问，用信号 uncache_en 来指明，这个信号由访存单元和访问 cache 的地址等信号一同给出。

## 总体设计

OpenLA500 用了使用了两个状态机，我认为这个设计过于复杂，虽然 cache write 和 cache read 能并行几个时钟周期并带来性能上的些许优势，但是同时带来了极高的复杂度。对于新手而言，两个状态机带来的思维障碍和收益不相符。因此可以考虑放弃一两个时钟周期的速度来大大简化设计难度。

因此，我采用了一个状态机，读和写都是阻塞的。每一个事务完成后，状态机都会回到 IDLE 状态来准备处理下一个事务。OpenLA500 的做法很激进，它可以一边处理一边接受新的请求，这样缩短一个事务周期，尤其是在连续访问 cache 的时候。但是同样，它会带来新的复杂度，比如考虑强续访存，就得手动保证上一个事务完成在处理下一个。但是我的设计完全不用考这些，它本来就是阻塞的。

## cache_read

命中返回就行，没命中直接替换后 retry。
没命中要处理两件事：
- 替换谁
- 替换后如果脏就写回，不然直接和 AXI 要数据

## cache_write

如果命中，直接写，如果不命中，也是同理。

但是这里要注意数据的拼凑：要对发来的 wdata 根据地址的 offset、wrtrb 等进行拼凑成一个写入单元，再写入 cache。

## uncache_read

如果一个请求是 uncache 类型的，那么问题应该更简单才对，但是做起来好像逻辑更复杂了。

这里的原因就是，得对 axi 模块进行修改了，因为这时候axi 的 size、len 都不一样了，要针对性地处理。另外想要复用已经存在的 cache_axi 数据通路的话，就得针对发射到 axi 的信号进行 mux:

```Verilog
    assign rd_req = request_buffer_uncache_en ? ...
    assign rd_type = request_buffer_uncache_en ? ...
    assign rd_addr = request_buffer_uncache_en ? ...

```

如果 axi 写得没错的话，只要发过去，就能拿到数据，之后将这个数据直接返回就行了。

## uncache_write

这个也是同理：
```Verilog
    assign wr_type  = request_buffer_uncache_en ? ...
    assign wr_addr  = request_buffer_uncache_en ? ...
    assign wr_wstrb = request_buffer_uncache_en ? ...
    assign wr_data  = request_buffer_uncache_en ? ...
```

## 总结
cache 是一个系统，和上下游结合后，它的复杂度将会大大提升，另外 debug 也很费时间，可以说是 30% 的时间在设计代码，60%的时间在排除错误以及 debug，另外的 10% 就是优化各种细节了。





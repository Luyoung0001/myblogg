---
title: 对 SPI、FLASH 的思考
date: 2025-2-9 17:57:27
tags:
    - flash
    - SPI
categories: ysyx
---
<!--more-->

## [](#从-flash-中读出数据（1） "从 flash 中读出数据（1）")从 flash 中读出数据（1）

之前在 `ysyxSoC/perip/spi/rtl/spi_top_apb.v` 中定义宏 `FAST_FLASH`，因此当程序访问 `0x30000000+X` 的时候，就会访问这个模块：

<table><tbody><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br><span class="line">9</span><br><span class="line">10</span><br><span class="line">11</span><br><span class="line">12</span><br><span class="line">13</span><br><span class="line">14</span><br><span class="line">15</span><br><span class="line">16</span><br><span class="line">17</span><br><span class="line">18</span><br></pre></td><td class="code"><pre><code class="hljs Verilog"><span class="hljs-meta">`<span class="hljs-keyword">ifdef</span> FAST_FLASH</span><p><span class="hljs-keyword">wire</span> [<span class="hljs-number">31</span>:<span class="hljs-number">0</span>] data;<br><span class="hljs-keyword">parameter</span> invalid_cmd = <span class="hljs-number">8'h0</span>;<br>flash_cmd flash_cmd_i(<br>  <span class="hljs-variable">.clock</span>(clock),<br>  <span class="hljs-variable">.valid</span>(in_psel &amp;&amp; !in_penable),<br>  <span class="hljs-variable">.cmd</span>(in_pwrite ? invalid_cmd : <span class="hljs-number">8'h03</span>),<br>  <span class="hljs-variable">.addr</span>({<span class="hljs-number">8'b0</span>, in_paddr[<span class="hljs-number">23</span>:<span class="hljs-number">2</span>], <span class="hljs-number">2'b0</span>}),<br>  <span class="hljs-variable">.data</span>(data)<br>);<br><span class="hljs-keyword">assign</span> spi_sck    = <span class="hljs-number">1'b0</span>;<br><span class="hljs-keyword">assign</span> spi_ss     = <span class="hljs-number">8'b0</span>;<br><span class="hljs-keyword">assign</span> spi_mosi   = <span class="hljs-number">1'b1</span>;<br><span class="hljs-keyword">assign</span> spi_irq_out= <span class="hljs-number">1'b0</span>;<br><span class="hljs-keyword">assign</span> in_pslverr = <span class="hljs-number">1'b0</span>;<br><span class="hljs-keyword">assign</span> in_pready  = in_penable &amp;&amp; in_psel &amp;&amp; !in_pwrite;<br><span class="hljs-keyword">assign</span> in_prdata  = data[<span class="hljs-number">31</span>:<span class="hljs-number">0</span>];</p></code></pre></td></tr></tbody></table>

接着就是：

<table><tbody><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br><span class="line">9</span><br><span class="line">10</span><br><span class="line">11</span><br><span class="line">12</span><br><span class="line">13</span><br><span class="line">14</span><br><span class="line">15</span><br><span class="line">16</span><br><span class="line">17</span><br><span class="line">18</span><br></pre></td><td class="code"><pre><code class="hljs Verilog"><span class="hljs-meta">`<span class="hljs-keyword">ifdef</span> FAST_FLASH</span><p><span class="hljs-keyword">wire</span> [<span class="hljs-number">31</span>:<span class="hljs-number">0</span>] data;<br><span class="hljs-keyword">parameter</span> invalid_cmd = <span class="hljs-number">8'h0</span>;<br>flash_cmd flash_cmd_i(<br>  <span class="hljs-variable">.clock</span>(clock),<br>  <span class="hljs-variable">.valid</span>(in_psel &amp;&amp; !in_penable),<br>  <span class="hljs-variable">.cmd</span>(in_pwrite ? invalid_cmd : <span class="hljs-number">8'h03</span>),<br>  <span class="hljs-variable">.addr</span>({<span class="hljs-number">8'b0</span>, in_paddr[<span class="hljs-number">23</span>:<span class="hljs-number">2</span>], <span class="hljs-number">2'b0</span>}),<br>  <span class="hljs-variable">.data</span>(data)<br>);<br><span class="hljs-keyword">assign</span> spi_sck    = <span class="hljs-number">1'b0</span>;<br><span class="hljs-keyword">assign</span> spi_ss     = <span class="hljs-number">8'b0</span>;<br><span class="hljs-keyword">assign</span> spi_mosi   = <span class="hljs-number">1'b1</span>;<br><span class="hljs-keyword">assign</span> spi_irq_out= <span class="hljs-number">1'b0</span>;<br><span class="hljs-keyword">assign</span> in_pslverr = <span class="hljs-number">1'b0</span>;<br><span class="hljs-keyword">assign</span> in_pready  = in_penable &amp;&amp; in_psel &amp;&amp; !in_pwrite;<br><span class="hljs-keyword">assign</span> in_prdata  = data[<span class="hljs-number">31</span>:<span class="hljs-number">0</span>];</p></code></pre></td></tr></tbody></table>

这样，其实就直接从用户自定义的 DPI-C 返回了，因此这很FAST，因为中间没有经过协议。

## [](#从-flash-中读出数据（2） "从 flash 中读出数据（2）")从 flash 中读出数据（2）

现实中，flash 并不是上面这样工作的，上面只是一个 flash 工作的非常粗略的行为级别模拟。

事实上，flash 的访问是需要控制器的，这里的例子是 spi-top。

当程序访问 flash 的时候（`0x30000000+X`），依然会经过 xbar 路由到 `ysyxSoC/perip/spi/rtl/spi_top_apb.v` 模块中的 APB 端口。

当注释掉 FAST\_FLASH 时，模块 spi\_top 将会起作用：

<table><tbody><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br><span class="line">9</span><br><span class="line">10</span><br><span class="line">11</span><br><span class="line">12</span><br><span class="line">13</span><br><span class="line">14</span><br><span class="line">15</span><br><span class="line">16</span><br><span class="line">17</span><br><span class="line">18</span><br><span class="line">19</span><br></pre></td><td class="code"><pre><code class="hljs Verilog">spi_top u0_spi_top (<br>  <span class="hljs-variable">.wb_clk_i</span>(clock),<br>  <span class="hljs-variable">.wb_rst_i</span>(reset),<br>  <span class="hljs-variable">.wb_adr_i</span>(in_paddr[<span class="hljs-number">4</span>:<span class="hljs-number">0</span>]),<br>  <span class="hljs-variable">.wb_dat_i</span>(in_pwdata),<br>  <span class="hljs-variable">.wb_dat_o</span>(in_prdata),<br>  <span class="hljs-variable">.wb_sel_i</span>(in_pstrb),<br>  <span class="hljs-variable">.wb_we_i</span> (in_pwrite),<br>  <span class="hljs-variable">.wb_stb_i</span>(in_psel),<br>  <span class="hljs-variable">.wb_cyc_i</span>(in_penable),<br>  <span class="hljs-variable">.wb_ack_o</span>(in_pready),<br>  <span class="hljs-variable">.wb_err_o</span>(in_pslverr),<br>  <span class="hljs-variable">.wb_int_o</span>(spi_irq_out),<p>  <span class="hljs-variable">.ss_pad_o</span>(spi_ss),<br>  <span class="hljs-variable">.sclk_pad_o</span>(spi_sck),<br>  <span class="hljs-variable">.mosi_pad_o</span>(spi_mosi),<br>  <span class="hljs-variable">.miso_pad_i</span>(spi_miso)<br>);</p></code></pre></td></tr></tbody></table>

也就是说，这时候访问的其实是 spi\_top，spi\_top 会和 spi\_shift 结合，然后spi\_shift 和 flash 这类 slave 设备连接。

<table><tbody><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br></pre></td><td class="code"><pre><code class="hljs Scala"><span class="hljs-keyword">val</span> flash = <span class="hljs-type">Module</span>(<span class="hljs-keyword">new</span> flash)<br>flash.io &lt;&gt; masic.spi<br>flash.io.ss := masic.spi.ss(<span class="hljs-number">0</span>)<br></code></pre></td></tr></tbody></table>

之后，就会发生数据交换，拿到数据后，再返回给程序。

因此这个过程非常慢，在后面的 XIP 下运行 cpu-tests，就会发现运行得非常非常慢。

## [](#运行-char-test（1） "运行 char_test（1）")运行 char\_test（1）

刚开始，把 char\_test 的 bin 文件装在 sram 中，这最简单，直接就能运行。整个架构是，将 bin 放在 sram 中后，直接让 NPC 的 PC 指向 sram 的起始地址，就可以跑了。这个过程不涉及任何 mrom、flash。

## [](#运行-char-test（2） "运行 char_test（2）")运行 char\_test（2）

接着，就要把 char\_test 的 bin 文件放在 mrom 中了，这里面就要做很多事情了，比如要修改链接脚本，要把 `.data section` 通过类似的 bootloader 的方式搬到 sram。然后 NPC 的 PC 指向 mrom 的起始地址开始执行。这一切目前还好。

## [](#运行-char-test（3） "运行 char_test（3）")运行 char\_test（3）

现在情况复杂起来了，我们有了 flash，还是带有 spi 的真正意义上的 flash。

要访问 flash，首先就得有一个程序 A，它能够通过特定的步骤来读取 flash，这使得我们可以通过 A 来将装载到flash 中的 char\_test 的 bin 文件搬到 sram 中。而且 A 还可以直接跳转到 sram 中执行 char\_test。

首先，我们把 A 程序装载 mrom 中，NPC 的 PC 指向 mrom 的起始地址执行 A，然后 A 就会开始搬运工作，搬运工作完成后就跳转到 sram 执行 char\_test。

可以看到，A 的行为有点像什么？操作系统？A 可以通过一系列步骤访问 flash，准确地来说是 spi-master。然后 A 还可以将访问到的数据加载到 sram，接着再跳转到 sram 执行目标程序。

那么这样就很麻烦。

## [](#运行-char-test（4） "运行 char_test（4）")运行 char\_test（4）

我们的目标是什么？是在大量便宜的存储单元中运行程序，因此我们需要便宜的可编程的 flash，而不是昂贵每次都要重新光刻的 mrom，以及少量的 sram（为了提高性能，事实上，如果不嫌弃慢的话，sram 都不需要）。

因此，我们的想法是，为什么不直接在 flash 中执行程序呢？

实际上，在装载程序本身的地方直接执行程序叫做`就地执行`(XIP, eXecute In Place)方式。

但是我们必须保留程序 A 的一部分功能，它必须能通过 spi-master 来访问flash，而且我们必须把 A 放在 mrom 中，NPC 可以访问 mrom 中的 A 程序来获取 flash 中的第一条指令，接着访问 mrom 中的 A 程序来获取 flash 中的第二条指令……

可以看到，这很麻烦，每次在 flash 中执行目标程序 char\_test，都要访问辅助程序 A，效率相当底下。

## [](#运行-char-test（5） "运行 char_test（5）")运行 char\_test（5）

因此，我们想到的是，直接将 程序A 用物理器件代替，直接让它成为 spi\_top\_apb 的一部分，这样，CPU 的 IFU 访问 flash 时，直接就能取到指令，就像访问sram 和 mrom 一样。

这就是最终的目标。
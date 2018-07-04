# Lab04 实验报告

## Meltdown漏洞背景

​	熔断（Meltdown）漏洞是一个存在于众多Intel x86/x86-64微处理器、部分IBM POWER架构处理器以及部分ARM架构处理器的硬件设计缺陷，这种缺陷使得低权限的进程无论是否被授权均可以访问具备高权限保护的内存空间。2018年1月Meltdown漏洞连同另一个业绩与分支预测以及缓存技术的、术语重量级信息安全漏洞的硬件缺陷“Spectre”被公布，正式名称为“Rouge Data Cache Load”（恶意数据缓存加载）。

​	由于Intel处理器以及IBM POWER处理器在市场上占据了极大的份额，Meltdown的影响波及到了整个x86、POWER服务器领域，几乎整个小型主机及大型主机市场、个人计算机市场等。更重要的是，Meltdown的危险程度非常高，发现者选择先与处理器厂商及核心客户协商备妥修补方案再另行公布该缺陷，否则会引发全球性的信息安全灾难。

​	Meltdown硬件缺陷目前虽然可以通过软件修复，但是会导致处理器性能的显著下降，而从根本上修复该缺陷的方法唯有重新设计处理器的架构，导致Intel、IBM、ARM都将新处理器微架构的推出日程大幅延后。

## Meltdown原理

​	Meltdown的攻击利用了CPU的“乱序执行”（Out-of-order execution）技术以及缓存侧信道攻击（Cache side-channel attack）的手段。

#### 乱序执行

​	现代CPU具有乱序执行以及预测执行的功能，当CPU无法确定下一条指令是否一定需要执行时，会进行分支预测（记录过去程序跳转的结果并用它来推测下一条可能被执行的指令），并根据预测的结果进行乱序执行。如果预测正确，执行指令的结果可以立即使用；如果不正确，CPU通过reoder buffer回滚操作结果。![图1](/Users/Flint/Desktop/EasyAiMegumi/2018spring/OSH/lab04/QQ20180704-161632@2x.png)

​	如上图所示，Intel的CPU体系结构中，流水线由前端（Frontend）、执行引擎（Execution Engine）和内存子系统（Memory System）组成。前端模块将x86指令从存储器中读取出来并解码成微操作（micro operations, $\mu$OP），$\mu$OP接着被发送给执行引擎，在这里实现了乱序执行。如上图：执行引擎中的重新排序缓冲区(reorder buffer）负责寄存器分配、寄存器重命名和将结果提交到软件可见的寄存器等等。当分支预测不正确时，重新排序缓冲区会被清除以及保留站会被重新初始化。

#### 缓存攻击

​	而缓存侧信道攻击是一种利用缓存导致的时间差异而进行攻击方法。当我们访问Memory中的数据时，已经存入cache的数据访问起来会特别快，而没有被存入cache的数据访问比较慢，利用两者之间的时间差，可以偷取数据（具体见后文）。在多种缓存攻击技术中，Meltdown的原始论文着重讨论了Flush+Reload方法。采取Flush+Reload方法的攻击者经常使用`cflush`指令将目标内存位置的cache刷掉，然后读目标内存的数据并测量目标内存中数据加载所需的时间。通过这个时间信息，攻击者可以获取另一个进程是否已经将数据加载到了缓存中。

#### 一个最简单的例子

​	Meltdown的论文中举了一个最简单的例子来说明Meltdown是如何利用CPU设计缺陷的。

```c
raise_exception();//触发了一个异常，导致控制流不会执行异常之后的代码
//the line below is never reached
access(probe_array[data*4096]);//由于乱序执行，CPU实际上会执行异常指令之后的指令
```

​	该示例代码在理论上不会访问`probal_array`数组，因为异常会立即陷入内核并终止该应用程序，但是由于乱序执行，CPU可能已经执行了异常指令之后的指令。当然，虽然执行了，但是指令并没有造成软件可见的影响，指令的执行结果并没有体现到软件可见的寄存器或者memory里，从软件工程师的角度看不到这些指令的执行，从寄存器和memory上也看不到指令产生的变化。但是从CPU微架构来看，发生了副作用——乱序执行过程中，加载内存值到寄存器同时也会把该值保存到cache中。当CPU回滚，丢弃乱序执行的结果时，cache中存储的内容并没有被丢弃。这时就可以发动微架构侧信道攻击的方法，比如——Flush+Reload，来检测指定的内存地址的数据是否被存到cache了，这样使得内存的信息变得对用户可见。![](/Users/Flint/Desktop/EasyAiMegumi/2018spring/OSH/lab04/QQ20180704-161711@2x.png)

​	回到示例代码，`probe_array`是一个按照4KB字节组织的数组，改变data的值可以按照4KB大小来遍历访问该数组。假如——在乱序执行时访问了`data`变量指定的`probe_array`数组内的某个4K内存块，那么`probe_array`数组对应的页的数据会加载导cache中，接下来只要利用flush+reload扫描数组中每个页面的cache情况就可以反推出`data`的值——因为`data`的值和`probe_array`的页面是一一对应的（Intel处理器中，page size之间的cache状态是完全独立的）。这样`data`的值就被泄漏给了我们。

​	上图通过Flush+Reload方法遍历了probe_array数组中的各个page：横坐标为页号，纵坐标为访问时间，如果数据不在cache中，需要400哥周期，如果cache hit，则访问时间在200周期左右。图中`data=84`时明显cache hit了。说明乱序执行导致本不该执行的指令也影响到了CPU微架构的状态。

#### Meltdown攻击的组成

​	上面这个简单的例子体现了，完整的Meltdown攻击的两个组件：

1. 执行瞬态指令：

   ​	CPU执行一个或多个在正常路径中永远不会执行的指令，称为瞬态指令。如果瞬态指令的执行依赖于一个受保护的数据，比如程序在运行于用户态时访问了特权页面，就会触发一个异常，该异常通常会终止程序。但如果攻击者的目标是一个特权页面中受保护的数据，那么我们必须处理这个异常——比如设置异常处理函数，在发生异常时调用该函数，此时瞬态指令序列就已经被执行，我们的目的就达到了。

2. 构建隐蔽通道（convert channel）：

   ​	我们需要把执行瞬态指令序列后，CPU微结构状态变化的信息转换为相应的体系结构状态。从这些状态中“推导出”被保护的数据。

![](/Users/Flint/Desktop/EasyAiMegumi/2018spring/OSH/lab04/QQ20180704-161721@2x.png)

​	如何构建这样的隐蔽通道呢？前文提到的缓存攻击技术就是一种办法。在隐蔽通道的发送端，瞬态指令序列会访问一个普通内存地址，从而导致地址的数据被加载到了cache（为了加速后续访问）。然后，接收端可以通过测量内存地址的访问时间来监视数据是否已加载到缓存中。因此，发送端可以通过访问内存地址（会加载到cache中）传递bit “1”的信息，或者通过不访问内存地址（不会加载到cache中）来发送bit “0”信息。而接收端可以通过监视cache的信息来接收这个bit “0”或者bit “1”的信息。

## 利用漏洞进行攻击的实现

[注：我参考了SEED Lab的实验教程]: http://www.cis.syr.edu/~wedu/seed/Labs_16.04/System/Meltdown_Attack/

### 实验环境

- Linux ubuntu 4.4.0版本内核     16.04版本系统
- 已安装meltdown补丁，后增加了启动参数nopti
  ![](/Users/Flint/Desktop/EasyAiMegumi/2018spring/OSH/lab04/QQ20180704-163100@2x.png)

### 具体实现

#### step1

​	在实际的Meltdown攻击中，对于想要窃取的内核中受保护的数据，攻击者需要先通过特别的方式取得该数据的地址，位了简便，我们将一段我们已知的数据插入到内核模块中，然后先得到该数据在内存中的地址。这样我们可以只专注于如何攻击的部分。具体代码详见`MeltdownKernel.c`，具体操作如下：

​	

```shell
$ make
$ sudo insmod MeltdownKernel.ko
$ dmesg | grep 'secret data address'
secret data address: 0xffffffffc02da000
```

​	我们存入的数据为一个数组`char secret[8] = {'S','E','E','D','L','a','b','s'}`，在内核中的地址为：`0xffffffffc02ba000`，此地址值不是固定的，而是由上述操作得出的。

### step2

```shell
$ gcc -march=native -o MeltdownAttack MeltdownAttack.c
$ ./MeltdownAttack
```

### 测试结果

运行10次MeltdownAttack程序得到的结果如下：

![](/Users/Flint/Desktop/EasyAiMegumi/2018spring/OSH/lab04/QQ20180704-171548@2x.png)，![QQ20180704-171658@2x](/Users/Flint/Desktop/EasyAiMegumi/2018spring/OSH/lab04/QQ20180704-171658@2x.png)

结果比较符合预期。

## 参考文献

[1]: https://meltdownattack.com/meltdown.pdf	"Meltdown Moritz Lipp, Michael Schwarz, Daniel Gruss, Thomas Prescher, Werner Haas, Stefan Mangard, Paul Kocher, Daniel Genkin, Yuval Yarom, Mike Hamburg"
[2]: http://www.cis.syr.edu/~wedu/seed/Labs_16.04/System/Meltdown_Attack/	"Meltdown Attack Lab Copyright © 2018 Wenliang Du, Syracuse University."


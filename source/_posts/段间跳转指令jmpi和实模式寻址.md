layout: linux
title: 段间跳转指令jmpi和实模式寻址（转）
date: 2016-02-25 15:53:52
tags: Linux 汇编
---

#### 0x00 前言
jmpi是段间跳转指令，用于x86实模式下，
如：BOOTSEG = 0x0c70
    jmpi    4, #BOOTSEG
假如当前段CS==00h，那么执行此指令后将跳转到段CS==0x0c70，当然段cs的值也变为0x0c70，接下来将执行指令0x0c70:0004处的指令。



 实模式下寻址是为了兼容8086处理器，8086是16位CPU(是ALU的数据宽度)，20位地址总线可寻址1M内存空间。其寻址方式：段基址+偏移  的方式，段基址保存在CS、DS、ES等段寄存器内，相当于寻址的高16位，而偏移是内部16位总线提供，在送往外部地址总线时，段基址和偏移合成20位地址，来寻址1M的物理地址空间。

 合成方式：段基址左移4位，然后加上偏移地址。但还不是一般的相加，由于相加前段基址已经左移4位，变成20位了(最低4位是0)，而偏移还是16位，所以，其实是段基址和偏移的高12位相加，偏移的低4位不变。

       如：段基址左移4位后：            0x 8880:0
       偏移地址(0x0440)：     +  0x  044  0
      外部总线20位地址：           0x88c4  0

可看出，这个所谓段式内存管理，并不是纯粹的基址加偏移的方式，据说这是Intel当时欺骗了大家。以下是，我看到的一篇文章中的说法：



#### 0x01 8086/8088的寻址问题

　　8088和80286都是16位CPU，Intel当初为什么会警告IBM和盖茨呢？到底发生了什么？

　　要了解发生了什么，我们要看看处理器的内部，会看到巨大的差异。首先，你找一片８０８８ＣＰＵ，把包装磨掉，磨到ＣＰＵ硅片，放到显微镜下，你会看到8086/88的内部结构，它根本不是一个新的设计，而是两个并联运行的8085（８位）微处理器再多那么一点点。

　　每个8085有它自己的8位数据和16位寻址能力。结合2个8位数据寄存器假装16位寄存器很容易。事实上这没有任何新东西，RCA COSMAC微处理器就使用16个8位寄存器，可作为内部的8位或16位寄存器使用,你可以有多达16个8位寄存器或8个16位寄存器或两者的任何组合。现在，一个中国的普通ＩＣ厂都可以轻易设计的出来。

　　可能由于受当时生产工艺所限，８０８８只能有４０个脚，ｉｎｔｅｌ的设计“精英”左思右想，确定了２０条地址线（１Ｍ的寻址空间），而且１６条数据线还要和２０条地址线中的１６条复用（分时复用，即一会是地址线，一会是数据线，对此要想了解，可看８０８８芯片手册的时序部分，也可看８０５２单片机书籍，它的地址线和数据线也是复用的）。

　　到了问题的实质了，８０８８内的两个８０８５各有一套１６位寻址寄存器，如何让他们寻址２０位的１Ｍ地址呢？其实把他们并在一起形成３２位寻址很简单，如果是那样后来的很多麻烦可能就都没有了（如Ａ２０门），但当时那些“精英”可能认为３２位寻址（４Ｇ地址空间）那是扯淡，估计地球消失了也用不到那么多的内存吧？再说了老板逼的又紧，于是他们采用了在一个硬件上使用两个８０８５非常好实现的方法－－分段：
　　他们把１０２４Ｋ地址空间分成１６字节的段，共６４Ｋ个段，用一个８０８５的１６位寻址寄存器作地址偏移寄存器（故段的长度是６４Ｋ），而另一个８０８５的１６位寻址寄存器作１６字节段的段地址寄存器，注意，他保存的不是１６字节段的地址，而是１６字节段的序号（０，１，．．．６５５３５）。

这样做的好处是：只要在两８０８５ＣＰＵ之间加一个移位器和一个２０位的加法器，就可以完成２０位的地址寻址－－一个８０８５的地址寄存器（段地址－－就是１６字节段的序号）左移４位（＊１６　＝　１６字节小段的首地址），加上另一个８０８５的地址寄存器就可以啦，哈哈！可以向老板交差了，制作成本低，设计速度快，有钱不抢是孙子！至于以后，。。。。

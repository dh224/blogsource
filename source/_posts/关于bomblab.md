---
title: 关于bomblab
date: 2021-08-03 12:00:07
tags: csapp
thumbnail: http://satt.oss-cn-hangzhou.aliyuncs.com/img/aa.png
---
今天看了csapp的第三章，打算做做bomblab。之前在课上，教授其实或多或少都讲过关于bomblab的介绍，我在之前也看过别人的弹幕，据说很难。因此也做好了心理准备。但是没想到这么难。

## 万事开头难

在[csapp网站](http://csapp.cs.cmu.edu/3e/labs.html)上下载到了提供自学的lab，然后放到我的wsl里。显然，我已经忘记了tar命令的操作了，一番查找。输入:

```bash
tar -xvf bomb.tar
bomb/
bomb/bomb
bomb/bomb.c
bomb/README
cd bomb/
```

终于进入了实验室,得到了bomb文件夹下的三个文件。*hooray!* 

打开README.

> This is an x86-64 bomb for self-study students.

在这里我得不到什么帮助，只能打开bomb.c文件，试图找到一点头绪。但是，里面似乎只是一长段奇怪的代码，以及莫名其妙的DR.EVIL的故事。

```c#
/***********************************************************
 * Dr. Evil's Insidious Bomb, Version 1.1
 * Copyright 2011, Dr. Evil Incorporated. All rights reserved.
 *
 * LICENSE:
 *
 * Dr. Evil Incorporated (the PERPETRATOR) hereby grants you (the
 * VICTIM) explicit permission to use this bomb (the BOMB).  This is a
 * time limited license, which expires on the death of the VICTIM.
 * The PERPETRATOR takes no responsibility for damage, frustration,
 * insanity, bug-eyes, carpal-tunnel syndrome, loss of sleep, or other
 * harm to the VICTIM.  Unless the PERPETRATOR wants to take credit,
 * that is.  The VICTIM may not distribute this bomb source code to
 * any enemies of the PERPETRATOR.  No VICTIM may debug,
 * reverse-engineer, run "strings" on, decompile, decrypt, or use any
 * other technique to gain knowledge of and defuse the BOMB.  BOMB
 * proof clothing may not be worn when handling this program.  The
 * PERPETRATOR will not apologize for the PERPETRATOR's poor sense of
 * humor.  This license is null and void where the BOMB is prohibited
 * by law.
 ***********************************************************/
```

最后只能百度，一番查找过后，才知道原来还有[Writeup](http://csapp.cs.cmu.edu/3e/bomblab.pdf)。在里面，总算得到一点提示。

> You can also run it under a debugger, watch what it does step by step, and use this information to defuse it. This is probably the fastest way of defusing it.

里面提到了gdb。自然，我去查找了一番gdb的使用方法，至此。终于可以开始解决bomblab了。

## bomblab的解决过程

当然，像我这样的菜鸡，并不具备独自解决问题的能力。只能参考已有的博客。根据博客上的说法，我只需要用gdb的disas命令，反汇编第一阶段的代码就能得到线索，我当然遵从。*其实看了眼bomb.c的源代码，里面有很明显的提示，但是我一开始没有意识到.*

```c
    /* Hmm...  Six phases must be more secure than one phase! */
    input = read_line();             /* Get input                   */
    phase_1(input);                  /* Run the phase               */
    phase_defused();                 /* Drat!  They figured it out!
				      * Let me know how they did it. */
    printf("Phase 1 defused. How about the next one?\n");
```

因此，为了解决问题，首先就是要解决phase_1()。

### phase_1()

在windows terminal中用gdb打开bomb，然后用disas命令反汇编。

```bash
gdb bomb //打开bomb
--------------
(gdb) disas phase_1
```
得到如下的代码:
```assembly
Dump of assembler code for function phase_1:
   0x0000000000400ee0 <+0>:     sub    $0x8,%rsp
   0x0000000000400ee4 <+4>:     mov    $0x402400,%esi
   0x0000000000400ee9 <+9>:     call   0x401338 <strings_not_equal>
   0x0000000000400eee <+14>:    test   %eax,%eax
   0x0000000000400ef0 <+16>:    je     0x400ef7 <phase_1+23>
   0x0000000000400ef2 <+18>:    call   0x40143a <explode_bomb>
   0x0000000000400ef7 <+23>:    add    $0x8,%rsp
   0x0000000000400efb <+27>:    ret
End of assembler dump.
```
让我们来分析分析。首先是第0行，我们知道%rsp是保存指向栈顶的指针的寄存器，而对他进行自减，就是为了给栈帧开辟0x8个字节的空间。第1行，将立即数0x402400保存到%esi中，然后调用了一个 `<strings_not_equal>`的方法 。在第3行，test了%eax的内容，如果相等,则跳转到第6行，即出栈，返回，函数结束;如果不相等，则跳转到 `<explode_bomb>` ，显然就是炸弹爆炸了。我们知道，%eax是32位系统中的返回值，容易料想到是保存了`<strings_not_equal>`的返回值。自此，分析结束。
%esi中保存的是用于比较的参数，很可能就是要输入的字串.也就是立即数0x402400是字符串的首地址。于是用gdb命令查看.

```bash
(gdb) x 0x402400
0x402400:       0x64726f42
```
结果比较诡异，应该是输出了字符串首地址的指针地址。显然，我还是不懂gdb。一番查找，得到正确的x命令参数.
```bash
(gdb) x/s 0x402400
0x402400:       "Border relations with Canada have never been better."
```
得到了phase_1()的答案。也就是:
>Border relations with Canada have never been better.

### phase_2()

接下去是phase_2()。有了之前打下的基础，phase_2()就不会那么没有头绪了。首先，依旧是使用命令`disas phase_2`。得到:

```assembly
   0x0000000000400efc <+0>:     push   %rbp
   0x0000000000400efd <+1>:     push   %rbx
   0x0000000000400efe <+2>:     sub    $0x28,%rsp
   0x0000000000400f02 <+6>:     mov    %rsp,%rsi
   0x0000000000400f05 <+9>:     call   0x40145c <read_six_numbers>
   0x0000000000400f0a <+14>:    cmpl   $0x1,(%rsp)
   0x0000000000400f0e <+18>:    je     0x400f30 <phase_2+52>
   0x0000000000400f10 <+20>:    call   0x40143a <explode_bomb>
   0x0000000000400f15 <+25>:    jmp    0x400f30 <phase_2+52>
   0x0000000000400f17 <+27>:    mov    -0x4(%rbx),%eax
   0x0000000000400f1a <+30>:    add    %eax,%eax
   0x0000000000400f1c <+32>:    cmp    %eax,(%rbx)
   0x0000000000400f1e <+34>:    je     0x400f25 <phase_2+41>
   0x0000000000400f20 <+36>:    call   0x40143a <explode_bomb>
   0x0000000000400f25 <+41>:    add    $0x4,%rbx
   0x0000000000400f29 <+45>:    cmp    %rbp,%rbx
   0x0000000000400f2c <+48>:    jne    0x400f17 <phase_2+27>
   0x0000000000400f2e <+50>:    jmp    0x400f3c <phase_2+64>
   0x0000000000400f30 <+52>:    lea    0x4(%rsp),%rbx
   0x0000000000400f35 <+57>:    lea    0x18(%rsp),%rbp
   0x0000000000400f3a <+62>:    jmp    0x400f17 <phase_2+27>
   0x0000000000400f3c <+64>:    add    $0x28,%rsp
   0x0000000000400f40 <+68>:    pop    %rbx
   0x0000000000400f41 <+69>:    pop    %rbp
   0x0000000000400f42 <+70>:    ret
```

这段显然比phase_1要长一些，但是分析的流程不会变.
首先是 `push   %rbp` 和 `push   %rbx`。这两个命令我不是很懂，大约就是压入了两个寄存器。后来一番查找，得到结论：
> push   %esi       //将程序的入口地址压入栈中
   push   %ebx        //将被调用者保存寄存器中的值压入栈中，以便在返回前可以恢复它们

总之，这两句对我们分析没有什么关联，继续看。`sub    $0x28,%rsp`栈指针分配了0x28即40<sub>d</sub>。然后就是将栈指针赋给%rsi。查书p120。可知%rsi用于函数的第二个参数.随后调用了`<read_six_numbers>`方法。让我们来看看这个方法。和phase_2类似，输入:`disas read_six_numbers`，得到如下代码：

```assembly
   0x000000000040145c <+0>:     sub    $0x18,%rsp
   0x0000000000401460 <+4>:     mov    %rsi,%rdx
   0x0000000000401463 <+7>:     lea    0x4(%rsi),%rcx
   0x0000000000401467 <+11>:    lea    0x14(%rsi),%rax
   0x000000000040146b <+15>:    mov    %rax,0x8(%rsp)
   0x0000000000401470 <+20>:    lea    0x10(%rsi),%rax
   0x0000000000401474 <+24>:    mov    %rax,(%rsp)
   0x0000000000401478 <+28>:    lea    0xc(%rsi),%r9
   0x000000000040147c <+32>:    lea    0x8(%rsi),%r8
   0x0000000000401480 <+36>:    mov    $0x4025c3,%esi
   0x0000000000401485 <+41>:    mov    $0x0,%eax
   0x000000000040148a <+46>:    call   0x400bf0 <__isoc99_sscanf@plt>
   0x000000000040148f <+51>:    cmp    $0x5,%eax
   0x0000000000401492 <+54>:    jg     0x401499 <read_six_numbers+61>
   0x0000000000401494 <+56>:    call   0x40143a <explode_bomb>
   0x0000000000401499 <+61>:    add    $0x18,%rsp
   0x000000000040149d <+65>:    ret
```

整个流程大约是：将栈指针赋给%^rdx， 将*栈指针+0x4*的值（内存地址）所指向的值赋给%rcx,将*栈指针+0x14*的值（内存地址）所指向的值赋给%rax，%rax是返回值。随后，将%rax的值（内存地址）赋值给*%rsp+0x8*（内存地址）所指向的值。随后，将*栈指针+0x10*的值所指向的值赋给%rax。随后，将%rax的值赋给%rsp。……有点长，接下去的也是类似的操作。
不过，经过整理后，可以看出，这些代码把各个寄存器的值所指向的地址设置为*栈指针* + 0，4，8，12，16，20。此刻应该有点感觉了。在调用`sscanf`之前，看到有将立即数$0x4025c3放到%esi中。根据之前的经验，可以通过`x/s 0x4025c3`来看它所指向的值。

```assembly
(gdb) x/s 0x4025c3
0x4025c3:       "%d %d %d %d %d %d"
```

因此，`<read_six_numbers>`方法显然正如他表示的，是读取六个数字。并且根据上文的格式，可知这个数字是整数。好，接下来让我们回到phase_2()中。

那么我们继续看下去。接下去执行的是` cmpl   $0x1,(%rsp)`和`je     0x400f30 <phase_2+52>`。显然，判断%rsp即栈指针内存指向的值是否为1，如果是，则跳转到+52行，不是，则炸弹爆炸。回想到`<read_six_numbers>`将读取到的数字分别存到栈指针+0、+4、+8、+12、+16、+20上去。因此，这段汇编代码告诉我们，输入的那六个数字中，第一个必须是0x1，即1<sub>d</sub>。

接下去是`lea    0x4(%rsp),%rbx`和`lea    0x18(%rsp),%rbp`。将栈指针下移0x4的内存地址赋给%rbx，将栈指针下移0x18的内存地址赋给%rbp。然后，代码又跳回去。执行`mov    -0x4(%rbx),%eax`、`add    %eax,%eax`、`cmp    %eax,(%rbx)`。即将栈指针指向的值赋给%eax，随后将%eax的值 \* 2，比较%eax和%rbx的值。根据上文`lea    0x4(%rsp),%rbx`可知，%rbx存的是第二个数字。因此，这段代码就是将第一个数字和第二个数字进行比较，判断第二个数字是否为第一个数字的两倍。如果相等，则跳转执行 `add    $0x4,%rbx`，`cmp    %rbp,%rbx`，将%rbx继续下移0x4，随后与%rbx比较。根据之前的代码可知，%rbx存的是最后一个数字的地址。所以，如果不相等，继续下移，相等，则意味着遍历结束，函数结束。

至此，整个phase_2()的代码逻辑已经十分了然了。我们甚至可以用C写出类似逻辑的代码来：

```c++
int a[6];
read_six_numbers(a);				//得到六个数字.
if(a[0] != 1) explode_bomb();		//不为1，炸弹爆炸。
else {
	for(int i =1;i<6;i++){
		if(a[i] != 2*a[i-1]) 
			explode_bomb();			//后数不为两倍，炸弹爆炸。
	}
}
```

于是，我们可以得到数字序列：1 2 4 8 16 32。运行bomb。得到:

<img src="http://satt.oss-cn-hangzhou.aliyuncs.com/img/phase_2success.png" alt="phase_2success" style="zoom:67%;" />

*hooray again!*接下去是第三个.

### phase_3()

执行`disas phase_3`得到:

```assembly
   0x0000000000400f43 <+0>:     sub    $0x18,%rsp
   0x0000000000400f47 <+4>:     lea    0xc(%rsp),%rcx
   0x0000000000400f4c <+9>:     lea    0x8(%rsp),%rdx
   0x0000000000400f51 <+14>:    mov    $0x4025cf,%esi
   0x0000000000400f56 <+19>:    mov    $0x0,%eax
   0x0000000000400f5b <+24>:    call   0x400bf0 <__isoc99_sscanf@plt>
   0x0000000000400f60 <+29>:    cmp    $0x1,%eax
   0x0000000000400f63 <+32>:    jg     0x400f6a <phase_3+39>
   0x0000000000400f65 <+34>:    call   0x40143a <explode_bomb>
   0x0000000000400f6a <+39>:    cmpl   $0x7,0x8(%rsp)
   0x0000000000400f6f <+44>:    ja     0x400fad <phase_3+106>
   0x0000000000400f71 <+46>:    mov    0x8(%rsp),%eax
   0x0000000000400f75 <+50>:    jmp    *0x402470(,%rax,8)
   0x0000000000400f7c <+57>:    mov    $0xcf,%eax
   0x0000000000400f81 <+62>:    jmp    0x400fbe <phase_3+123>
   0x0000000000400f83 <+64>:    mov    $0x2c3,%eax
   0x0000000000400f88 <+69>:    jmp    0x400fbe <phase_3+123>
   0x0000000000400f8a <+71>:    mov    $0x100,%eax
   0x0000000000400f8f <+76>:    jmp    0x400fbe <phase_3+123>
   0x0000000000400f91 <+78>:    mov    $0x185,%eax
   0x0000000000400f96 <+83>:    jmp    0x400fbe <phase_3+123>
   0x0000000000400f98 <+85>:    mov    $0xce,%eax
   0x0000000000400f9d <+90>:    jmp    0x400fbe <phase_3+123>
   0x0000000000400f9f <+92>:    mov    $0x2aa,%eax
   0x0000000000400fa4 <+97>:    jmp    0x400fbe <phase_3+123>
   0x0000000000400fa6 <+99>:    mov    $0x147,%eax
   0x0000000000400fab <+104>:   jmp    0x400fbe <phase_3+123>
   0x0000000000400fad <+106>:   call   0x40143a <explode_bomb>
   0x0000000000400fb2 <+111>:   mov    $0x0,%eax
   0x0000000000400fb7 <+116>:   jmp    0x400fbe <phase_3+123>
   0x0000000000400fb9 <+118>:   mov    $0x137,%eax
   0x0000000000400fbe <+123>:   cmp    0xc(%rsp),%eax
   0x0000000000400fc2 <+127>:   je     0x400fc9 <phase_3+134>
   0x0000000000400fc4 <+129>:   call   0x40143a <explode_bomb>
   0x0000000000400fc9 <+134>:   add    $0x18,%rsp
   0x0000000000400fcd <+138>:   ret
```

随着实验进程的推进，我们的经验越来越丰富了，当然，问题的难度应该也逐渐递增。从汇编代码的长度也可以看出来这一点。老样子，先分析一波汇编代码。

首先将%rcx赋值为%rsp+0xc的内存地址、%rdx赋值为%rsp+0x8的内存地址。随后，将立即数0x4025cf赋值给%esi。随后执行输入方法。显然老样子，输入`x/s 0x4025cf`看看格式。
`0x4025cf:       "%d %d"`.看来这次的输入是两个整数。`cmp    $0x1,%eax`的目的是判断输入是否成功，失败则炸弹爆炸。而`cmpl   $0x7,0x8(%rsp)`则是判断原%rdx指向的内存地址是否大于7.观察后续代码可知，如果大于，则炸弹爆炸。*看来第一个数字应该小于7*。随后的代码`mov    0x8(%rsp),%eax`则是给%eax赋值（输入的第一个数字）。以上逻辑可以用这张图片来表示：

<img src="http://satt.oss-cn-hangzhou.aliyuncs.com/img/phase_3的前半部分.png" alt="phase_3的前半部分" style="zoom:80%;" />

接下去看到了奇怪的代码：`jmp    *0x402470(,%rax,8)`。于是，让我们来手动算算。假设我输入的第一个数字是0，那么%rax的值也是0.接下去就只需要看看\*0x402470的值是什么.

```
(gdb) print *(0x402470)
$1 = 4198268
```

可以看到是一个奇怪的数字。我到这里就卡住了。后来看了别人的解释，才发现原来需要转换成十六进制。于是：

```
(gdb) print/x *(0x402470)
$4 = 0x400f7c
```

这个数字就比较眼熟了。

```assembly
   0x0000000000400f81 <+62>:    jmp    0x400fbe <phase_3+123>
   0x0000000000400f7c <+57>:    mov    $0xcf,%eax
```

原来如此，合理推断，剩下的汇编代码中，几个类似结构就是对应不同的输入跳转到不同的代码。目前已知，当x1=0时，x2应该是0xcf，即207<sub>d</sub>。剩下的只是类似的操作而已，不赘。

测试一波。

```bash
$ ./bomb
Welcome to my fiendish little bomb. You have 6 phases with
which to blow yourself up. Have a nice day!
Border relations with Canada have never been better.
Phase 1 defused. How about the next one?
1 2 4 8 16 32
That's number 2.  Keep going!
0 207
Halfway there!
```

phase_3()结束了，对我来说，比前两个要容易不少。接下去是phase_4。

### phase_4()








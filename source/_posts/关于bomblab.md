---
title: 关于bomblab
date: 2021-08-03 12:00:07
tags: csapp
thumbnail: http://satt.oss-cn-hangzhou.aliyuncs.com/img/bomblabtu.png
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
让我们来分析分析。首先是第0行，我们知道%rsp是保存指向栈顶的指针的寄存器，而对他进行自减，就是为了给栈帧开辟0x8个字节的空间。第1行，将立即数0x402400保存到%esi中，然后调用了一个 *<strings_not_equal>* 的方法 。在第3行，test了%eax的内容，如果相等,则跳转到第6行，即出栈，返回，函数结束;如果不相等，则跳转到 *<explode_bomb>* ，显然就是炸弹爆炸了。我们知道，%eax是32位系统中的返回值，容易料想到是保存了  *<strings_not_equal>*的返回值。自此，分析结束。
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
得到了phase_1()的答案。也就是
>Border relations with Canada have never been better.

### phase_2()
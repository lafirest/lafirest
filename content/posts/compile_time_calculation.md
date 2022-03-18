+++
title = "Programming Language Roam: 编译期计算"
date = 2022-03-18T21:09:00+08:00
tags = ["Common-Lisp"]
draft = false
+++

最近在工作中的一些场景,让我想到了「编译期计算」这个概念,然而一直没有时间整理、复习下相关概念。


## 什么是编译期计算 {#什么是编译期计算}

「编译期计算」是指: **可以在编译期间执行的的运算或者这种能力**

这并不是什么新的概念,最古老的语言之一的 _C_ 的 _宏_ 就具有很弱的编译期计算能力,而之后的编译语言中,凡是支持
元编程的,多多少少都能支持编译期间预算。

而另外一方面,编译期计算也是一种常见的编译优化手段,高优化的编译器会尽可能的,对在编译期间可以求值的计算过程进行
求值,然后内联其计算结果,比如：如果一个循环内所有变量都是常量,在 -O2 和 -O3 的情况下,GCC 生成的代码多半只有
循环结果,而没有循环的过程。

但是,相对而言,大部分编程语言所支持的 _编译期计算_ 和 Lisp 比起来,都很弱

做为最古老的编程语言之一,Lisp 发明了很多 **超时代** 的特性,最出名的莫过于 **垃圾回收(GC)** 了,但是 Lisp 对
计算的控制粒度和灵活度,目前也鲜有对手。


## Common Lisp 和 编译期计算 {#common-lisp-和-编译期计算}

C++11 引入了一个关键字 [constexpr](https://en.cppreference.com/w/cpp/language/constexpr),用来声明一个函数或者变量,在编译期是可能进行求值的,相比 _宏_ 和 _模板_ ,
这是一个更加强大的编译期计算的能力。

不过,这种能力却是 30 多年前的 Common Lisp 里面的内建功能,这里用一个求和函数进行演示

```lisp
(defpackage ct
  (:use :cl)
  (:export :test1 :test2))

(in-package ct)

;;; 声明一个求和函数 sum
;;; 设定函数 sum 具有在编译期和执行时进行运算的能力
;;; 默认情况下函数只能在执行时(execute)才能进行计算
(eval-when (:compile-toplevel :execute)
           (defun sum (&rest args)
             (apply #'+ args)))

;;; 声明函数 test1,使用求和函数求值
(defun test1 ()
  (let ((x (sum 1 2 3 )))
    x))

;;; 声明函数 test2,使用语言自带的加法运算求值
(defun test2 ()
  (let ((x (+ 1 2 3)))
    x))

```

然后在 REPL 中执行编译和加载

```lisp
(compile-file "ct.lisp")
(load "ct.fasl")
```

先输出 test2 的汇编代码

```lisp
(disassemble #'ct:test2)
```

```asm
; disassembly for CT:TEST2
; Size: 21 bytes. Origin: #x534C531C                          ; CT:TEST2
; 1C:       498B4510         MOV RAX, [R13+16]                ; thread.binding-stack-pointer
; 20:       488945F8         MOV [RBP-8], RAX
; 24:       BA0C000000       MOV EDX, 12
; 上面的 12 就是 1 + 2 + 3 = 6 在 SBCL 中的表示, SBCL 中的 fixnum 最低位应该是 0
; 所以值相当于左移了一位
; 29:       488BE5           MOV RSP, RBP
; 2C:       F8               CLC
; 2D:       5D               POP RBP
; 2E:       C3               RET
; 2F:       CC10             INT3 16                          ; Invalid argument count trap
```

可以看到 SBCL 编译器实现的加法运算在编译期对运算进行了求值,最后生成的代码中只剩下了求值结果,而没求值过程

再来输出 test1 中的汇编代码

```lisp
(disassemble #'ct:test1)
```

```asm
; disassembly for CT:TEST1
; Size: 56 bytes. Origin: #x535C05CC                          ; CT:TEST1
; 5CC:       498B4510         MOV RAX, [R13+16]               ; thread.binding-stack-pointer
; 5D0:       488945F8         MOV [RBP-8], RAX
; 5D4:       4883EC10         SUB RSP, 16
; 5D8:       BA02000000       MOV EDX, 2
; 5DD:       BF04000000       MOV EDI, 4
; 5E2:       BE06000000       MOV ESI, 6
; 上面三行代码分别将 1 2 3 拷贝到寄存器中
; 5E7:       B906000000       MOV ECX, 6
; 5EC:       48892C24         MOV [RSP], RBP
; 5F0:       488BEC           MOV RBP, RSP
; 5F3:       E80AC1E2FC       CALL #x503EC702                 ; #<FDEFN CT::SUM>
; 调用 ct::sum 求值
; 5F8:       480F42E3         CMOVB RSP, RBX
; 5FC:       488BE5           MOV RSP, RBP
; 5FF:       F8               CLC
; 600:       5D               POP RBP
; 601:       C3               RET
; 602:       CC10             INT3 16                         ; Invalid argument count trap
```

可以看到 _sum_ 函数没有在编译期被求值,汇编代码忠实的还原了函数调用的过程

但是,在文件开通, _sum_ 被声明可以在编译期进行计算,那为什么这里没有被计算呢？

这里, _sum_ 之所以没有被计算,是因为它是在一个函数(test1)的声明中,在编译期间,编译器只是生成了一个函数,
并没有任何地方告诉编译器,在这个函数内部的某个表达式中,有一个 _sum_ 是可以被求值的, 所以只要能在编译器进
行编译时,触发对 _sum_ 的求值就行

```lisp
(defmacro compile-time-sum (&rest args)
  "定义一个宏,让其在编译时调用sum,而不是返回一个表达式"
  (apply #'sum args))

(defun test3 ()
  "函数 test3 使用上面的宏替换掉 sum"
  (let ((x (compile-time-sum 1 2 3)))
    x))
```

对文件重新编译、加载后,查看 test3 的汇编代码

```asm
; disassembly for CT::TEST3
; Size: 21 bytes. Origin: #x5358324C                          ; CT::TEST3
; 4C:       498B4510         MOV RAX, [R13+16]                ; thread.binding-stack-pointer
; 50:       488945F8         MOV [RBP-8], RAX
; 54:       BA0C000000       MOV EDX, 12
; 59:       488BE5           MOV RSP, RBP
; 5C:       F8               CLC
; 5D:       5D               POP RBP
; 5E:       C3               RET
; 5F:       CC10             INT3 16                          ; Invalid argument count trap
```

可以看到,test3 生成的代码,和编译器自己实现的加法运算的代码完全一样,计算过程被优化掉了,只剩下计算结果

但是这样就会产生另外一个问题,对于每个编译期函数,如果都需要进行一层包装,岂不是很麻烦？实际上,并不是,Common Lisp
中有专门定义 _编译器宏_ 的宏


## define-compilr-macro {#define-compilr-macro}

先看代码,

```lisp
(defpackage ct2
  (:use :cl)
  (:export :test1 :test2))

(in-package ct2)

;;; 和之前一样,先声明求和函数可以在编译期进行计算
(eval-when (:compile-toplevel :load-toplevel  :execute)
  (defun sum (&rest args)
    (apply #'+ args)))

;;; 定义一个编译器宏 sum, 这个宏的逻辑如下:
;;; 如果所有参数在编译期间都是常量,则调用 求和函数sum 进行求值
;;; 如果不是,则返回原表达式
;;; 这里实际上就是相当于在编译时,对语法树进行重写
(define-compiler-macro sum (&whole from &rest args)
  (if (every #'constantp args)
      (apply #'sum args)
      from))

;;; test1 里面 sum 的参数全是常量
(defun test1 (x)
  (let ((y (sum 1 2 3 4)))
    y))

;;; test2 里面 sum 的参数含有变量
(defun test2 (x)
  (let ((y (sum 1 2 3 x)))
    y))

```

test1 的汇编代码:

```asm
; disassembly for CT2:TEST1
; Size: 21 bytes. Origin: #x535C178D                          ; CT2:TEST1
; 8D:       498B4510         MOV RAX, [R13+16]                ; thread.binding-stack-pointer
; 91:       488945F8         MOV [RBP-8], RAX
; 95:       BA14000000       MOV EDX, 20
; 9A:       488BE5           MOV RSP, RBP
; 9D:       F8               CLC
; 9E:       5D               POP RBP
; 9F:       C3               RET
; A0:       CC10             INT3 16                          ; Invalid argument count trap

```

test2 的汇编代码:

```asm
; disassembly for CT2:TEST2
; Size: 92 bytes. Origin: #x533F0D54                          ; CT2:TEST2
; 54:       498B4510         MOV RAX, [R13+16]                ; thread.binding-stack-pointer
; 58:       488945F8         MOV [RBP-8], RAX
; 5C:       4C8D4424F0       LEA R8, [RSP-16]
; 61:       4883EC20         SUB RSP, 32
; 65:       BA02000000       MOV EDX, 2
; 6A:       BF04000000       MOV EDI, 4
; 6F:       BE06000000       MOV ESI, 6
; 74:       4D8948F0         MOV [R8-16], R9
; 78:       B908000000       MOV ECX, 8
; 7D:       498928           MOV [R8], RBP
; 80:       498BE8           MOV RBP, R8
; 83:       E85ABAFFFC       CALL #x503EC7E2                  ; #<FDEFN CT2::SUM>
; 88:       480F42E3         CMOVB RSP, RBX
; 8C:       4C8B4DF0         MOV R9, [RBP-16]
; 90:       488D42F1         LEA RAX, [RDX-15]
; 94:       A801             TEST AL, 1
; 96:       750D             JNE L0
; 98:       3C0A             CMP AL, 10
; 9A:       7409             JEQ L0
; 9C:       A80F             TEST AL, 15
; 9E:       750B             JNE L1
; A0:       803829           CMP BYTE PTR [RAX], 41
; A3:       7706             JNBE L1
; A5: L0:   488BE5           MOV RSP, RBP
; A8:       F8               CLC
; A9:       5D               POP RBP
; AA:       C3               RET
; AB: L1:   CC57             INT3 87                          ; OBJECT-NOT-NUMBER-ERROR
; AD:       08               BYTE #X08                        ; RDX(d)
; AE:       CC10             INT3 16                          ; Invalid argument count trap
```

可以看见,test1 因为符合 编译器宏 sum 里面的运算规则,所以在编译期间被求值了,而 test2 因为含有变量,无法被求值,
所以最终生成的代码并没有优化,而是忠实的还原了函数调用过程。(实际上只要变量也是可以在编译期求值的,test2 也是可以
进行优化的,不过那样的例子太复杂了)


## 编译期求值可以做什么 {#编译期求值可以做什么}

因为真实世界存在大量 IO 副作用,不存在真的的纯函数,所以编译期求值的适用范围比运行时求值要小很多。

最常见的还是数值计算、文本处理、序列化/反序列化、类型转换等的优化。

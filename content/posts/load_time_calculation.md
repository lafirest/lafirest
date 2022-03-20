+++
title = "Programming Language Roam: Load-Time Calculation"
date = 2022-03-20T17:08:00+08:00
tags = ["Common-Lisp"]
draft = false
+++

「加载期计算」，是指: **在代码加载时执行运算或者这种能力**

和 [编译期计算]({{< relref "compile_time_calculation" >}}) 相比，加载期计算可能适用范围更广


## 场景 {#场景}

加载期计算主要适用于以下的场景：

1.  程序运行时需要的某些信息，如环境变量等，无法在编译期间确定
2.  这些信息在程序启动后几乎不会发生变化
3.  程序运行时需要根据不同的环境信息，执行不同的运算

比较典型的场景就是跨平台应用，以及使用静态配置的应用，这些应用在真正运行前是无法获得目标平台、或者目标配置的
值。而运行后，要么对这些信息进行缓存以便反复使用，要么就是每次使用这些信息时，都重新进行一次读取/查询。

而如果程序支持「加载期计算」，则可以在代码加载时，读取相关信息，然后直接将信息写入到调用的代码处，甚至，可以根据
信息的值来选择性生成不同的处理代码。

这样，对于程序而言，这些信息就变成了静态的常量了，且无用的代码分支也会被裁剪掉，这样就减少了程序在缓存、调用、
分支预测上的开销了


## Load-Time Hook/Callback {#load-time-hook-callback}

部分语言，比如 Erlang、C#(Unity) 等支持在模块、类、程序集加载时，执行某个函数，这种行为可以看作是代码加载时的一个
回调，或者说是一种受限的「加载期计算」，因为这样的计算，执行的都是编译期已经生成好的代码，无法通过在加载期间执行的运算来影响到原有的代码逻辑。

C# (.Net) 可以在加载时，找到对应属性的程序集、类、方法、乃至字段等，然后动态修改、生成代码。
这种操作，可以算是在代码加载期间执行的 JIT 操作，而「加载期计算」更多的是强调「计算」或者说 「求值」，
但总的来说，区别不大，可以看作是一回事。


## 示例，零开销配置 {#示例-零开销配置}

这里做一个简单的演示，使用 _message_ 文本文件做为配置文件，里面的内容为:

```text
Hello， World!
```

定义函数 _get-message_ ，直接读取整个文本文件，当作是配置读取操作：

```lisp
(defun get-message ()
  (uiop:read-file-string "~/message"))
```

定义函数 _test_ ， 直接使用 _get-message_ 模拟读取配置操作

```lisp
(defun test ()
  (let ((msg (get-message)))
    msg))
```

为了防止编译器优化，这里写的很啰嗦，第三行的 _msg_ 实际上可以看作是具体的配置操作过程，只是这里简单的用返回这个行为
来模拟

定义一个函数 _test2_ ， 使用加载期计算，在代码加载时，求出 _get-message_ 的值，然后直接写入到 _test2_ 的代码内

```lisp
(eval-when (:load-toplevel)
  (let ((msg (get-message)))
    (defun test2 ()
      (let ((in-msg `，msg))
        in-msg))))
```

现在来看， _test_ 生成的汇编代码:

```asm
;disassembly for RT::TEST
;Size: 40 bytes. Origin: #x5359C06C                           RT::TEST
;6C:       498B4510         MOV RAX， [R13+16]                 thread.binding-stack-pointer
;70:       488945F8         MOV [RBP-8]， RAX
;74:       4883EC10         SUB RSP， 16
;78:       31C9             XOR ECX， ECX
;7A:       48892C24         MOV [RSP]， RBP
;7E:       488BEC           MOV RBP， RSP
;81:       B8A2C73E50       MOV EAX， #x503EC7A2               #<FDEFN RT::GET-MESSAGE>
;86:       FFD0             CALL RAX
;88:       480F42E3         CMOVB RSP， RBX
;8C:       488BE5           MOV RSP， RBP
;8F:       F8               CLC
;90:       5D               POP RBP
;91:       C3               RET
;92:       CC10             INT3 16                           Invalid argument count trap
```

可以看到， _test_ 调用了 _get-message_ ，且整个执行过程指令很多，一直到 _91_ 处才 _RET_

然后， _test2_ 的汇编代码:

```asm
;disassembly for RT::TEST2
;Size: 38 bytes. Origin: #x535D21F0                           RT::TEST2
;1F0:       498B4510         MOV RAX， [R13+16]                thread.binding-stack-pointer
;1F4:       488945F0         MOV [RBP-16]， RAX
;1F8:       488BE5           MOV RSP， RBP
;1FB:       F8               CLC
;1FC:       5D               POP RBP
;1FD:       C3               RET
;1FE: L0:   FF24255800A052   JMP QWORD PTR [#x52A00058]       ALLOC-TRAMP
;205:       CC10             INT3 16                          Invalid argument count trap
;207:       6A20             PUSH 32
;209:       E8F0FFFFFF       CALL L0
;20E:       5E               POP RSI
;20F:       E976FFFFFF       JMP #x535D218A
;214:       CC10             INT3 16                          Invalid argument count trap
```

可以看到， _test2_ 中没有调用 _get-message_ 函数，且在 _1FD_ 处就已经 _RET_ 了，整个执行过程实际上只有一条
_MOV_ 指令，用于把常量区的文本地址拷贝到寄存器中。

上面的示例比较简单，实际上使用时，可以实现的很复杂，比如跨平台应用，如果不想针对不同的平台进行单独打包，希望一个包
可以运行在所用平台上，但又不想在代码执行时不断的做平台判断，就可以使用加载期计算，在代码加载时，读取当前的平台信息，
然后使用和上面示例中一样的方法，只生成/插入对应平台的代码就行。

另外一些有趣的应用，可以看作是手动 JIT，如果程序运行时，会生成一些统计信息，比如各个分支的成功概率等，那么下次代码重新
加载时，可以读取进行信息，然后重新优化分支顺序，提高分支预测的成功率等。
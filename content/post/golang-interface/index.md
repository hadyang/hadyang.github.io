---
title: 【Golang进阶】结构体与方法
draft: false
date: 2021-04-10
categories: 
    - Go
---

在之前的 [《【Golang进阶】函数与栈》]({{< ref "/post/golang-func-stack" >}})文章中，我们通过汇编学习了 Go 的栈结构。今天，我们也通过汇编再来看看结构体和方法调用的实现。

## 初始化

首先，我们定一个结构体 `Dog` 以及一个指针接收者的方法 `Say`，下面我把行号也直接写上，方便与汇编代码对应。

```go
3:	type Dog struct {
4:		Name string
5:		Age  int
6:	}
7:	
8:	func (t *Dog) Say() string {
9:		return t.Name
10:	}
11:	
12:	func main() {
13:		var d1 = &Dog{"煎饼", 43}
14:		_ = d1.Name
15:		_ = d1.Say()
16:	}
```

执行命令 `go tool compile -S -l -N main.go` 得到汇编代码，我们先来看 `main` 函数

```nasm
"".main STEXT size=123 args=0x0 locals=0x48 funcid=0x0
	...
	0x001d 00029 (main.go:13)	MOVQ	$0, ""..autotmp_2+40(SP)
	0x0026 00038 (main.go:13)	XORPS	X0, X0
	0x0029 00041 (main.go:13)	MOVUPS	X0, ""..autotmp_2+48(SP)
	0x002e 00046 (main.go:13)	LEAQ	""..autotmp_2+40(SP), AX
	0x0033 00051 (main.go:13)	MOVQ	AX, ""..autotmp_1+32(SP)
	0x0038 00056 (main.go:13)	TESTB	AL, (AX)
	0x003a 00058 (main.go:13)	MOVQ	$6, ""..autotmp_2+48(SP)
	0x0043 00067 (main.go:13)	LEAQ	go.string."煎饼"(SB), CX
	0x004a 00074 (main.go:13)	MOVQ	CX, ""..autotmp_2+40(SP)
	0x004f 00079 (main.go:13)	TESTB	AL, (AX)
	0x0051 00081 (main.go:13)	MOVQ	$43, ""..autotmp_2+56(SP)
	0x005a 00090 (main.go:13)	MOVQ	AX, "".d1+24(SP)
	0x005f 00095 (main.go:14)	TESTB	AL, (AX)
	0x0061 00097 (main.go:15)	MOVQ	AX, (SP)
	0x0065 00101 (main.go:15)	PCDATA	$1, $0
	0x0065 00101 (main.go:15)	CALL	"".(*Dog).Say(SB)
	...
	rel 5+4 t=17 TLS+0
	rel 70+4 t=16 go.string."煎饼"+0
	rel 102+4 t=8 "".(*Dog).Say+0
	rel 117+4 t=8 runtime.morestack_noctxt+0
    ...
go.string."煎饼" SRODATA dupok size=6
	0x0000 e7 85 8e e9 a5 bc                                ......
```




```nasm
"".(*Dog).Say STEXT nosplit size=33 args=0x18 locals=0x0 funcid=0x0
    ...
	0x0000 00000 (main.go:8)	XORPS	X0, X0
	0x0003 00003 (main.go:8)	MOVUPS	X0, "".~r0+16(SP)
	0x0008 00008 (main.go:9)	MOVQ	"".t+8(SP), AX
	0x000d 00013 (main.go:9)	TESTB	AL, (AX)
	0x000f 00015 (main.go:9)	MOVQ	(AX), CX
	0x0012 00018 (main.go:9)	MOVQ	8(AX), AX
	0x0016 00022 (main.go:9)	MOVQ	CX, "".~r0+16(SP)
	0x001b 00027 (main.go:9)	MOVQ	AX, "".~r0+24(SP)
	0x0020 00032 (main.go:9)	RET
    ...
```

TESTB    AL, (AX)

Load the value pointed to by AX and do something with it. The parens around AX mean dereference the pointer in the AX register. It doesn't matter here what the TESTB instruction does; it was chosen because it is short to encode. It's the deferencing that matters. If the load faults, the runtime will receive a signal and turn that into a panic.

从 AX 中加载数据，如果加载失败会触发段错误信号 `SIGSEGV`，触发 Go Runtime 抛出 Panic。选择 TESTB 指令仅仅是因为其短小。

```
    14:	func main() {
    15:		var c1 = &Cat{"ASHAO", 43}
    16:		var s1 fmt.Stringer = c1
    17:	
    18:		s1.String()
    19:	}
```

```nasm
...
0x0021 00033 (main.go:15)    MOVQ       $0, ""..autotmp_4+72(SP)
0x002a 00042 (main.go:15)    XORPS      X0, X0
0x002d 00045 (main.go:15)    MOVUPS     X0, ""..autotmp_4+80(SP)
0x0032 00050 (main.go:15)    LEAQ       ""..autotmp_4+72(SP), AX
0x0037 00055 (main.go:15)    MOVQ       AX, ""..autotmp_3+40(SP)
0x003c 00060 (main.go:15)    TESTB      AL, (AX)
0x003e 00062 (main.go:15)    MOVQ       $5, ""..autotmp_4+80(SP)
0x0047 00071 (main.go:15)    LEAQ       go.string."ASHAO"(SB), CX
0x004e 00078 (main.go:15)    MOVQ       CX, ""..autotmp_4+72(SP)
0x0053 00083 (main.go:15)    TESTB      AL, (AX)
0x0055 00085 (main.go:15)    MOVQ       $43, ""..autotmp_4+88(SP)
0x005e 00094 (main.go:15)    MOVQ       AX, "".c1+24(SP)
...
```

```nasm
...
main.go:15    0x49a221    48c744244800000000        mov qword ptr [rsp+0x48], 0x0
main.go:15    0x49a22a    0f57c0                    xorps xmm0, xmm0
main.go:15    0x49a22d    0f11442450                movups xmmword ptr [rsp+0x50], xmm0
main.go:15    0x49a232    488d442448                lea rax, ptr [rsp+0x48]
main.go:15    0x49a237    4889442428                mov qword ptr [rsp+0x28], rax
main.go:15    0x49a23c    8400                      test byte ptr [rax], al
main.go:15    0x49a23e    48c744245005000000        mov qword ptr [rsp+0x50], 0x5
main.go:15    0x49a247    488d0d36210200            lea rcx, ptr [rip+0x22136]
main.go:15    0x49a24e    48894c2448                mov qword ptr [rsp+0x48], rcx
main.go:15    0x49a253    8400                      test byte ptr [rax], al
main.go:15    0x49a255    48c74424582b000000        mov qword ptr [rsp+0x58], 0x2b
main.go:15    0x49a25e    4889442418                mov qword ptr [rsp+0x18], rax
...
```



```nasm
0x0063 00099 (main.go:16)    MOVQ    AX, ""..autotmp_2+48(SP)
0x0068 00104 (main.go:16)    LEAQ    go.itab.*"".Cat,fmt.Stringer(SB), CX
0x006f 00111 (main.go:16)    MOVQ    CX, "".s1+56(SP)
0x0074 00116 (main.go:16)    MOVQ    AX, "".s1+64(SP)
```

```nasm
main.go:16    0x49a263    4889442430        mov qword ptr [rsp+0x30], rax
main.go:16    0x49a268    488d0d09ef0200    lea rcx, ptr [rip+0x2ef09]
main.go:16    0x49a26f    48894c2438        mov qword ptr [rsp+0x38], rcx
main.go:16    0x49a274    4889442440        mov qword ptr [rsp+0x40], rax
```


```nasm
    0x0079 00121 (main.go:18)    MOVQ    "".s1+56(SP), AX
    0x007e 00126 (main.go:18)    MOVQ    "".s1+64(SP), CX
    0x0083 00131 (main.go:18)    LEAQ    go.itab.*"".Cat,fmt.Stringer(SB), DX
    0x008a 00138 (main.go:18)    CMPQ    DX, AX
    0x008d 00141 (main.go:18)    JEQ    145
    0x008f 00143 (main.go:18)    JMP    169
    0x0091 00145 (main.go:18)    MOVQ    CX, ""..autotmp_5+32(SP)
    0x0096 00150 (main.go:18)    MOVQ    CX, (SP)
    0x009a 00154 (main.go:18)    PCDATA    $1, $0
    0x009a 00154 (main.go:18)    CALL    "".(*Cat).String(SB)
    0x009f 00159 (main.go:19)    MOVQ    96(SP), BP
    0x00a4 00164 (main.go:19)    ADDQ    $104, SP
    0x00a8 00168 (main.go:19)    RET
    0x00a9 00169 (main.go:18)    MOVQ    AX, (SP)
    0x00ad 00173 (main.go:18)    LEAQ    type.*"".Cat(SB), AX
    0x00b4 00180 (main.go:18)    MOVQ    AX, 8(SP)
    0x00b9 00185 (main.go:18)    LEAQ    type.fmt.Stringer(SB), AX
    0x00c0 00192 (main.go:18)    MOVQ    AX, 16(SP)
    0x00c5 00197 (main.go:18)    CALL    runtime.panicdottypeI(SB)
    0x00ca 00202 (main.go:18)    XCHGL    AX, AX
    0x00cb 00203 (main.go:18)    NOP
    0x00cb 00203 (main.go:14)    PCDATA    $1, $-1
    0x00cb 00203 (main.go:14)    PCDATA    $0, $-2
    0x00cb 00203 (main.go:14)    CALL    runtime.morestack_noctxt(SB)
    0x00d0 00208 (main.go:14)    PCDATA    $0, $-1
    0x00d0 00208 (main.go:14)    JMP    0
```

```nasm
TEXT main.main(SB) /home/shihaoyang/Projects/LeetCode-Go/main.go


    main.go:18    0x49a279    488b442438        mov rax, qword ptr [rsp+0x38]
    main.go:18    0x49a27e    488b4c2440        mov rcx, qword ptr [rsp+0x40]
    main.go:18    0x49a283    488d15eeee0200        lea rdx, ptr [rip+0x2eeee]
    main.go:18    0x49a28a    4839c2            cmp rdx, rax
    main.go:18    0x49a28d    7402            jz 0x49a291
    main.go:18    0x49a28f    eb18            jmp 0x49a2a9
    main.go:18    0x49a291    48894c2420        mov qword ptr [rsp+0x20], rcx
    main.go:18    0x49a296    48890c24        mov qword ptr [rsp], rcx
    main.go:18    0x49a29a    e821ffffff        call $main.(*Cat).String
    main.go:19    0x49a29f    488b6c2460        mov rbp, qword ptr [rsp+0x60]
    main.go:19    0x49a2a4    4883c468        add rsp, 0x68
    main.go:19    0x49a2a8    c3            ret
    main.go:18    0x49a2a9    48890424        mov qword ptr [rsp], rax
    main.go:18    0x49a2ad    488d05ecda0000        lea rax, ptr [rip+0xdaec]
    main.go:18    0x49a2b4    4889442408        mov qword ptr [rsp+0x8], rax
    main.go:18    0x49a2b9    488d05e0fc0000        lea rax, ptr [rip+0xfce0]
    main.go:18    0x49a2c0    4889442410        mov qword ptr [rsp+0x10], rax
    main.go:18    0x49a2c5    e8160df7ff        call $runtime.panicdottypeI
    main.go:18    0x49a2ca    90            nop
    main.go:14    0x49a2cb    e87026fdff        call $runtime.morestack_noctxt
    .:0        0x49a2d0    e92bffffff        jmp $main.main
```
---
title: 【Golang基础】Map实现
draft: true
date: 2021-03-26
categories: 
    - Go
---

map 作为 go 的基础数据类型，使用场景广泛，但其底层原理隐藏的很深。map 在底层存储时将内存拆分为桶（bucket），每个桶能容纳的 kv 为 8 个，多个桶组成一个内存数组。当 map 容量大于 `bucket*6.5` 时，会进行扩容操作。



主要使用 内存哈希（memhash） 字符串哈希（strhash） 算法


## 初始化

map 在 go 中通过 `runtime.hmap` 来表示，其中的 hash 种子由 `runtime.fastrand` 生成。当 map 的第一个 bucket 能在栈上分配时，优先在栈上进行分配。否则，调用 `runtime.makemap` 在堆上进行分配。

```go
type hmap struct {
	count     int // map 中key的个数
	flags     uint8
	B         uint8  // log2(bucket)
	noverflow uint16 // approximate number of overflow buckets; see incrnoverflow for details
	hash0     uint32 // hash种子；由 runtime.fastrand 生成
	buckets    unsafe.Pointer // 存储数据的数组，包含 2^B 个 bucket
	oldbuckets unsafe.Pointer //
	nevacuate  uintptr        // progress counter for evacuation (buckets less than this have been evacuated)
	extra *mapextra
}
```

在栈上初始化 map 的时候，会使用 `DUFFZERO` 指令，该指令非 CPU 指令集下的，其在生成可执行文件时会被解析为函数调用，跳转到 `runtime·duffzero` 函数偏移量为 250 的汇编代码中，完成内存的初始化。 `DUFFZERO` 初始化后的空间，会赋值到 `buckets` 上，作为 map 的存储空间。




## Hash算法



## 扩容




func main() {
	m := make(map[int32]string, 8)
	m[1] = "xxx"
}

CALL	runtime.mapassign_fast32(SB) //mapassign_fast32(t *maptype, h *hmap, key uint32)


rsp=0x000000c00007be58
rbp = 0x000000c00007bf78

                            SP+48               0x000000c00007be88
                            SP+56               0x000000c00007be90
count     int               SP+64   rsp+0x40    0x000000c00007be98
flags     uint8             SP+72               0x000000c00007bea0
B         uint8                                 0x000000c00007bea1
noverflow uint16                                0x000000c00007bea2
hash0     uint32                                0x000000c00007bea4
buckets    unsafe.Pointer   SP+80   rsp+0x50    0x000000c00007bea8
oldbuckets unsafe.Pointer   SP+88
nevacuate  uintptr          SP+96   rsp+0x60    0x000000c00007beb8
extra *mapextra             SP+104
                            SP+112

rdi=0x000000c00007bec8
rdi=0x000000c00007beb8

0x000000c00007be48=rbp(0x000000c00007bf78)
rbp=0x000000c00007be48

DUFFZERO	$250                    ;rdi=0x000000c00007bf78     3x64byte
LEAQ	""..autotmp_1+64(SP), AX    ;rax=0x000000c00007be98
MOVQ	AX, ""..autotmp_3+56(SP)    ;SP+56=&count
LEAQ	""..autotmp_2+112(SP), AX   ;rax=0x000000c00007bec8
MOVQ	AX, ""..autotmp_1+80(SP)    ;buckets=0x000000c00007bec8
LEAQ	""..autotmp_1+64(SP), AX    ;rax=0x000000c00007be98
MOVQ	AX, ""..autotmp_4+48(SP)    ;SP+48=&count
CALL	runtime.fastrand(SB)        ;SP=fastrand 返回值
MOVQ	""..autotmp_4+48(SP), AX    ;AX=0x000000c00007be98
MOVL	(SP), CX                    ;CX=fastrand 返回值
MOVL	CX, 12(AX)                  ;hash=fastrand 返回值
LEAQ	""..autotmp_1+64(SP), AX    ;
MOVQ	AX, "".m+32(SP)             ;


// makemap implements Go map creation for make(map[k]v, hint).
// If the compiler has determined that the map or the first bucket
// can be created on the stack, h and/or bucket may be non-nil.
// If h != nil, the map can be created directly in h.
// If h.buckets != nil, bucket pointed to can be used as the first bucket.
func makemap(t *maptype, hint int, h *hmap) *hmap
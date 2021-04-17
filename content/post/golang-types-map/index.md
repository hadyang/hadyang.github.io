---
title: 【Golang基础】Map实现
draft: true
date: 2021-03-26
categories: 
    - Go
---


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
flags     uint8             SP+72
B         uint8             
noverflow uint16            
hash0     uint32            
buckets    unsafe.Pointer   SP+80   rsp+0x50    0x000000c00007bea8
oldbuckets unsafe.Pointer   SP+88
nevacuate  uintptr          SP+96   rsp+0x60    0x000000c00007beb8
extra *mapextra             SP+104
                            SP+112

rdi=0x000000c00007bec8
rdi=0x000000c00007beb8

0x000000c00007be48=rbp(0x000000c00007bf78)
rbp=0x000000c00007be48

DUFFZERO	$250                    ;rdi=0x000000c00007bf78
LEAQ	""..autotmp_1+64(SP), AX    ;rax=0x000000c00007be98
MOVQ	AX, ""..autotmp_3+56(SP)    ;SP+56=&count
LEAQ	""..autotmp_2+112(SP), AX   ;rax=0x000000c00007bec8
MOVQ	AX, ""..autotmp_1+80(SP)    ;buckets=0x000000c00007bec8
LEAQ	""..autotmp_1+64(SP), AX    ;rax=0x000000c00007be98
MOVQ	AX, ""..autotmp_4+48(SP)    ;SP+48=&count
CALL	runtime.fastrand(SB)        ;
MOVQ	""..autotmp_4+48(SP), AX    ;
MOVL	(SP), CX                    ;
MOVL	CX, 12(AX)                  ;
LEAQ	""..autotmp_1+64(SP), AX    ;
MOVQ	AX, "".m+32(SP)             ;
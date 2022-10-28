## 1 计算机启动过程

### 1.1 BIOS程序存储在ROM中，ROM映射地址段为:

|起始|结束|大小|描述|
|-|-|-|-|
|0xFFFF0|0xFFFFF|16B|0xFFFF0是BIOS入口地址，对应跳转指令 jmp f000:e05b|
|0xF0000|0xFFFEF|64KB-16B|BIOS程序的地址范围|

### 1.2 CPU启动时，默认 cs : ip 为 0xf000 : 0xfff0

### 1.3 通过 地址加法器 得到20位地址 0xffff0

### 1.3 CPU读取地址 0xffff0 对应的BIOS指令 jmp f000:e05b

### 1.4 CPU 跳转地址 0xfe05b 读取指令，BIOS代码真正开始的地方

### 1.5 BIOS开始检测内存、显卡等外设，当检测通过，初始化好硬件后，开始在内存中0x000 ~ 0x3FF建立数据结构，中断向量表IVT并填写中断例程。
|起始|结束|大小|描述|
|-|-|-|-|
|0x000|0x3FF|1KB|Interrupt Vector Table(中断向量表)|

### 1.6 检测完成后，调用BIOS中断0x19h 汇编语句为 call int 19h，此中断函数检测计算机有多少硬盘和软盘。

### 1.7 检测到任何可用的磁盘后，校验磁盘 0盘 0道 1扇区（1扇区 = 512B），当扇区最后两个字节内容分别是魔数 0x55 和 0xaa，BIOS认为此扇区存在可执行的主引导记录（MBR）。

### 1.8 BIOS将MBR加载至DRAM地址 0x7c00 ~ 0x7dff
|起始|结束|大小|描述|
|-|-|-|-|
|0x7C00|0x7DFF|512KB|BIOS加载 0盘 0道 1扇区 MBR 到此地址段|

### 1.9 BIOS 跳转至 0x7c00 汇编语句为 jmp 0 : 0x7c00。地址跳转至0x7c00后，启动由BIOS程序转到MBR。

## 2 Main Boot Record（MBR） 主引导记录

### 2.1 NASM 汇编语法说明

|汇编语句|说明|
|-|-|
|SECTION MBR vstart=0x7c00|==vstart== 告诉编译器把其后面的汇编语句对应于地址编码为 0x7c00|




```assembler
;main boot record
;--------------------------------------------------------------------------------
SECTION MBR vstart=0x7c00
	mov ax,cs
	mov ds,ax
	mov es,ax
	mov ss,ax
	mov fs,ax
	mov sp,0x7c00

;clear screen untilize 0x06, roll up all rows, then clear screen
;----------------------------------------------------------------------------------
;INT 0x10	function code: 0x06	description:roll up window
;----------------------------------------------------------------------------------
;Input:
;AH funcation code = 0x06
;AL = number of roll up rows(0 means roll up all)
;BH = attribute of roll up rows
;(CL,CH) = windows left up corner position(x,y)
;(DL,DH) = windows right down corner position(x,y)
;Non return:
	mov ax, 0x600
	mov bx, 0x700
	mov cx, 0	; left up corner:(0,0)
	mov dx, 0x184f	; right down corner:(80,25)
			; VGA text model, 80 chars in 1 row, and total 25 rows
			; Index start from 0, 0x18=24, 0x4f=79
	int 0x10	; int 0x10

;;;;;;;;;;;;;;;;;;;;;;;;;;; Get the position of cursor ;;;;;;;;;;;;;;;;;;;;;;;;;;;
; .get_cursor get the position and print chars
	mov ah, 3	;input: 3 function means get the position of cursor, and save it to ah
	mov bh, 0	;bh store the page of cursor
	
	int 0x10	;output: ch=cursor start row, cl=cursor end row
	
;;;;;;;;;;;;;;;;;;;;;;;;;;; Get the end position of cursor ;;;;;;;;;;;;;;;;;;;;;;;

;;;;;;;;;;;;;;;;;;;;;;;;;;; print chars                ;;;;;;;;;;;;;;;;;;;;;;;;;;;
	; still use 10h breaker, and use 13 function to print chars
	mov ax, message
	mov bp, ax	;es:bp is string address, es is the same as cs
	
	; cursor need to use the content in dx, can ignore the position in cx
	mov cx, 5	;cx is string length
	mov ax, 0x1301	;13 function code is display char and feature, store in ah
			;al set up write char method al=01: display string, move the cursor
	mov bx, 0x2	;bh store number of page, here is page 0
			; bl is char feature, black background green font color(bl=02h)
	int 0x10	; excute BIOS 0x10 breaker

;;;;;;;;;;;;;;;;;;;;;;;;;;; print string end           ;;;;;;;;;;;;;;;;;;;;;;;;;;;
	jmp $		; hold on 

	message db "1 MBR"
	times 510-($-$$) db 0
	db 0x55, 0xaa
```


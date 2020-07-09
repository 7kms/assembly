# int指令
int 指令引发的中断也是内中断

## int指令
- 格式: `int n`, n为中断类型码. 它的功能是引发中断过程

中断过程:
1. 取出中断类型码:N
2. 标志寄存器入栈, IF=0,TF=0
3. CS,IP入栈
4. `(IP)=(n*4),(CS)=(n*4+2)`

可以在程序中使用int指令调用任何一个中断处理程序.

中断处理程序简称`中断例程`

## 编写供应用程序调用的中断例程
中断例程和子程序的写法基本一样. 

因为后面还有不少中断例程要写, 所以省略这一部分相对简单的例子.

核心步骤:
1. 安装中断例程
2. 设置中断向量表, 使中断类型码和中断例程入口对应起来
3. 运行步骤和子程序是一致的
4. 使用`int n`进行调用; 使用`iret`进行返回

## 对int,iret和栈的深入理解

问题: 用7ch中断例程完成loo指令的功能

分析: `loop s`执行需要2个信息, 循环次数和到s的位移. 可用cx存放循环次数, bx存放位移

应用举例: 在屏幕中间显示80个`!`


```s
assume cs:code

code segment
  start: mov ax,0b800h
         mov es,ax
         mov di,160*12

         mov bx,offset s-offset se      ;设置从标号se到标号s的转移位移
         mov cx,80

      s: mov byte ptr es:[di], '!'
         add di,2
         int 7ch                        ;如果(cx)≠0, 转移到标号s处
code ends

end start
```

分析: 为了模拟loop指令, 7ch中断例程应该具备以下功能
1. dec cx
2. 如果(cx)≠0, 转到标号s处执行, 否则向下执行

在中断过程中,当前标志寄存器,CS和IP都要压栈. 此时压入的CS和IP中的内容, 分别是调用程序的段地址(可以认为是标号s的段地址)和`int 7ch`后第一条指令的便偏移地址(即标号se的偏移地址)

所以在中断例程中, 可以从栈里取得标号s的段地址和标号se的偏移地址(`se的偏移地址+bx中存放的转移位移=s的偏移地址`)

3. 如何设置CS:IP
    - 前面的步骤中, 已经将栈中se的偏移地址设置成了s的偏移地址(原来栈中的IP已经变成s的偏移地址了).
    - 利用`iret`指令, 用栈中的内容设置CS,IP. 从而实现转移到标号s处.

```s
    lp: push bp
        mo bp,sp
        dec cx
        jcxz lpret
        add [bp+2],bx     ;如果(cx)=0, 则不需要修改栈中se的偏移值
 
 lpret: pop bp
        iret
```

## BIOS和DOS所提供的中断例程

在系统版的ROM中存放着一套程序, 称为BIOS(基本输入输出系统), BIOS中主要包含以下几部分内容
1. 硬件系统的检测和初始化程序
2. 外部中断和内部中断的中断例程
3. 用于对硬件设备进行I/O操作的中断例程
4. 其他和硬件系统相关的中断例程

操作系统DOS也提供了中断例程. DOS的中断例程就是操作系统向程序员提供的编程资源.

BIOS和DOS在所提供的中断例程中包含了很多子程序. 这些子程序实现了程序员在编程时经常用到的功能. 程序员可以用int指令直接调用BIOS和DOS提供的中断例程

和硬件设备相关的DOS中断例程中, 一般都调用了BIOS中的中断例程

## BIOS和DOS中断例程的安装过程
1. 开机后, CPU一加电, 初始化(CS)=0ffffh, (IP)=0, 自动从`FFFF:0`单元开始执行程序. `FFFF:0`处有一条跳转指令, CPU执行该指令后, 转去执行BIOS中的硬件系统检测和初始化程序
2. 初始化程序将建立BIOS所支持的中断向量, 即将BIOS提供的中断例程的入口地址登记在中断向量表中
    - 注意: 对于BIOS所提供的中断例程, 只需将入口地址登记在中断向量表中欧即可, 因为他们是固化到ROM中的程序, 一直在内存中存在
3. 硬件系统检测和初始化完成后, 调用`int 19h`进行操作系统的引导. 从此将计算机交由操作系统控制
4. DOS启动后,除完成其他工作外, 还将它所提供的中断例程装入内存, 并建立相应的中断向量

## BIOS中断例程应用
一般来说, 一个供程序员调用的中断例程中往往包括多个子程序, 中断例程内部用传递进来的参数决定执行哪一个子程序.
BIOS和DOS提供的中断例程, 都用`ah`来传递内部子程序的编号.

编程: 在屏幕的5行12列显示3个红底高亮闪烁绿色的'a'.

`int 10h`中断例程是BIOS提供的中断例程, 其中包含了多个和屏幕输出相关的子程序.

```s
assume cs:code
code segment
    mov ah,2          ;置光标
    mov bh,0          ;第0页ROM
    mov dh,5          ;dh中放行号
    mov dl,12         ;dl中放列号
    int 10h

    mov ah,9          ;在光标位置显示字符
    mov al,'a'        ;字符
    mov bl,11001010b  ;颜色属性
    mov bh,0          ;第0页
    mov cx,3          ;字符重复个数
    int 10h

    mov ax, 4c00h
    int 21h
code ends
end
```
## DOS中断例程应用
`int 21h`中断例程是DOS提供的中断例程, 其中包含了DOS提供给程序员在编程时调用的子程序.

编程: 在屏幕的5行12列显示字符串"Welcome to masm!"

(ah)=9表示调用第21h号中断程序的9号子程序, 功能为在光标位置显示字符串, 可以提供要显示字符串的地址作为参数

```s
assume cs:code
data segment
  db 'Welcome to masm', '$'
data ends
code segment

  start: mov ah,2         ;置光标
         mov bh,0         ;第0页
         mov dh,5         ;dh中放行号
         mov dl,12        ;dl中放列号
         int 10h

         mov ax,data
         mov ds,ax
         mov dx,0         ;ds:dx指向字符串的首地址 data:0
         mov ah,9
         int 21h

         mov ax,4c00h
         int 21h
code ends
end start
```

## 实验13 编写应用中断例程
1. 编写并安装`int 7ch`中断例程, 功能为显示一个用0结束的字符串, 中断例程安装在0:200处

参数: (dh)=行号, (dl)=列号, (cl)=颜色, ds:si指向字符串首地址

安装程序:
```s
assume cs:code

code segment
    mov ax,0
    mov es,ax
    mov di,200h          ;设置es:di指向目标地址

    mov ax,cs
    mov ds,ax
    mov si,offset do0   ;设置ds:si指向源地址

    mov cx, offset do0end-offset do0        ;设置cx为传输长度
    cld                                     ;设置传输方向为正
    rep movsb            ;第一步, 将中断例程复制到目标地址

    mov ax,0
    mov es,ax
    mov word ptr es:[7ch*4],200h
    mov word ptr es:[7ch*4+2],0h ;第二步,  设置中断向量表, 使得第7ch表项指向中断例程的入口(0:200)

    mov ax,4c00h
    int 21h

  do0:push ax
      push es
      push bx

      mov ax,0b800h
      mov es,ax
      mov al,160
      mul dh
      mov bx,ax
      mov ax,2
      mul dl
      add bx,ax             ;bx中保存初始偏移地址
show: cmp byte ptr ds:[si], 0
      je ok                 ;如果是0跳出循环
      mov al,ds:[si]
      mov ah,cl
      mov es:[bx],ax
      inc si
      add bx,2
      jmp short show

  ok: pop es
      pop ax
      pop bx
      iret

do0end: nop

code ends

end
```
应用程序:
```s
assume cs:code
data segment
  db "welcome to masm! ",0
data ends

code segment
  start: 
        mov dh,10
        mov dl,10
        mov cl,2
        mov ax,data
        mov ds,ax
        mov si,0
        int 7ch
        mov ax,4c00h
        int 21h
code ends

end start
```

2. 编写并安装`int 7ch`中断例程, 功能为完成loop指令功能
参数: (cx)=循环次数, (bx)=位移
安装程序:

```s
assume cs:code

code segment
      mov ax,0
      mov es,ax
      mov di,200h         ;1, 设置es:di指向目标地址0:200

      mov ax,cs
      mov ds,ax
      mov si,offset lp   ;2, 设置ds:si指向源地址


      mov cx, offset lpend-offset lp   ;3, 设置传输长度
      cld                 ;4, 设置传输方向为正
      rep movsb           ;5, 开始传输

      mov ax,0
      mov es,ax
      mov word ptr es:[7ch*4],200h
      mov word ptr es:[7ch*4+2],0     ;6,设置中断向量表
      
      mov ax,4c00h
      int 21h

    lp: push bp
        mov bp,sp
        dec cx
        jcxz lpret
        add [bp+2],bx     ;如果(cx)=0, 则不需要修改栈中se的偏移值

 lpret: pop bp
        iret

lpend: nop

code ends

end
```

应用程序:
```s
assume cs:code

code segment
  start: mov ax,0b800h
         mov es,ax
         mov di,160*12

         mov bx,offset s-offset se      ;设置从标号se到标号s的转移位移
         mov cx,80

      s: mov byte ptr es:[di], '!'
         add di,2
         int 7ch                        ;如果(cx)≠0, 转移到标号s处

    se:  nop
         mov ax,4c00h
         int 21h
code ends

end start
```
3. 下面的程序, 分别在屏幕的第2,4,6,8行显示4句英文诗

```s
assume cs:code

code segment
  s1: db 'Good,better,best,', '$'
  s2: db 'Nerver let it rest,', '$'
  s3: db 'Till good is better,', '$'
  s4: db 'And,better,best,', '$'
  s : dw offset s1,offset s2,offset s3,offset s4
  row: db 2,4,6,8

  start: mov ax,cs
         mov ds,ax
         mov bx,offset s
         mov si,offset row
         mov cx,4

    ok:  mov bh,0         ;第0页
         mov dh,ds:[si]   ;lh放行号
         mov dl,0         ;dl放列号
         mov ah,2         ;置光标
         int 10h

         mov dx,ds:[bx]   ;ds:dx指向字符串
         mov ah,9         ;功能号9, 表示在光标位置显示字符串
         int 21h
         
         inc si
         add bx,2
         loop ok

         mov ax,4c00h
         int 21h
code ends

end start
```
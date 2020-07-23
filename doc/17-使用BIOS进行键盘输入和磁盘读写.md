# 使用BIOS进行键盘输入和磁盘读写
键盘输入是最基本的输入, 磁盘最最常用的存储设备. BIOS为这两种外设的I/O提供了最基本的中断例程

## int9中断例程对键盘输入的处理
键盘输入将引发9号中断, BIOS提供了int9中断例程. CPU在9号中断发生后, 执行int9中断例程, 从60端口读出扫描码, 并将其转化为相应的ASCII码或状态信息, 存储在内存的指定空间(键盘缓冲区或状态字节)中

## 使用int 16h中断例程读取键盘缓冲区
int 16h中断例程的0号功能:
1. 检测键盘缓冲区中是否有数据
2. 没有则继续做第1步
3. 读取缓冲区第一个字单元中的键盘输入
4. 将读取的扫描码送入ah, ASCII码送入al
5. 将已读取的键盘输入从缓冲区中删除

BIOS的int 9中断例程和int 16h中断例程是一对相互配合的程序. int 9中断例程向键盘缓冲区中写入, int 16h中断例程从缓冲区中读出.


编程, 接收用户的键盘输入, 输入"r","g","b", 将屏幕上的字符设置为红色,绿色, 蓝色;

```s
assume cs:code
code segment
start: mov ah,0
       int 16h

       mov ah,1
       cmp al,'r'
       je red
       cmp al,'g'
       je green
       cmp al,'b'
       je blue
       jmp short sret

  red: shl ah, 1
green: shl ah, 1

blue:  mov bx,0b800h
       mov es,bx
       mov bx,1
       mov cx,2000
    s: and byte ptr es:[bx], 11111000b
       or es:[bx],ah
       add bx,2
       loop s

sret:  mov ax,4c00h
       int 21h

code ends
end start
```

## 字符串输入
最基本的字符串输入程序:
1. 在输入的同事需要显示这个字符串
2. 一般在输入回车符后,字符串输入结束
3. 能够删除已经输入的字符

分析:
1. 可以用栈的方式来管理字符串的存储空间
2. 在输入回车符后,可以在字符串中加入0, 表示字符串结束
3. 每次有新的字符输入和删除的时候, 应该重新显示字符串

过程:
1. 调用int 16h读取键盘输入
2. 如果是字符, 进入字符栈, 显示字符栈中所有字符; 继续执行1
3. 如果是退格键, 从字符栈中弹出一个字符, 显示所有字符; 继续执行1
4. 如果是enter键, 向字符栈中压入0, 返回

子程序: 字符栈的入栈,出栈,显示
参数: (ah)=功能号, 0表示入栈,1表示出栈,2表示显示
      ds:si指向字符栈空间
      0: (al)=入栈字符
      1: (al)=返回字符
      2: (dh),(dl)=字符串在屏幕上显示的行,列位置

```s
charstack: jmp short charstart

    table dw charpush,charpop,charshow
    top   dw 0                          ;栈顶

charstart: push bx
           push dx
           push di
           push es

           cmp ah,2
           ja  sret
           mov bl,ah
           mov bh,0
           add bx,bx
           jmp word ptr table[bx]

charpush:  mov bx,top
           mov [si][bx],al
           inc top
           jmp sret

charpop:   cmp top 0
           je sret
           dec top
           mov bx,top
           mov al,[si][bx]
           jmp sret

charshow:  mov bx,0b800h
           mov es,bx
           mov al,160
           mov ah,0
           mul dh
           mov di,ax
           add dl,dl
           mov dh,0
           add di,dx

           mov bx,0

charshows: cmp bx,top
           jne noempty
           mov byte ptr es:[di], ' '
           jmp sret
noempty:   mov al,[si][bx]
           mov es:[di],al
           mov byte ptr es:[di+2], ' '
           inc bx
               add di,2
           jmp charshows

sret:      pop es
           pop di
           pop dx
           pop bx
           ret
```

接收字符串输入的子程序:
```s

getstr:     push ax

getstrs:    mov ah,0
            int 16h
            cmp al,20h
            jb nochar           ;ASCII码小于20h, 说明不是字符
            mov ah,0
            call charstack      ;字符入栈
            mov ah,2
            call charstack      ;显示栈中的字符
            jmp getstrs

nochar:     cmp ah,0eh          ;退格键的扫描码
            je backspace
            cmp ah,1ch          ;enter键的扫描码
            je enter
            jmp getstrs

backspace:  mov ah,1
            call charstack      ;字符出栈
            mov ah,2
            call charstack      ;显示栈中的字符
            jmp getstrs

enter:      mov ah,0
            mov al,0
            call charstack      ;0入栈
            mov ah,2
            call charstack      ;显示栈中的字符
            pop ax
            ret
```
## 应用int 13h中断例程对磁盘进行读写

以3.5英寸软盘为例, 分为上下两面, 每面有80个磁道, 每个磁道又分为18个扇区, 每个扇区的大小为512个字节. `2面*80磁道*18扇区*512字节=1440KB`. 

只能以`扇区`为单位对磁盘进行读写. 在读写扇区的时候, 要给出面号(0开始),磁道号(0开始),扇区号(1开始).

BIOS提供是访问磁盘的中断例程是int 13h. 读取0面0道1扇区的内容到0:200的程序如下:

入口参数:
(ah)=int 13h的功能号(2 表示读扇区)
(al)=读取的扇区数
(ch)=磁道号
(cl)=扇区号
(dh)=磁头号(对于软盘即面号,因为一个面用一个磁头来读写)
(dl)=驱动器号   软驱从0开始, 0:软驱A, 1:软驱B
               硬盘从80h开始, 80h:硬盘C, 81h: 硬盘D

ex:bx指向接收从扇区读入数据的内存区

返回参数:
操作成功: (ah)=0, (al)=读入的扇区数
操作失败: (ah)=出错代码
```s
  mov ax,0
  mov es,ax
  mov bx,200h

  mov al,1    ;读取的扇区数
  mov ch,0    ;磁道号
  mov cl,1    ;扇区号
  mov dl,0    ;驱动器号
  mov dh,0    ;磁头号(面号)
  mov ah,2    ;int 13h的功能号
  int 13h
```

将0:200中的内容写入0面0道1扇区

入口参数:
(ah)=int 13h的功能号(3 表示写扇区)
(al)=写入的扇区数
(ch)=磁道号
(cl)=扇区号
(dh)=磁头号(面)
(dl)=驱动器号
ex:bx指向将写入的磁盘的数据

返回参数:
操作成功: (ah)=0, (al)=写入的扇区数
操作失败: (ah)=出错代码
```s
mov ax,0
mov ex,ax
mov bx,200h

mov al,1
mov ch,0
mov cl,1
mov dh,0
mov dl,0

mov ah,3
int 13h
```

编程: 将当前屏幕的内容保存在磁盘上

分析: 1屏的内容占4000个字节, 需要8个扇区, 用0面0道的1-8扇区存储显存中的内容

```s
assume cs:code
code segment
start: 
      mov ax,0b800h
      mov ex,ax
      mov bx,0

      mov al,8
      mov ch,0
      mov cl,1
      mov dh,0
      mov dl,0

      mov ah,3
      int 13h

      mov ax,4c00h
      int 21h
code ends
end start
```
## 实验17 编写包含多个功能子程序的中断例程

用面号, 磁道号, 扇区号来访问磁盘不太方便. 可以考虑对位于不同磁道,面上的所有扇区进行统一编号. 编号从0开始,一直到2879,我们称这个编号为逻辑扇区编号

`逻辑扇区编号 = (面号*80 + 磁道号)*18 + 扇区号-1`

根据逻辑扇区号算出物理编号:
- int(): 描述性运算符, 取商
- rem(): 描述性运算符, 取余数
- 逻辑扇区号=(面号*80 + 磁道号)*18 + 扇区号-1
- 面号=int(逻辑扇区号/1440)
- 磁道号=int(rem(逻辑扇区号/1440)/18)
- 扇区号=rem(rem(逻辑扇区号/1440)/18)+1

安装一个新的int 7ch中断例程, 实现通过逻辑扇区号对软盘进行读写

参数说明:
1. 用ah寄存器传递功能号: 0表示读, 1表示写
2. 用dx寄存器传递要读写的扇区的逻辑扇区号
3. 用es:bx指向存储读出数据或写入数据的内存区

分析: 用逻辑扇区计算出面号, 磁道号, 扇区号后, 调用int 13h中断例程进行实际的读写
(ah)=int 13h的功能号(3 表示写扇区)
(al)=写入的扇区数
(ch)=磁道号
(cl)=扇区号
(dh)=磁头号(面)
(dl)=驱动器号

子程序:
```s
assume cs:code

stack segment
  db 128 dup (0)
stack ends
code segment

start:   mov ax,stack
         mov ss,ax
         mov sp,128

         push cs
         pop ds

         mov ax,0
         mov es,ax

         mov si,offset logical_sector                 ;设置ds:si指向源地址
         mov di,204h                                  ;设置es:di指向目标地址
         mov cx,offset logical_sector_end-offset logical_sector  ;设置cx为传输长度
         cld                                          ;设置传输方向为正
         rep movsb

         push es:[7ch*4]
         pop es:[200h]
         push es:[7ch*4+2]
         pop es:[202h]      ;暂存BIOS原有的int7c中断例程地址

         cli
         mov word ptr es:[7ch*4],204h
         mov word ptr es:[7ch*4+2],0  ;设置新的int7c中断例程的地址0:204h
         sti    

         mov ax,4c00h
         int 21h

logical_sector:
    jmp short run
    store db dup(16) 0
    table db 2,3      ;偏移地址表

run:
    cmp ah,1
    ja ok             ;>1则退出

    push ah
    push dx
    push bx
    mov ax,dx
    mov dx,0
    mov bx,1440
    div bx      ;(ax)=面号, (dx)=余数
    mov store[0], al
    mov ax,dx
    mov bl,18
    div bl      ;(al)=磁道号, (ah)=余数
    mov store[1], al
    add ah,1    ;(ah)=扇区号
    mov store[2], ah

    mov al,1
    mov ch,store[1]
    mov cl,store[2]
    mov dh,store[0]
    mov dl,0

    mov bh,0
    mov bl,ah         ;将ah的值转移到bx中
    mov ah,table[bx]  ;设置
    pop bx
    int 13h

    pop dx
    pop ah
ok: iret
logical_sector_end: nop
code ends
end
```
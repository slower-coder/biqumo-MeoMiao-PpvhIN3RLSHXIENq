
# 花指令及反混淆


## 1\.花指令


  花指令是反调试的一种基本的方法。其存在是干扰选手静态分析，但不会影响程序的运行。实质就是一串垃圾指令，它与程序本身的功能无关，并不影响程序本身的逻辑。在软件保护中，花指令被作为一种手段来增加静态分析的难度。IDA并不能正常识别花指令，导致可看可分析代码被破坏，因此需要我们自己去分析一下。花指令主要分为两类：可执行花指令和不可执行花指令。


* 可执行花指令：花指令在程序正常运行的时候被执行，但不会影响程序正常运行
* 不可执行花指令：花指令在程序正常运行的时候不会被执行


**常见混淆的字节码：**




| 机器码 | 汇编语言 |
| --- | --- |
| **9A** | **CALL immed32** |
| **E8** | **CALL immed16** |
| **E9** | **JMP immed16** |
| **EB** | **JMP immed8** |


## 2\.常见花指令分析


### （1）单字节



```
#include 
int main()
{
    __asm {
        jz start; //jz 和 jnz 同时出现，导致永恒跳转
        jnz start;
        _emit 0xE8; //这个地方就是故意插入的花指令 CALL + 地址
    }
start:
    printf("ok!");
    return 0;
}

```


**例如：\[NSSRound\#3 Team]jump\_by\_jump**


发现main函数编译不了 向下观察发现花指令


![](https://img2024.cnblogs.com/blog/3540252/202412/3540252-20241210204706813-569267570.png)


去除花指令 按D快捷键 先将call 转成硬编码 E8


![](https://img2024.cnblogs.com/blog/3540252/202412/3540252-20241210204721995-1510331260.png)


再将光标放到 db 0E8上 将E8改成 nop(90\) 再次按C键 点yes 将硬编码修复成代码


![](https://img2024.cnblogs.com/blog/3540252/202412/3540252-20241210204730819-636519689.png)


然后向下逐⼀修复 将光标放置在⻩⾊⾏上 按C修复 直到没有⻩⾊地址


![](https://img2024.cnblogs.com/blog/3540252/202412/3540252-20241210204736647-1794898615.png)


将光标放置到函数开始的位置按 P键⽣成函数 最后tab转成伪代码


![](https://img2024.cnblogs.com/blog/3540252/202412/3540252-20241210204851231-1493226536.png)


### （2）永恒跳转



```
int main()
{
    __asm {
    xor eax, eax;// eax ^ eax = 0 
    jz s; // 必然成立跳转，一定会跳转
    _emit 0x11; //填充垃圾指令 byte类型
    _emit 0x22; //填充垃圾指令 byte类型
    _emit 0x33; //填充垃圾指令 byte类型
    s:
    }
	printf("test \n");
}

```

### （3）更改ESP



```
int main()
{
    __asm {
        xor eax, eax;
        jz s;
        add esp, 0x11; // IDA 会把此指令识别对函数堆栈进行操作，导致识别函数失败
    s:
    }
    printf("test \n");
}

```

### （4）jmp跳转插入无效垃圾指令



```
#define _CRT_SECURE_NO_WARNINGS
#include 
int main()
{
    _asm {
    
        jmp $+5
        _emit 0x71
        _emit 2
        _emit 0xE9
        _emit 0xED
   }
 lable:
    printf("ok2");

    return 0;
}

```

### （5）嵌套永恒跳转



```
#define _CRT_SECURE_NO_WARNINGS
#include 
int main()
{
    _asm {
        jz Label3;
        jnz Label3;
        _emit 0xE8;
    }
Label2:
    _asm {
        jz Label4;
        jnz Label4;
        _emit 0xE8;
    }


Label3:
    _asm {
        jz Label1;
        jnz Label1;
        _emit 0xE9;
    }
Label1:
    _asm {
        jz Label2;
        jnz Label2;
        _emit 0xE9;
    }
Label4:
    printf("ok2");

    return 0;
}

```



## 3\.反混淆


### 方法一：手动恢复


**适用条件：混淆少且类型单一**
  为了尽快解除题目，我们一般选择先手动去除混淆，迅速拿到flag
这里需要掌握IDA的基本快捷键：U、C、P


1. U: 在IDA Pro中，按“U”重新定义汇编，转换为字节码格式
2. C: 在IDA Pro中按“C”，转换字节码为汇编形式
3. P: 在IDA Pro中，按“P” 转换汇编语言为高级语言函数视图


### 方法二：IDA\-Python脚本恢复


**适用条件：混淆大量，手工基本不可去除**
  需要拿到混淆的字节码组成，利用脚本去除大量混淆。


**例如：\[GFCTF 2021]wordy**


![](https://img2024.cnblogs.com/blog/3540252/202412/3540252-20241210204922547-1833814177.png)


出现大量机器码为EBFF的花指令，如果手动nop很费，所以这里用idapython



```
import idc
import ida_bytes

start_add=0x1144
end_add=0x3100
for address in range(start_add, end_add):
  new_byte = ida_bytes.get_byte(address)
  next = ida_bytes.get_byte(address + 1)
  nnext = ida_bytes.get_byte(address + 2)
  if new_byte == 0xeb and next == 0xff and nnext == 0xc0:  
    ida_bytes.patch_byte(address, 0x90)

```

  首先循环遍历从 `0x1144` 到 `0x3100` 的地址。在每个地址处，检查当前字节是否是 `0xeb`，下一个字节是否是 `0xff`，再下一个字节是否是 `0xc0`。如果匹配到 `0xeb 0xff 0xc0` 这个字节序列，就将当前地址处的字节 `0xeb` 修改为 `0x90`。`0x90` 在汇编语言中是 NOP 指令，表示“无操作”，即这个指令不会对程序执行产生影响。


 本博客参考[楚门加速器官网](https://chuanggeye.com)。转载请注明出处！

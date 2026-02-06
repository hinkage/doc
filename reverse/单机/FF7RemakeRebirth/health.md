# 2026-01-30
## 这次尝试先自己找

进入游戏，触发战斗，挨打掉血，按ESC暂停再搜索血量数值，最后只剩下一个候选地址。

用内存访问断点监视，得到：
```
ff7rebirth_.exe+89EC0C:
7FF61841EC02 - 44 88 7D 17  - mov [rbp+17],r15b
7FF61841EC06 - 89 B0 A8030000  - mov [rax+000003A8],esi
7FF61841EC0C - 89 77 10  - mov [rdi+10],esi <<
7FF61841EC0F - 48 8B 4D 1F  - mov rcx,[rbp+1F]
7FF61841EC13 - 48 85 C9  - test rcx,rcx

RAX=00000213EB880088
RBX=00000213EB880088
RCX=00000213EB880000
RDX=0000000000000000
RSI=0000000000000D76
RDI=00007FF61EC41F60
RBP=00000098B797CF09
RSP=00000098B797CE40
R8=0000000000000000
R9=00007FF61EC41F40
R10=00007FF61EC41F00
R11=00000213EB880000
R12=00007FF09CC57C00
R13=00007FF05C4FDB20
R14=00000098B797E000
R15=0000000000000000
RIP=00007FF61841EC0F
```
直接修改这个地址的数值，生命值并没有变化。

所以猜测rdi+10不是角色生命值，而是UI数值。

那么它的上一行，rax+3a8就大概率是角色地址+生命值偏移。
```
>>> hex(0x213EB880088+0x3a8)
'0x213eb880430'
```
修改这个地址的数值，果然生命值就变化了。

在主模块搜索：
```
89 B0 A8 03 00 00 89 77 10 48 8B 4D 1F 48 85 C9
```
只有一个结果，可以作为特征码。

## 结果
```
[ENABLE]

aobscanmodule(aobhealth,ff7rebirth_.exe,89 B0 A8 03 00 00 89 77 10 48 8B 4D 1F 48 85 C9)
alloc(newmem,0x1000,aobhealth)
label(return)
label(health)

registersymbol(health)

newmem:
  mov esi,[health]              // 强制血量
  mov [rax+000003A8],esi        // 写角色HP
  jmp return

newmem+200:
health:
  dd 2111

aobhealth:
  jmp newmem
  nop                           // 补足 6 字节

return:

[DISABLE]

aobhealth:
  db 89 B0 A8 03 00 00           // 原指令1
dealloc(newmem)
```
测试发现，血量会对主角和队友生效，怪物血量无影响，可能不需要进一步过滤了。

## 再来看看同行的
```
  v1 = "\n"
       "[ENABLE]\n"
       "aobscanmodule(aobhpmp,ff7rebirth_.exe,E8 * * * * * 85 * 74 * 44 8B 90 s1.2.*.0x300.0x500 00 00)\n"
       "alloc(newmem,$1000,aobhpmp)\n"
       "label(code)\n"
       "label(return)\n"
       "label(hp mp)\n"
       "registersymbol(hp mp)\n"
       "\n"
       "newmem:\n"
       "  lea r10,[rax+s1]\n"
       "  cmp [hp],1\n"
       "  jne @f\n"
       "  mov [r10],#99999\n"
       "@@:\n"
       "  cmp [mp],1\n"
       "  jne @f\n"
       "  mov [r10+04],#9999\n"
       "code:\n"
       "  mov r10d,[r10]\n"
       "  jmp return\n"
       "\n"
       "newmem+200:\n"
       "hp:\n"
       "dd 0\n"
       "mp:\n"
       "dd 0\n"
       "\n"
       "aobhpmp+A:\n"
       "  jmp newmem\n"
       "  nop 2\n"
       "return:\n"
       "registersymbol(aobhpmp)\n"
       "\n"
       "[DISABLE]\n"
       "aobhpmp+A:\n"
       "  db 44 8B 90 s1 00 00\n"
       "dealloc(newmem)\n";
```
搜索E8 * * * * * 85 * 74 * 44 8B 90得到：
```
ff7rebirth_.exe+AC3D4A - E8 A9000000           - call ff7rebirth_.exe+AC3DF8
ff7rebirth_.exe+AC3D4F - 48 85 C0              - test rax,rax
ff7rebirth_.exe+AC3D52 - 74 07                 - je ff7rebirth_.exe+AC3D5B
ff7rebirth_.exe+AC3D54 - 44 8B 90 A8030000     - mov r10d,[rax+000003A8]
ff7rebirth_.exe+AC3D5B - 48 8B 5C 24 30        - mov rbx,[rsp+30]
ff7rebirth_.exe+AC3D60 - 41 8B C2              - mov eax,r10d
ff7rebirth_.exe+AC3D63 - 48 83 C4 20           - add rsp,20
ff7rebirth_.exe+AC3D67 - 5F                    - pop rdi
ff7rebirth_.exe+AC3D68 - C3                    - ret 
```
是一个返回角色生命值的函数。

同行的特征码是动态获取3a8这个偏移，应该更能跨版本。

## 魔法值
数值搜索过滤两次即可唯一确定一个地址。

从访问断点得到
```
ff7rebirth_.exe+89E5DC:
7FF61841E5D2 - 44 88 7D 17  - mov [rbp+17],r15b
7FF61841E5D6 - 89 BB AC030000  - mov [rbx+000003AC],edi
7FF61841E5DC - 89 78 18  - mov [rax+18],edi <<
7FF61841E5DF - 48 8B 4D 1F  - mov rcx,[rbp+1F]
7FF61841E5E3 - 48 85 C9  - test rcx,rcx
```
可见3ac就是魔法值偏移了。


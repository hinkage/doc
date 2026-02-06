# 2026-01-30
## 先看同行的实现

```cpp
int sub_140001F30()
{
  __int64 v0; // rax
  const char *v1; // rcx
  __int64 v2; // rdx
  __int64 v3; // r8
  __int128 v4; // xmm0

  v0 = sub_140028F20(512);
  v1 = "\n"
       "[ENABLE]\n"
       "aobscanmodule(aobmoney,ff7rebirth_.exe,48 85 * 74 * 8B 5E 0C 2B D8 8B)\n"
       "alloc(newmem,$1000,aobmoney)\n"
       "label(code)\n"
       "label(return)\n"
       "label(money)\n"
       "registersymbol(money)\n"
       "\n"
       "newmem:\n"
       "  mov ebx,[money]\n"
       "  cmp ebx,0\n"
       "  jle code\n"
       "  cmp byte ptr [rsi+02],8\n"
       "  jne code\n"
       "  cmp [rsi+08],1\n"
       "  jne code\n"
       "  mov [rsi+0C],ebx\n"
       "code:\n"
       "  mov ebx,[rsi+0C]\n"
       "  sub ebx,eax\n"
       "  jmp return\n"
       "\n"
       "newmem+200:\n"
       "money:\n"
       "dd 0\n"
       "\n"
       "aobmoney+5:\n"
       "  jmp newmem\n"
       "return:\n"
       "registersymbol(aobmoney)\n"
       "\n"
       "[DISABLE]\n"
       "aobmoney+5:\n"
       "  db 8B 5E 0C 2B D8\n"
       "dealloc(newmem)\n";
  xmmword_14011BCB0 = (__int128)_mm_load_si128((const __m128i *)&xmmword_140103B80);
  qword_14011BCA0 = v0;
  v2 = v0;
  v3 = 3;
  do
  {
    v2 += 128;
    v4 = *(_OWORD *)v1;
    v1 += 128;
    *(_OWORD *)(v2 - 128) = v4;
    *(_OWORD *)(v2 - 112) = *((_OWORD *)v1 - 7);
    *(_OWORD *)(v2 - 96) = *((_OWORD *)v1 - 6);
    *(_OWORD *)(v2 - 80) = *((_OWORD *)v1 - 5);
    *(_OWORD *)(v2 - 64) = *((_OWORD *)v1 - 4);
    *(_OWORD *)(v2 - 48) = *((_OWORD *)v1 - 3);
    *(_OWORD *)(v2 - 32) = *((_OWORD *)v1 - 2);
    *(_OWORD *)(v2 - 16) = *((_OWORD *)v1 - 1);
    --v3;
  }
  while ( v3 );
  *(_OWORD *)v2 = *(_OWORD *)v1;
  *(_OWORD *)(v2 + 16) = *((_OWORD *)v1 + 1);
  *(_OWORD *)(v2 + 32) = *((_OWORD *)v1 + 2);
  *(_OWORD *)(v2 + 48) = *((_OWORD *)v1 + 3);
  *(_OWORD *)(v2 + 64) = *((_OWORD *)v1 + 4);
  *(_OWORD *)(v2 + 80) = *((_OWORD *)v1 + 5);
  *(_OWORD *)(v2 + 96) = *((_OWORD *)v1 + 6);
  *(_DWORD *)(v2 + 112) = *((_DWORD *)v1 + 28);
  *(_BYTE *)(v0 + 500) = 0;
  return atexit(sub_1400CEE30);
}
```

用同行的特征码在主模块进行搜索，
```
48 85 ?? 74 ?? 8B 5E 0C 2B D8 8B
```
得到
```
aobmoney - 48 85 F6              - test rsi,rsi
ff7rebirth_.exe+BE5B5D- 74 05                 - je ff7rebirth_.exe+BE5B64
ff7rebirth_.exe+BE5B5F- 8B 5E 0C              - mov ebx,[rsi+0C]
ff7rebirth_.exe+BE5B62- 2B D8                 - sub ebx,eax
ff7rebirth_.exe+BE5B64- 8B C3                 - mov eax,ebx
```
可见rsi+0C就是金钱数量。

## 再自己验证
进入商店，买卖物品，让金钱数值变化。
用CE搜索定位到两个候选地址。
选择这个主模块+偏移的地址。

```
ff7rebirth_.exe+7111AE4
```

用内存访问断点监视，只有一个结果。
```
ff7rebirth_.exe+BE5B5F:
7FF618765B5A - 48 85 F6  - test rsi,rsi
7FF618765B5D - 74 05 - je ff7rebirth_.exe+BE5B64
7FF618765B5F - 8B 5E 0C  - mov ebx,[rsi+0C] <<
7FF618765B62 - 2B D8  - sub ebx,eax
7FF618765B64 - 8B C3  - mov eax,ebx

RAX=0000000000000000
RBX=0000000000090EED
RCX=00007FEFBC3CBF40
RDX=00000000000186AA
RSI=00007FF61EC91AD8
RDI=00007FEFBC3CBF40
RBP=00000098B797F740
RSP=00000098B797F610
R8=0000000000000064
R9=0000000000000064
R10=0000000000000000
R11=00007FEFBC3CBF40
R12=0000000000000001
R13=0000000000000001
R14=0000000000000004
R15=0000000000000004
RIP=00007FF618765B62

First seen:22:08:24
Last seen:22:08:29
```
这个结果，恰好也就是同行代码里那个特征码搜索的结果。

## 直接抄同行代码，问题解决
```
[ENABLE]

aobscanmodule(aobmoney,ff7rebirth_.exe,48 85 ?? 74 ?? 8B 5E 0C 2B D8 8B)
alloc(newmem,$1000,aobmoney)
label(code)
label(return)
label(money)
registersymbol(money)

newmem:
  mov ebx,[money]
  cmp ebx,0
  jle code
  cmp byte ptr [rsi+02],8
  jne code
  cmp [rsi+08],1
  jne code
  mov [rsi+0C],ebx
code:
  mov ebx,[rsi+0C]
  sub ebx,eax
  jmp return

newmem+200:
money:
  dd 99999

aobmoney+5:
  jmp newmem
return:
registersymbol(aobmoney)

[DISABLE]

aobmoney+5:
  db 8B 5E 0C 2B D8
dealloc(newmem)
```
同行代码对rsi+08和rsi+0C做了检查，但是游戏源码都没做这个检查，有啥用呢？


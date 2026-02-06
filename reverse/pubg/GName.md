# GName 即 FNamePool
```c
FString FSoundSource::Describe(bool bUseLongName)
{
	return FString::Printf(TEXT("Wave: %s, Volume: %6.2f, Owner: %s"),
		bUseLongName ? *WaveInstance->WaveData->GetPathName() : *WaveInstance->WaveData->GetName(),
		WaveInstance->GetVolume(),
		WaveInstance->ActiveSound ? *WaveInstance->ActiveSound->GetOwnerName() : TEXT("None"));
}
```
这个函数中的字符串未被隐藏。

从这里找到GetPathName。

```c
/**
 * Version of GetPathName() that eliminates unnecessary copies and appends an existing string.
 */
void UObjectBaseUtility::GetPathName(const UObject* StopOuter, FString& ResultString) const;
```
从GetPathName找到解密ClassPrivate, OuterPrivate, NamePrivate的代码, 还可以找到FNamePool。

```c
void FName::AppendString(FString& Out) const
{
	const FNameEntry* const NameEntry = GetDisplayNameEntry();

	NameEntry->AppendNameToString( Out );
	if (GetNumber() != NAME_NO_NUMBER_INTERNAL)
	{
		Out += TEXT('_');
		Out.AppendInt(NAME_INTERNAL_TO_EXTERNAL(GetNumber()));
	}
}
```
可以用Unicorn直接调用这个AppendString得到字符串。

```c
const FNameEntry* FName::GetDisplayNameEntry() const
{
	return &GetNamePool().Resolve(GetDisplayIndex());
}

_QWORD *__fastcall fn_FName::GetDisplayNameEntry_7FF6BCC03940(int *a1, _QWORD *a2)
{
  int v4; // esi
  _QWORD *v5; // rcx
  __int64 v6; // rdx
  __int64 v8_FNamePool; // [rsp+30h] [rbp+8h] BYREF

  fn_GetNamePool_7FF6BEECB6F0(&v8_FNamePool);
  v4 = *a1;
  v5 = (_QWORD *)MEMORY[0x7FF4E3632132](1212073531, v8_FNamePool);
  v6 = *(_QWORD *)(*(_QWORD *)(MEMORY[0x7FF4E3632132](1212073559, *v5) + 8LL * (v4 / 0x40D8)) + 8LL * (v4 % 0x40D8));
  *a2 = MEMORY[0x7FF4E363210A](1212073890, v6);
  return a2;
}

  v11 = (_QWORD *)MEMORY[0x7FF4E3632132](1212073948, v6);
  v12 = *(_QWORD *)MEMORY[0x7FF4E3632132](1212073975, *v11);
  v13 = MEMORY[0x7FF4E3632132](1212073486, v12);
  *a1 = MEMORY[0x7FF4E363210A](1212073531, v13);
```
注意到FNamePool解密是3层。

实际上Display的FNamePool是宽字符串，获取字符串的代码更复杂，只能用Unicorn仿真call得到结果。

UC用的是另一个窄字符串的FNamePool，更容易获取字符串。

```c
_QWORD *__fastcall sub_7FF7AB7D00C0(_QWORD *a1)
{
  _DWORD *v2; // rbx
  _QWORD *v3; // rax
  _QWORD *v4; // rax
  _QWORD *v5; // rax
  _QWORD *v6; // rax
  __int64 v7; // rcx

  TslGame_Win64_Shipping_291();
  TslGame_Win64_Shipping_291();
  TslGame_Win64_Shipping_291();
  TslGame_Win64_Shipping_291();
  TslGame_Win64_Shipping_291();
  TslGame_Win64_Shipping_291();
  TslGame_Win64_Shipping_291();
  TslGame_Win64_Shipping_291();
  TslGame_Win64_Shipping_291();
  TslGame_Win64_Shipping_291();
  v2 = (_DWORD *)(*(_QWORD *)NtCurrentTeb()->ThreadLocalStoragePointer + 0xC0LL);
  if ( dword_7FF7B9FE12F8 > *v2 )
  {
    sub_7FF7AB3FDDA8(&dword_7FF7B9FE12F8);
    if ( dword_7FF7B9FE12F8 == -1 )
    {
      g_FNamePool_7FF7B9FE1300 = MEMORY[0x7FF48EDD1AC4](1212075505, 0);// 窄字符串FNamePool
      sub_7FF7AB3FDD3C(&dword_7FF7B9FE12F8);
    }
  }
  if ( dword_7FF7B9FE1308 > *v2 )
  {
    sub_7FF7AB3FDDA8(&dword_7FF7B9FE1308);
    if ( dword_7FF7B9FE1308 == -1 )
    {
      qword_7FF7B9FE1310 = MEMORY[0x7FF48EDD1AC4](1212075478, &g_FNamePool_7FF7B9FE1300);
      sub_7FF7AB3FDD3C(&dword_7FF7B9FE1308);
    }
  }
  if ( dword_7FF7B9FE1318 > *v2 )
  {
    sub_7FF7AB3FDDA8(&dword_7FF7B9FE1318);
    if ( dword_7FF7B9FE1318 == -1 )
    {
      qword_7FF7B9FE1320 = MEMORY[0x7FF48EDD1AC4](1212075455, &qword_7FF7B9FE1310);
      sub_7FF7AB3FDD3C(&dword_7FF7B9FE1318);
    }
  }
  v3 = (_QWORD *)MEMORY[0x7FF48EDD1AEE](1212075455, qword_7FF7B9FE1320);
  v4 = (_QWORD *)MEMORY[0x7FF48EDD1AEE](1212075478, *v3);
  if ( !MEMORY[0x7FF48EDD1AEE](1212075505, *v4) )
    JUMPOUT(0x7FF7AB7D026ELL);
  v5 = (_QWORD *)MEMORY[0x7FF48EDD1AEE](1212075455, qword_7FF7B9FE1320);
  v6 = (_QWORD *)MEMORY[0x7FF48EDD1AEE](1212075478, *v5);
  v7 = MEMORY[0x7FF48EDD1AEE](1212075505, *v6);
  *a1 = MEMORY[0x7FF48EDD1AC4](1212075034, v7);
  return a1;
}
```

```c
static uint64_t m_address = 0;
/*
  fn_GetNamePool_7FF6BEECB6F0(&v8_FNamePool);
  v4 = *a1;
  v5 = (_QWORD *)MEMORY[0x7FF4E3632132](1212073531, v8_FNamePool);
  v6 = *(_QWORD *)(*(_QWORD *)(MEMORY[0x7FF4E3632132](1212073559, *v5) + 8LL * (v4 / 0x40D8)) + 8LL * (v4 % 0x40D8));
*/
void FName_init() {
    // auto tmp = g_memory.read8b(g_memory.get_base() + offsets.GName);
    // tmp = local_decrypt.decrypt_pointer(tmp);
    // tmp = g_memory.read8b(tmp);
    // tmp = local_decrypt.decrypt_pointer(tmp);
    // tmp = g_memory.read8b(tmp);
    // tmp = local_decrypt.decrypt_pointer(tmp);
    // g_FNamePool = tmp;
    // LOG("g_FNamePool: %llx", tmp);
    // 这两个FNamePool指针不同，但是都可以用
    // 上面那个是display字符串，是宽字符。下面这个是窄字符。
    {
        auto tmp = local_decrypt.decrypt_pointer(g_memory.read<uint64_t>(g_memory.get_base() + offsets.GNames));
        LOG("FName_init: tmp=%llx", tmp);
        ulib::reverse::runtime_print_hex(tmp, 0x20, tmp);
        for (uint32_t off = 0; off < 0x30; off += 8) {
            auto addr = local_decrypt.decrypt_pointer(g_memory.read<uint64_t>(tmp + off));
            if (addr > 0x10000000000) {
                m_address = addr;
                LOG("FName_init: off: %x, %llx", off, m_address);
                break;
            }
        }
        // m_address = local_decrypt.decrypt_pointer(g_memory.read<uint64_t>(tmp + offsets.GNames_off));
        LOG("FName_init: %llx", m_address);
    }
}

void FName_to_string_local(std::string *out, FName &fname) {
    // LOG("> FName_idx_to_string: %d", idx_name);
    auto it = cache_map.find(fname);
    if (it != cache_map.end()) {
        // LOG("gnames: cache hit");
        *out = it->second;
        return;
    }
    auto idx_name = fname.ComparisonIndex;
    auto p_chunk = g_memory.read<uint64_t>(m_address + (idx_name / offsets.ElementsPerChunk) * 0x8);
    if (!p_chunk) {
        return;
    }
    auto p_name = g_memory.read<uint64_t>(p_chunk + (idx_name % offsets.ElementsPerChunk) * 0x8);
    if (!p_name) {
        return;
    }
    // UE4限制字符串长度最大为1024
    char buffer[1024]{0};
    g_memory.read_bytes(p_name + 0x10, buffer, sizeof(buffer) - 0x1);
    buffer[sizeof(buffer) - 0x1] = '\0';
    *out = std::string(buffer);
    // 带上Number会导致字符串相等判断很复杂，所以不用
    // if (fname.Number != 0) {
    //     *out += "_" + std::to_string(fname.Number);
    // }
    // LOG("FName_to_string_local: %d => %s", fname.ComparisonIndex, out->c_str());
    cache_map[fname] = *out;
}

void FName_to_string(std::string *out, FName &fname) {
    FName_to_string_local(out, fname);
    return;
    //
    auto it = cache_map.find(fname);
    if (it != cache_map.end()) {
        // LOG("gnames: cache hit");
        *out = it->second;
        return;
    }
    // Unicorn模拟call AppendString得到宽字符串，再转为窄字符
    auto wstr = g_ec0.call_AppendString(fname);
    *out = util::wstring2string(wstr);
    cache_map[fname] = *out;
}
```
### 仿真
```c
std::wstring PUBG_ExternalCall::call_AppendString(FName &fname) {
    // LOG("call_AppendString");
    // m_debug = true;
    auto fn_AppendString = g_memory.get_base() + offsets.fn_AppendString;
    // LOG("fn_AppendString: %llx", fn_AppendString);
    // reverse::runtime_print_disassemble(fn_AppendString, 0x100);
    // getchar();
    m_ctx.entry_va = fn_AppendString;
    m_ctx.exit_va = FakeRet;
    m_ctx.breakpoints = {
        // 0x9D63,  //
    };
    auto rsp = reset();
    uint64_t arg0 = rsp + 0x100;
    uc_mem_write(m_uc, arg0, &fname, 8);
    uc_reg_write(m_uc, UC_X86_REG_RCX, &arg0);
    // 传参FString，实际上就是一个TArray
    // capacity设为UE4字符串最大字节数2048，就不会触发malloc，不会触发系统调用
    TArray tarr{.m_address = rsp + 0x300, .m_count = 0, .m_max = 2048};
    uint64_t arg1 = rsp + 0x200;
    uc_mem_write(m_uc, arg1, &tarr, sizeof(TArray));
    uc_reg_write(m_uc, UC_X86_REG_RDX, &arg1);
    do_uc_emu_start(rsp);
    // 读取处理后的字符串
    uc_mem_read(m_uc, arg1, &tarr, sizeof(TArray));
    // LOG("call_AppendString: %llx, %u, %u", tarr.m_address, tarr.m_count, tarr.m_max);
    std::wstring out;
    // 注意FString的长度就是宽字符个数，不是字节数，这里不乘以2
    out.resize(tarr.m_count);
    // uc_mem_read的参数是字节数，要乘以2
    uc_mem_read(m_uc, tarr.m_address, out.data(), tarr.m_count * 2);
    // PUBG已经改为全用wchar_t了？
    // 实际上这是读的Display字符串，原始的仍然是Ansi字符串
    // LOG("out: %ws", out.c_str());
    // getchar();
    // m_debug = false;
    return out;
}
```

## 自动扫描
```cpp
{
    uint64_t p = search_signature_in_buffer(g_patterns.GNames);
    uint64_t val = 0;
    g_dis.print(p, va_to_raw(p), 0x100, [&val](uint64_t addr, ZydisDecodedInstruction &instr, ZydisDecodedOperand *ops) {
        if (instr.mnemonic != ZYDIS_MNEMONIC_MOV) return false;
        if (ops[1].reg.value != ZYDIS_REGISTER_RAX) {
            return false;
        }
        auto &dest = ops[0];
        if (!dest.mem.disp.has_displacement) return false;
        // mov     cs:qword_7FF7D7FC9D60, rax
        val = addr + dest.mem.disp.value;
        return true;
    });
    LOG("GNames=%llx", val);
    of::add_one_offset("GNames", val - m_pe.image_base);
}
```


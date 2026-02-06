# GUObjectArray
## 2026-01-16
Engine/Source/Runtime/CoreUObject/Public/UObject/UObjectArray.h

```c
/**
* Simple array type that can be expanded without invalidating existing entries.
* This is critical to thread safe FNames.
* @param ElementType Type of the pointer we are storing in the array
* @param MaxTotalElements absolute maximum number of elements this array can ever hold
* @param ElementsPerChunk how many elements to allocate in a chunk
**/
class FChunkedFixedUObjectArray
{
	enum
	{
		NumElementsPerChunk = 64 * 1024,
	};
	/** Master table to chunks of pointers **/
	FUObjectItem** Objects;
}

/***
*
* FUObjectArray replaces the functionality of GObjObjects and UObject::Index
*
* Note the layout of this data structure is mostly to emulate the old behavior and minimize code rework during code restructure.
* Better data structures could be used in the future, for example maybe all that is needed is a TSet<UObject *>
* One has to be a little careful with this, especially with the GC optimization. I have seen spots that assume
* that non-GC objects come before GC ones during iteration.
*
**/
class COREUOBJECT_API FUObjectArray
{
private:
	//typedef TStaticIndirectArrayThreadSafeRead<UObjectBase, 8 * 1024 * 1024 /* Max 8M UObjects */, 16384 /* allocated in 64K/128K chunks */ > TUObjectArray;
	typedef FChunkedFixedUObjectArray TUObjectArray;

	/** Array of all live objects.											*/
	TUObjectArray ObjObjects;

public:

	/** INTERNAL USE ONLY: gets the internal FUObjectItem array */
	TUObjectArray& GetObjectItemArrayUnsafe()
	{
		return ObjObjects;
	}
};

/**
* Single item in the UObject array.
*/
struct FUObjectItem
{
	// Pointer to the allocated object
	class UObjectBase* Object;
}

// DeltaForce用的Chunked数组，下面这个就是分块数组
/** Global UObject allocator							*/
extern COREUOBJECT_API FUObjectArray GUObjectArray;

// PUBG用的不是Chunked数组，是一个线性数组，直接用下标访问的

/**
 * Final phase of UObject initialization. all auto register objects are added to the main data structures.
 */
void UObjectBaseInit()
{
	SCOPED_BOOT_TIMING("UObjectBaseInit");

	// Zero initialize and later on get value from .ini so it is overridable per game/ platform...
	int32 MaxObjectsNotConsideredByGC = 0;
	int32 SizeOfPermanentObjectPool = 0;
	int32 MaxUObjects = 2 * 1024 * 1024; // Default to ~2M UObjects
	bool bPreAllocateUObjectArray = false;	

	// To properly set MaxObjectsNotConsideredByGC look for "Log: XXX objects as part of root set at end of initial load."
	// in your log file. This is being logged from LaunchEnglineLoop after objects have been added to the root set. 

	// Disregard for GC relies on seekfree loading for interaction with linkers. We also don't want to use it in the Editor, for which
	// FPlatformProperties::RequiresCookedData() will be false. Please note that GIsEditor and FApp::IsGame() are not valid at this point.
	if (FPlatformProperties::RequiresCookedData())
	{
		FString Value;
		bool bIsCookOnTheFly = FParse::Value(FCommandLine::Get(), TEXT("-filehostip="), Value);
		if (bIsCookOnTheFly)
		{
			extern int32 GCreateGCClusters;
			GCreateGCClusters = false;
		}
		else
		{
			GConfig->GetInt(TEXT("/Script/Engine.GarbageCollectionSettings"), TEXT("gc.MaxObjectsNotConsideredByGC"), MaxObjectsNotConsideredByGC, GEngineIni);

			// Not used on PC as in-place creation inside bigger pool interacts with the exit purge and deleting UObject directly.
			GConfig->GetInt(TEXT("/Script/Engine.GarbageCollectionSettings"), TEXT("gc.SizeOfPermanentObjectPool"), SizeOfPermanentObjectPool, GEngineIni);
		}

		// Maximum number of UObjects in cooked game
		GConfig->GetInt(TEXT("/Script/Engine.GarbageCollectionSettings"), TEXT("gc.MaxObjectsInGame"), MaxUObjects, GEngineIni);

		// If true, the UObjectArray will pre-allocate all entries for UObject pointers
		GConfig->GetBool(TEXT("/Script/Engine.GarbageCollectionSettings"), TEXT("gc.PreAllocateUObjectArray"), bPreAllocateUObjectArray, GEngineIni);
	}
	else
	{
#if IS_PROGRAM
		// Maximum number of UObjects for programs can be low
		MaxUObjects = 100000; // Default to 100K for programs
		GConfig->GetInt(TEXT("/Script/Engine.GarbageCollectionSettings"), TEXT("gc.MaxObjectsInProgram"), MaxUObjects, GEngineIni);
#else
		// Maximum number of UObjects in the editor
		GConfig->GetInt(TEXT("/Script/Engine.GarbageCollectionSettings"), TEXT("gc.MaxObjectsInEditor"), MaxUObjects, GEngineIni);
#endif
	}


	// Log what we're doing to track down what really happens as log in LaunchEngineLoop doesn't report those settings in pristine form.
	UE_LOG(LogInit, Log, TEXT("%s for max %d objects, including %i objects not considered by GC, pre-allocating %i bytes for permanent pool."), 
		bPreAllocateUObjectArray ? TEXT("Pre-allocating") : TEXT("Presizing"),
		MaxUObjects, MaxObjectsNotConsideredByGC, SizeOfPermanentObjectPool);

	GUObjectAllocator.AllocatePermanentObjectPool(SizeOfPermanentObjectPool);
	GUObjectArray.AllocateObjectPool(MaxUObjects, MaxObjectsNotConsideredByGC, bPreAllocateUObjectArray);

	void InitAsyncThread();
	InitAsyncThread();

	// Note initialized.
	Internal::GetUObjectSubsystemInitialised() = true;

	UObjectProcessRegistrants();
}
```

有很多字符串可以用来找到下面这个函数。
```c
__int64 __fastcall fn_UObjectBaseInit_7FF7A9B729D0(int a1)
{
  int v1; // ecx
  int v2; // ecx
  __int64 v3; // rax
  unsigned int v4; // ebx
  __int64 v5; // rax
  int v7; // [rsp+40h] [rbp+8h] BYREF
  int v8; // [rsp+48h] [rbp+10h] BYREF
  int v9; // [rsp+50h] [rbp+18h] BYREF

  v8 = 0;
  v7 = 0;
  v9 = 0x200000;
  sub_7FF7A9452E84(
    a1,
    (unsigned int)L"/Script/Engine.GarbageCollectionSettings",
    (unsigned int)L"gc.MaxObjectsNotConsideredByGC",
    (unsigned int)&v8,
    (__int64)&qword_7FF7B9D4D898);
  sub_7FF7A9452E84(
    v1,
    (unsigned int)L"/Script/Engine.GarbageCollectionSettings",
    (unsigned int)L"gc.SizeOfPermanentObjectPool",
    (unsigned int)&v7,
    (__int64)&qword_7FF7B9D4D898);
  sub_7FF7A9452E84(
    v2,
    (unsigned int)L"/Script/Engine.GarbageCollectionSettings",
    (unsigned int)L"gc.MaxObjectsInGame",
    (unsigned int)&v9,
    (__int64)&qword_7FF7B9D4D898);
  dword_7FF7B9D53600 = v7;
  v3 = sub_7FF7A928827C(v7, 0);
  v4 = v9;
  qword_7FF7B9D53608 = v3;
  qword_7FF7B9D53610 = v3;
  qword_7FF7B9D53618 = v3;
  dword_7FF7B9D53888 = v8;
  dword_7FF7B9D53880 = (v8 <= 0) - 1;
  if ( v9 <= 0 )
  {
    sub_7FF7AB7CF038(
      "D:\\wk\\cwd1b\\build\\UnrealEngine\\Engine\\Source\\Runtime\\CoreUObject\\Private\\UObject\\UObjectArray.cpp",
      40,
      L"Max UObject count is invalid. It must be a number that is greater than 0.");
    sub_7FF7AB7CEB1C(
      (unsigned int)&unk_7FF7B7E26AAC,
      (unsigned int)"D:\\wk\\cwd1b\\build\\UnrealEngine\\Engine\\Source\\Runtime\\CoreUObject\\Private\\UObject\\UObjectArray.cpp",
      40,
      (unsigned int)L"Max UObject count is invalid. It must be a number that is greater than 0.");
  }
  sub_7FF7A9B72664((__int64)&g_GUObjects_7FF7B9D53890, v4);// 这里面有GUObjects指针
  if ( dword_7FF7B9D53888 > 0 )
    sub_7FF7A9B72AE0(&g_GUObjects_7FF7B9D53890);
  v5 = sub_7FF7A9312EE8();
  sub_7FF7A9B72DA4(v5);
  byte_7FF7B9AEA1D8 = 1;
  return sub_7FF7A9FC2330();
}

__int64 __fastcall sub_7FF7A9B72664(__int64 a1_g_GUObjects, unsigned int a2_MaxUObjects)
{
  __int64 v2; // rsi
  __int64 v4; // rdi
  __int64 v5; // rax
  __int64 v6; // rbx
  __int64 v7; // rax
  __int64 v8; // r8
  __int64 v9; // rax

  v2 = a2_MaxUObjects;
  v4 = (int)a2_MaxUObjects;
  is_mul_ok((int)a2_MaxUObjects, 0x18u);
  sub_7FF7A928827C();
  v6 = v5;
  if ( v5 )
    sub_7FF7A92A2B20(v5, 24, v4, sub_7FF7A9945990);
  else
    v6 = 0;
  v7 = MEMORY[0x7FF48EDD1AC4](1212075093, v6);
  *(_QWORD *)(a1_g_GUObjects + 0x10) = v7;      // 指针
  MEMORY[0x7FF48EDD1AEE](1212075093, v7);
  v9 = MEMORY[0x7FF48EDD1AC4](1212075513, v2, v8, v2);
  *(_QWORD *)(a1_g_GUObjects + 8) = v9;         // 不是capacity
  return MEMORY[0x7FF48EDD1AEE](1212075513, v9);
}
```

```c
    v10_InternalIndex = (*(_DWORD *)(v9 + 0x18) << 10)
                      & 0xFFFF0000
                      ^ __ROR4__(*(_DWORD *)(v9 + 0x18) ^ 0x68524A8E, 6)
                      ^ 0x6DE12AD9;
    v11 = v10_InternalIndex >= g_GUObjects_7FF7B9D53890// 这不是capacity也不是size，是InternalIndex上限？
        ? 0LL
        : MEMORY[0x7FF48EDD1AEE](1212075093, qword_7FF7B9D538A0) + 24LL * v10_InternalIndex;
                                                // 直接用下标，当作线性数组访问，不是分块数组
```
### 自动扫描
```c
    {
        uint64_t p = search_signature_in_buffer(g_patterns.GUObjects);
        uint64_t GUObjects = 0;
        uint64_t fn = 0;
        uint64_t fn2 = 0;
        uint64_t cnt = 0;
        g_dis.print(p, va_to_raw(p), 0x50,
                    [&fn, &fn2, &cnt, &GUObjects](uint64_t addr, ZydisDecodedInstruction &instr, ZydisDecodedOperand *ops) {
                        // 48 8D 0D F5 0D 1E 10                          lea     rcx, g_GUObjects_7FF7B9D53890
                        if (instr.mnemonic == ZYDIS_MNEMONIC_LEA) {
                            auto &src = ops[1];
                            if (src.mem.disp.has_displacement) {
                                GUObjects = addr + src.mem.disp.value;
                            }
                        }
                        uint64_t t = 0;
                        if (instr.mnemonic == ZYDIS_MNEMONIC_CALL) {
                            t = addr + ops[0].imm.value.s;
                            cnt++;
                        }
                        if (t && !fn) {
                            fn = t;
                            LOG("fn: %llx", fn);
                        }
                        if (t && cnt == 2 && !fn2) {
                            fn2 = t;
                            LOG("fn2: %llx", fn2);
                            return true;
                        }
                        return false;
                    });
        uint64_t pointer = 0;
        p = fn;
        g_dis.print(p, va_to_raw(p), 0x200,
                    [&pointer, &p](uint64_t addr, ZydisDecodedInstruction &instr, ZydisDecodedOperand *ops) {
                        // 49 89 46 10                                   mov     [r14+10h], rax
                        if (instr.mnemonic == ZYDIS_MNEMONIC_MOV) {
                            if (ops[1].reg.value == ZYDIS_REGISTER_RAX && ops[0].mem.base != ZYDIS_REGISTER_RSP &&
                                ops[0].mem.disp.has_displacement) {
                                pointer = ops[0].mem.disp.value;
                                LOG("pointer: %llx", pointer);
                                p = addr;
                                return true;
                            }
                        }
                        return false;
                    });
        uint64_t size = 0;
        p = fn2;
        g_dis.print(p, va_to_raw(p), 0x200,
                    [&size, &p](uint64_t addr, ZydisDecodedInstruction &instr, ZydisDecodedOperand *ops) {
                        // add     [rcx], edx
                        if (instr.mnemonic == ZYDIS_MNEMONIC_ADD) {
                            if (ops[0].mem.base == ZYDIS_REGISTER_RCX) {
                                size = ops[0].mem.disp.value;
                                LOG("Size_GUObjects: %llx", size);
                                return true;
                            }
                        }
                        return false;
                    });
        of::add_one_offset("GUObjects", GUObjects + pointer - m_pe.image_base);
        of::add_one_offset("Size_GUObjects", GUObjects + size - m_pe.image_base);
    }
```

```
00007ff7a9b72a92  8b d3                          mov edx, ebx
00007ff7a9b72a94  48 8d 0d f5 0d 1e 10           lea rcx, qword ptr [0x00007ff7b9d53890]
00007ff7a9b72a9b  e8 c4 fb ff ff                 call 0x00007ff7a9b72664
[LOG] fn: 7ff7a9b72664
00007ff7a9b72aa0  8b 15 e2 0d 1e 10              mov edx, dword ptr [0x00007ff7b9d53888]
00007ff7a9b72aa6  85 d2                          test edx, edx
00007ff7a9b72aa8  7e 0c                          jle 0x00007ff7a9b72ab6
00007ff7a9b72aaa  48 8d 0d df 0d 1e 10           lea rcx, qword ptr [0x00007ff7b9d53890]
00007ff7a9b72ab1  e8 2a 00 00 00                 call 0x00007ff7a9b72ae0
[LOG] fn2: 7ff7a9b72ae0
00007ff7a9b72664  48 89 5c 24 10                 mov qword ptr [rsp+0x10], rbx
00007ff7a9b72669  56                             push rsi
00007ff7a9b7266a  57                             push rdi
00007ff7a9b7266b  41 56                          push r14
00007ff7a9b7266d  48 83 ec 20                    sub rsp, 0x20
00007ff7a9b72671  8b f2                          mov esi, edx
00007ff7a9b72673  4c 8b f1                       mov r14, rcx
00007ff7a9b72676  48 63 fa                       movsxd rdi, edx
00007ff7a9b72679  48 89 7c 24 40                 mov qword ptr [rsp+0x40], rdi
00007ff7a9b7267e  b8 18 00 00 00                 mov eax, 0x18
00007ff7a9b72683  48 f7 e7                       mul rdi
00007ff7a9b72686  48 c7 c1 ff ff ff ff           mov rcx, 0xffffffffffffffff
00007ff7a9b7268d  48 0f 40 c1                    cmovo rax, rcx
00007ff7a9b72691  b9 01 00 00 00                 mov ecx, 0x01
00007ff7a9b72696  48 85 c0                       test rax, rax
00007ff7a9b72699  48 0f 45 c8                    cmovnz rcx, rax
00007ff7a9b7269d  33 d2                          xor edx, edx
00007ff7a9b7269f  e8 d8 5b 71 ff                 call 0x00007ff7a928827c
00007ff7a9b726a4  48 8b d8                       mov rbx, rax
00007ff7a9b726a7  48 89 44 24 50                 mov qword ptr [rsp+0x50], rax
00007ff7a9b726ac  48 85 c0                       test rax, rax
00007ff7a9b726af  74 19                          jz 0x00007ff7a9b726ca
00007ff7a9b726b1  4c 8d 0d d8 32 dd ff           lea r9, qword ptr [0x00007ff7a9945990]
00007ff7a9b726b8  4c 8b c7                       mov r8, rdi
00007ff7a9b726bb  ba 18 00 00 00                 mov edx, 0x18
00007ff7a9b726c0  48 8b c8                       mov rcx, rax
00007ff7a9b726c3  e8 58 04 73 ff                 call 0x00007ff7a92a2b20
00007ff7a9b726c8  eb 02                          jmp 0x00007ff7a9b726cc
00007ff7a9b726ca  33 db                          xor ebx, ebx
00007ff7a9b726cc  bf 55 cc 3e 48                 mov edi, 0x483ecc55
00007ff7a9b726d1  48 83 3d 27 fa b1 0e 00        cmp qword ptr [0x00007ff7b8692100], 0x00
00007ff7a9b726d9  75 10                          jnz 0x00007ff7a9b726eb
00007ff7a9b726db  48 8b d3                       mov rdx, rbx
00007ff7a9b726de  8b cf                          mov ecx, edi
00007ff7a9b726e0  48 8b 05 39 fa b1 0e           mov rax, qword ptr [0x00007ff7b8692120]
00007ff7a9b726e7  ff d0                          call rax
00007ff7a9b726e9  eb 3c                          jmp 0x00007ff7a9b72727
00007ff7a9b726eb  8b c3                          mov eax, ebx
00007ff7a9b726ed  35 8c f6 f0 7f                 xor eax, 0x7ff0f68c
00007ff7a9b726f2  f7 d0                          not eax
00007ff7a9b726f4  05 32 f5 ad 6d                 add eax, 0x6dadf532
00007ff7a9b726f9  35 42 fc a2 88                 xor eax, 0x88a2fc42
00007ff7a9b726fe  f7 d0                          not eax
00007ff7a9b72700  89 44 24 40                    mov dword ptr [rsp+0x40], eax
00007ff7a9b72704  48 c1 eb 20                    shr rbx, 0x20
00007ff7a9b72708  81 f3 65 57 81 e6              xor ebx, 0xe6815765
00007ff7a9b7270e  f7 d3                          not ebx
00007ff7a9b72710  81 c3 72 6e f2 6e              add ebx, 0x6ef26e72
00007ff7a9b72716  81 f3 17 39 73 23              xor ebx, 0x23733917
00007ff7a9b7271c  f7 d3                          not ebx
00007ff7a9b7271e  89 5c 24 44                    mov dword ptr [rsp+0x44], ebx
00007ff7a9b72722  48 8b 44 24 40                 mov rax, qword ptr [rsp+0x40]
00007ff7a9b72727  49 89 46 10                    mov qword ptr [r14+0x10], rax
[LOG] pointer: 10
00007ff7a9b72ae0  40 53                          push rbx
00007ff7a9b72ae2  48 83 ec 20                    sub rsp, 0x20
00007ff7a9b72ae6  01 11                          add dword ptr [rcx], edx
[LOG] Size_GUObjects: 0
GUObjects = 0x10d038a0; // 20260114
Size_GUObjects = 0x10d03890; // 20260114
```

### 遍历
```c
void GUObjectsHelper::read_slots() {
    LOG("> GUObjectsHelper::read_slots");
    static bool has_read_slots = false;
    if (has_read_slots) {
        return;
    }
    auto base = g_memory.get_base();
    LOG("base: %llx", base);
    g_memory.read_batch<uint64_t>(base + offsets.GUObjects, &m_guobjects);
    g_memory.read_batch<int32_t>(base + offsets.Size_GUObjects, &m_size_guobjects);
    g_memory.execute_read_batch();
    m_guobjects = local_decrypt.decrypt_pointer(m_guobjects);
    LOG("gobjects: %llx, size_guobjects: %d", m_guobjects, m_size_guobjects);
    if (!util::is_virtual_addr_valid(m_guobjects) || !(m_size_guobjects > 0 && m_size_guobjects < 9999999)) {
        FATAL_EXIT(1, "guobjects not valid");
        return;
    }
    // m_size_guobjects = 1000;
    m_objects_array.resize(m_size_guobjects);
    m_slots_array.resize(m_size_guobjects);
    // slot读取后还需要再deref一次才是UObject
    for (int32_t i = 0; i < m_objects_array.size(); i++) {
        auto &obj = m_objects_array[i];
        // 注意和DeltaForce不一样，这里下标直接用不需要解密
        auto internal_index = i;
        auto slot = m_guobjects + internal_index * 24;
        m_slots_array[i] = slot;
    }
    LOG("has read slots: %llu", m_slots_array.size());
    has_read_slots = true;
}
// [LOG] > GUObjectsHelper::read_slots
// [LOG] base: 7ff7a9050000
// [LOG] gobjects: 21f99710000, size_guobjects: 370420
// [LOG] has read slots: 370420
```

可以通过反射机制，得到类的虚表，UFunction函数名，函数地址，UField成员变量名，成员变量偏移。

可以看到很多名称是加了混淆的，变成随机字符串了。
```
ii: 5, obj: 21f978c02c0, name: /Script/Engine, class: Package, vt: 7ff7b74f48c0
ii: 6, obj: 21f91a27680, name: *900b88416d, class: Class, vt: 7ff7b862ac50
ii: 7, obj: 21f91a27940, name: *f46a2ed26a, class: Class, vt: 7ff7b862ac50
ii: 8, obj: 21f91a27100, name: *f66d21396d, class: Class, vt: 7ff7b862ac50
ii: 9, obj: 21f91a268c0, name: Actor, class: Class, vt: 7ff7b862ac50
ii: 10, obj: 21f91a26b80, name: HUD, class: Class, vt: 7ff7b862ac50

ii: 48345, obj: 21f98033180, name: WorldOriginLocationChanged, class: Function, outer: LevelScriptActor, addr: 7ff7a94fafe8
ii: 48346, obj: 21f980332c0, name: NewOriginLocation, class: StructProperty, outer: WorldOriginLocationChanged, offset: 0x0
ii: 48347, obj: 21f98033360, name: OldOriginLocation, class: StructProperty, outer: WorldOriginLocationChanged, offset: 0x52
```


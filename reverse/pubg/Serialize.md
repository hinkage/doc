# Serialize
序列化必须知道成员变量的偏移和大小，所以这是逆向UE4的重要突破口。

## 2026-01-17
```c
class COREUOBJECT_API UObject : public UObjectBaseUtility
	/** 
	 * Handles reading, writing, and reference collecting using FArchive.
	 * This implementation handles all UProperty serialization, but can be overridden for native variables.
	 */
	virtual void Serialize(FArchive& Ar);
	virtual void Serialize(FStructuredArchive::FRecord Record);
}
```

"Script serialization mismatch: Got %i, expected %i"

这个字符串竟然没被隐藏，直接定位到UStruct::Serialize。

```c
void UStruct::Serialize( FArchive& Ar )
{
	Super::Serialize( Ar );

#if USTRUCT_FAST_ISCHILDOF_IMPL == USTRUCT_ISCHILDOF_STRUCTARRAY
	UStruct* SuperStructBefore = GetSuperStruct();
#endif

	Ar << SuperStruct;

#if USTRUCT_FAST_ISCHILDOF_IMPL == USTRUCT_ISCHILDOF_STRUCTARRAY
	if (Ar.IsLoading())
	{
		this->ReinitializeBaseChainArray();
	}
	// Handle that fact that FArchive takes UObject*s by reference, and archives can just blat
	// over our SuperStruct with impunity.
	else if (SuperStructBefore)
	{
		UStruct* SuperStructAfter = GetSuperStruct();
		if (SuperStructBefore != SuperStructAfter)
		{
			this->ReinitializeBaseChainArray();
		}
	}
#endif

	Ar.UsingCustomVersion(FFrameworkObjectVersion::GUID);
	if (Ar.CustomVer(FFrameworkObjectVersion::GUID) < FFrameworkObjectVersion::RemoveUField_Next)
	{
		Ar << Children;
	}
	else
	{
		TArray<UField*> ChildArray;
		if (Ar.IsLoading())
		{
			Ar << ChildArray;
			if (ChildArray.Num())
			{
				for (int32 Index = 0; Index + 1 < ChildArray.Num(); Index++)
				{
					ChildArray[Index]->Next = ChildArray[Index + 1];
				}
				Children = ChildArray[0];
				ChildArray[ChildArray.Num() - 1]->Next = nullptr;
			}
			else
			{
				Children = nullptr;
			}
		}
		else
		{
			UField* Child = Children;
			while (Child)
			{
				ChildArray.Add(Child);
				Child = Child->Next;
			}
			Ar << ChildArray;
		}
	}


	if (Ar.IsLoading())
	{
		FStructScriptLoader ScriptLoadHelper(/*TargetScriptContainer =*/this, Ar);
#if USE_CIRCULAR_DEPENDENCY_LOAD_DEFERRING
		bool const bAllowDeferredScriptSerialization = true;
#else  // USE_CIRCULAR_DEPENDENCY_LOAD_DEFERRING
		bool const bAllowDeferredScriptSerialization = false;
#endif // USE_CIRCULAR_DEPENDENCY_LOAD_DEFERRING

		// NOTE: if bAllowDeferredScriptSerialization is set to true, then this
		//       could temporarily skip script serialization (as it could 
		//       introduce unwanted dependency loads at this time)
		ScriptLoadHelper.LoadStructWithScript(this, Ar, bAllowDeferredScriptSerialization);

		if (!dynamic_cast<UClass*>(this) && !(Ar.GetPortFlags() & PPF_Duplicate)) // classes are linked in the UClass serializer, which just called me
		{
			// Link the properties.
			Link(Ar, true);
		}
	}
	else
	{
		int32 ScriptBytecodeSize = Script.Num();
		int64 ScriptStorageSizeOffset = INDEX_NONE;

		if (Ar.IsSaving())
		{
			FArchive::FScopeSetDebugSerializationFlags S(Ar, DSF_IgnoreDiff);

			Ar << ScriptBytecodeSize;

			int32 ScriptStorageSize = 0;
			// drop a zero here.  will seek back later and re-write it when we know it
			ScriptStorageSizeOffset = Ar.Tell();
			Ar << ScriptStorageSize;
		}

		// Skip serialization if we're duplicating classes for reinstancing, since we only need the memory layout
		if (!GIsDuplicatingClassForReinstancing)
		{

			// no bytecode patch for this struct - serialize normally [i.e. from disk]
			int32 iCode = 0;
			int64 const BytecodeStartOffset = Ar.Tell();

			if (Ar.IsPersistent() && Ar.GetLinker())
			{
				// make sure this is a ULinkerSave
				FLinkerSave* LinkerSave = CastChecked<FLinkerSave>(Ar.GetLinker());

				// remember how we were saving
				FArchive* SavedSaver = LinkerSave->Saver;

				// force writing to a buffer
				TArray<uint8> TempScript;
				FMemoryWriter MemWriter(TempScript, Ar.IsPersistent());
				LinkerSave->Saver = &MemWriter;

				// now, use the linker to save the byte code, but writing to memory
				while (iCode < ScriptBytecodeSize)
				{
					SerializeExpr(iCode, Ar);
				}

				// restore the saver
				LinkerSave->Saver = SavedSaver;

				// now write out the memory bytes
				Ar.Serialize(TempScript.GetData(), TempScript.Num());

				// and update the SHA (does nothing if not currently calculating SHA)
				LinkerSave->UpdateScriptSHAKey(TempScript);
			}
			else
			{
				while (iCode < ScriptBytecodeSize)
				{
					SerializeExpr(iCode, Ar);
				}
			}

			if (iCode != ScriptBytecodeSize)
			{
				UE_LOG(LogClass, Fatal, TEXT("Script serialization mismatch: Got %i, expected %i"), iCode, ScriptBytecodeSize);
			}

			if (Ar.IsSaving())
			{
				FArchive::FScopeSetDebugSerializationFlags S(Ar, DSF_IgnoreDiff);

				int64 const BytecodeEndOffset = Ar.Tell();

				// go back and write on-disk size
				Ar.Seek(ScriptStorageSizeOffset);
				int32 ScriptStorageSize = BytecodeEndOffset - BytecodeStartOffset;
				Ar << ScriptStorageSize;

				// back to where we were
				Ar.Seek(BytecodeEndOffset);
			}
		} // if !GIsDuplicatingClassForReinstancing
	}
}
```

```c
__int64 __fastcall fn_UStruct::Serialize_7FF7A950801C(__int64 a1_UStruct, __int64 a2_Ar)
{
  __int64 v4; // rax
  __int64 v5; // r8
  unsigned __int64 v6; // rdx
  __int64 v7; // r9
  __int64 result; // rax
  __int64 v9; // rcx
  __int64 v10; // r12
  int **v11; // rcx
  _BYTE *v12; // r14
  int **v13; // rdx
  int v14; // r13d
  __int64 v15; // rax
  int v16; // r8d
  __int64 v17; // r14
  __int64 v18; // r15
  __int64 v19; // rcx
  __int64 v20; // rdx
  __int64 v21; // rcx
  __int64 v22; // rsi
  __int64 v23; // r14
  __int64 *v24; // r8
  __int64 v25; // rdx
  __int64 v26; // rcx
  unsigned int v27; // eax
  int v28; // r9d
  __int64 v29; // rdi
  __int64 v30; // rdx
  __int64 v31; // [rsp+20h] [rbp-99h]
  __int64 v32; // [rsp+30h] [rbp-89h] BYREF
  __int64 v33; // [rsp+38h] [rbp-81h]
  _BYTE v34[208]; // [rsp+40h] [rbp-79h] BYREF
  unsigned int v35; // [rsp+120h] [rbp+67h] BYREF
  int v36; // [rsp+128h] [rbp+6Fh] BYREF
  __int64 v37; // [rsp+130h] [rbp+77h] BYREF

  fn_UObject_Serialize_7FF7A92FF258(a1_UStruct, (_BYTE *)a2_Ar);// Super::Serialize( Ar );
                                                // 即UField::Serialize，但实际上是UObject::Serialize
  (*(void (__fastcall **)(__int64, __int64))(*(_QWORD *)a2_Ar + 0x28LL))(a2_Ar, a1_UStruct + 0x30);
  (*(void (__fastcall **)(__int64, __int64))(*(_QWORD *)a1_UStruct + 0x2A0LL))(a1_UStruct, a2_Ar);
  (*(void (__fastcall **)(__int64, __int64))(*(_QWORD *)a2_Ar + 0x28LL))(a2_Ar, a1_UStruct + 0x78);// UStruct_Children
  if ( (*(_BYTE *)(a2_Ar + 0x28) & 1) != 0 )
  {
    sub_7FF7A950812C(&v32, a1_UStruct, a2_Ar);
    sub_7FF7A9508CE0(&v32, a1_UStruct, a2_Ar);
    v4 = sub_7FF7A92D6A30();
    v5 = 0x1F4716EF042A1C8BLL;
    v6 = (((*(_QWORD *)(a1_UStruct + 0x10) ^ 0xA5B2657BD9956F30uLL) & 0xFFFFFFFFFF000000uLL) << 8)// UObject_ClassPrivate
       ^ 0x1F4716EF042A1C8BLL
       ^ __ROR8__(*(_QWORD *)(a1_UStruct + 0x10) ^ 0xA5B2657BD9956F30uLL, 24);
    v7 = v4 + 232;
    result = *(int *)(v4 + 0xF0);
    if ( (int)result > *(_DWORD *)(v6 + 0xF0)
      || (v9 = result, result = *(_QWORD *)(v6 + 0xE8), *(_QWORD *)(result + 8 * v9) != v7)
      || !a1_UStruct )
    {
      if ( (*(_DWORD *)(a2_Ar + 0x30) & 0x1000) == 0 )
      {
        LOBYTE(v5) = 1;
        return (*(__int64 (__fastcall **)(__int64, __int64, __int64))(*(_QWORD *)a1_UStruct + 0x258LL))(
                 a1_UStruct,
                 a2_Ar,
                 v5);
      }
    }
    return result;
  }
  result = *(unsigned int *)(a1_UStruct + 0x88);
  v35 = *(_DWORD *)(a1_UStruct + 0x88);
  v10 = -1;
  if ( (*(_BYTE *)(a2_Ar + 40) & 2) != 0 )
  {
    v11 = *(int ***)(a2_Ar + 8);
    if ( *v11 + 1 > v11[1] )
    {
      (*(void (__fastcall **)(__int64, unsigned int *, __int64))(*(_QWORD *)a2_Ar + 72LL))(a2_Ar, &v35, 4);
      v12 = (_BYTE *)(a2_Ar + 41);
      if ( (*(_BYTE *)(a2_Ar + 41) & 8) != 0 )
        sub_7FF7AB7CF88C(a2_Ar, (__int64)&v35, 4);
    }
    else
    {
      v35 = *(*v11)++;
      v12 = (_BYTE *)(a2_Ar + 0x29);
    }
    v36 = 0;
    result = (*(__int64 (__fastcall **)(__int64))(*(_QWORD *)a2_Ar + 0x88LL))(a2_Ar);
    v10 = result;
    v13 = *(int ***)(a2_Ar + 8);
    if ( *v13 + 1 > v13[1] )
    {
      result = (*(__int64 (__fastcall **)(__int64, int *, __int64))(*(_QWORD *)a2_Ar + 72LL))(a2_Ar, &v36, 4);
      if ( (*v12 & 8) != 0 )
        result = sub_7FF7AB7CF88C(a2_Ar, (__int64)&v36, 4);
    }
    else
    {
      v36 = *(*v13)++;
    }
  }
  if ( !byte_7FF7B9FF82E1 )
  {
    v36 = 0;
    v14 = (*(__int64 (__fastcall **)(__int64))(*(_QWORD *)a2_Ar + 0x88LL))(a2_Ar);
    if ( (*(_BYTE *)(a2_Ar + 0x28) & 0x20) == 0
      || !(*(__int64 (__fastcall **)(__int64))(*(_QWORD *)a2_Ar + 0x80LL))(a2_Ar) )
    {
      while ( 1 )
      {
        result = v35;
        v28 = v36;
        if ( v36 >= (int)v35 )
          break;
        (*(void (__fastcall **)(__int64, int *, __int64))(*(_QWORD *)a1_UStruct + 0x288LL))(a1_UStruct, &v36, a2_Ar);
      }
      goto LABEL_47;
    }
    v15 = (*(__int64 (__fastcall **)(__int64))(*(_QWORD *)a2_Ar + 0x80LL))(a2_Ar);
    v17 = v15;
    if ( !v15 || *(_DWORD *)(v15 + 152) != 2 )
      v17 = 0;
    v18 = *(_QWORD *)(v17 + 0x2D8);
    v32 = 0;
    v33 = 0;
    v37 = 0;
    LOBYTE(v16) = (*(_BYTE *)(a2_Ar + 40) & 0x20) != 0;
    sub_7FF7A968CA2C((unsigned int)v34, (unsigned int)&v32, v16, 0, 0);
    *(_QWORD *)(v17 + 0x2D8) = v34;
    while ( v36 < (int)v35 )
      (*(void (__fastcall **)(__int64, int *, __int64))(*(_QWORD *)a1_UStruct + 0x288LL))(a1_UStruct, &v36, a2_Ar);
    *(_QWORD *)(v17 + 0x2D8) = v18;
    (*(void (__fastcall **)(__int64, __int64, _QWORD))(*(_QWORD *)a2_Ar + 72LL))(a2_Ar, v32, (int)v33);
    v19 = *(_QWORD *)(v17 + 0x248);
    if ( v19 && (_DWORD)v33 )
      sub_7FF7A94BE8BC(v19, v32);
    sub_7FF7A93D7224(v34);
    v22 = v32;
    if ( !v32 )
    {
LABEL_44:
      result = v35;
      v28 = v36;
LABEL_47:
      if ( v28 != (_DWORD)result )              // if (iCode != ScriptBytecodeSize)
                                                //             {
                                                //                 UE_LOG(LogClass, Fatal, TEXT("Script serialization mismatch: Got %i, expected %i"), iCode, ScriptBytecodeSize);
                                                //             }
      {
        sub_7FF7AB7CF038(
          "D:\\wk\\cwd1b\\build\\UnrealEngine\\Engine\\Source\\Runtime\\CoreUObject\\Private\\UObject\\Class.cpp",
          1310,
          L"Script serialization mismatch: Got %i, expected %i");
        LODWORD(v31) = v36;
        result = sub_7FF7AB7CEB1C(
                   (unsigned int)&unk_7FF7B7E26A8A,
                   (unsigned int)"D:\\wk\\cwd1b\\build\\UnrealEngine\\Engine\\Source\\Runtime\\CoreUObject\\Private\\UObject\\Class.cpp",
                   1310,
                   (unsigned int)L"Script serialization mismatch: Got %i, expected %i",
                   v31,
                   v35);
      }
      if ( (*(_BYTE *)(a2_Ar + 0x28) & 2) != 0 )// if (Ar.IsSaving())
      {
        v29 = (*(__int64 (__fastcall **)(__int64))(*(_QWORD *)a2_Ar + 0x88LL))(a2_Ar);// int64 const BytecodeEndOffset = Ar.Tell();
        (*(void (__fastcall **)(__int64, __int64))(*(_QWORD *)a2_Ar + 0xA8LL))(a2_Ar, v10);// Ar.Seek(ScriptStorageSizeOffset);
        LODWORD(v37) = v29 - v14;               // int32 ScriptStorageSize = BytecodeEndOffset - BytecodeStartOffset;
        v30 = *(_QWORD *)(a2_Ar + 8);
        if ( (unsigned __int64)(*(_QWORD *)v30 + 4LL) > *(_QWORD *)(v30 + 8) )
        {
          (*(void (__fastcall **)(__int64, __int64 *, __int64))(*(_QWORD *)a2_Ar + 0x48LL))(a2_Ar, &v37, 4);
          if ( (*(_BYTE *)(a2_Ar + 41) & 8) != 0 )
            sub_7FF7AB7CF88C(a2_Ar, (__int64)&v37, 4);
        }
        else
        {
          LODWORD(v37) = **(_DWORD **)v30;
          *(_QWORD *)v30 += 4LL;
        }
        return (*(__int64 (__fastcall **)(__int64, __int64))(*(_QWORD *)a2_Ar + 0xA8LL))(a2_Ar, v29);// Ar.Seek(BytecodeEndOffset);
      }
      return result;
    }
    v23 = qword_7FF7B99E7000;
    if ( !qword_7FF7B99E7000 )
    {
      sub_7FF7A97281C4(v21, v20);
      (*(void (__fastcall **)(__int64, __int64))(*(_QWORD *)qword_7FF7B99E7000 + 32LL))(qword_7FF7B99E7000, v22);
      goto LABEL_44;
    }
    v24 = *(__int64 **)(*(_QWORD *)qword_7FF7B99E7000 + 32LL);
    if ( v24 != qword_7FF7A9FB0050 )
    {
      ((void (__fastcall *)(__int64, __int64))v24)(qword_7FF7B99E7000, v32);
      goto LABEL_44;
    }
    if ( (_WORD)v32 )
    {
      if ( dword_7FF7B99BE340 )
      {
        v25 = MEMORY[0x7FFFF3FD5880]();
        if ( v25 )
        {
          if ( *(_BYTE *)((v22 & 0xFFFFFFFFFFFF0000uLL) + 3) == 0xE3 )
          {
            v26 = 32LL * *(unsigned __int8 *)((v22 & 0xFFFFFFFFFFFF0000uLL) + 2);
            v27 = *(_DWORD *)(v26 + v25 + 16);
            if ( v27 < 0x40 && *(unsigned __int16 *)(v22 & 0xFFFFFFFFFFFF0000uLL) * v27 < 0x10000 )
              goto LABEL_41;
            if ( !*(_QWORD *)(v26 + v25 + 24) )
            {
              *(_OWORD *)(v26 + v25 + 24) = *(_OWORD *)(v26 + v25 + 8);
              *(_QWORD *)(v26 + v25 + 8) = 0;
              *(_DWORD *)(v26 + v25 + 16) = 0;
LABEL_41:
              *(_QWORD *)v22 = *(_QWORD *)(v26 + v25 + 8);
              *(_QWORD *)(v26 + v25 + 8) = v22;
              *(_DWORD *)(v22 + 8) = ++*(_DWORD *)(v26 + v25 + 16);
              goto LABEL_44;
            }
          }
        }
      }
    }
    sub_7FF7A9FB0110(v23, v22);
    goto LABEL_44;
  }
  return result;
}
```

### 找出Serialize函数在虚表中的偏移量
```
fn_UObject_Serialize_7FF7A92FF258(a1_UStruct, (_BYTE *)a2_Ar);// Super::Serialize( Ar );
                                            // 即UField::Serialize，但实际上是UObject::Serialize

ii: 3705, obj: 21f978c1fa0, name: Default__Object, class: Object, vt: 7ff7b74f3d80
从反射机制，得到UObject虚表地址7ff7b74f3d80

在虚表中找到UObject::Serialize函数，得出其虚表偏移量。
offset_vt_UObject_Serialize = 0x7FF7B74F3E20 - 0x7ff7b74f3d80
即0xa0。
破案了，其它继承自UObject的对象的Serialize函数，都可以通过这个虚表偏移来定位了。
所有字段的偏移，就都不是问题了。
这是UE4逆向的超级突破口。

.rdata:00007FF7B74F3D80 88 42 A2 B4 F7 7F 00 00       off_7FF7B74F3D80 dq offset qword_7FF7B4A23C90+5F8h
.rdata:00007FF7B74F3D80                                                                       ; DATA XREF: sub_7FF7A96B3BF8+15↑o
.rdata:00007FF7B74F3D80                                                                       ; sub_7FF7A981EB8C+22↑o ...
.rdata:00007FF7B74F3D88 40 B6 60 A9 F7 7F 00 00                       dq offset TslGame_Win64_Shipping_291
.rdata:00007FF7B74F3D90 B0 11 FC A9 F7 7F 00 00                       dq offset loc_7FF7A9FC11B0
.rdata:00007FF7B74F3D98 D0 B7 9A A9 F7 7F 00 00                       dq offset ?Reserve@WriteBytesCount@AK@@UEAA_NJ@Z ; AK::WriteBytesCount::Reserve(long)
.rdata:00007FF7B74F3DA0 38 6C 2D A9 F7 7F 00 00                       dq offset sub_7FF7A92D6C38
.rdata:00007FF7B74F3DA8 78 79 50 A9 F7 7F 00 00                       dq offset sub_7FF7A9507978
.rdata:00007FF7B74F3DB0 40 B6 60 A9 F7 7F 00 00                       dq offset TslGame_Win64_Shipping_291
.rdata:00007FF7B74F3DB8 4C 63 50 A9 F7 7F 00 00                       dq offset sub_7FF7A950634C
.rdata:00007FF7B74F3DC0 B8 B6 A2 B4 F7 7F 00 00                       dq offset sub_7FF7B4A2B6B8
.rdata:00007FF7B74F3DC8 88 6C 2D A9 F7 7F 00 00                       dq offset sub_7FF7A92D6C88
.rdata:00007FF7B74F3DD0 40 B6 60 A9 F7 7F 00 00                       dq offset TslGame_Win64_Shipping_291
.rdata:00007FF7B74F3DD8 D0 B7 9A A9 F7 7F 00 00                       dq offset ?Reserve@WriteBytesCount@AK@@UEAA_NJ@Z ; AK::WriteBytesCount::Reserve(long)
.rdata:00007FF7B74F3DE0 40 B6 60 A9 F7 7F 00 00                       dq offset TslGame_Win64_Shipping_291
.rdata:00007FF7B74F3DE8 40 B6 60 A9 F7 7F 00 00                       dq offset TslGame_Win64_Shipping_291
.rdata:00007FF7B74F3DF0 68 0C 67 A9 F7 7F 00 00                       dq offset sub_7FF7A9670C68
.rdata:00007FF7B74F3DF8 40 8D 62 A9 F7 7F 00 00                       dq offset sub_7FF7A9628D40
.rdata:00007FF7B74F3E00 94 F4 30 A9 F7 7F 00 00                       dq offset sub_7FF7A930F494
.rdata:00007FF7B74F3E08 84 1C 67 A9 F7 7F 00 00                       dq offset sub_7FF7A9671C84
.rdata:00007FF7B74F3E10 10 AF B3 A9 F7 7F 00 00                       dq offset sub_7FF7A9B3AF10
.rdata:00007FF7B74F3E18 98 D2 55 A9 F7 7F 00 00                       dq offset sub_7FF7A955D298
.rdata:00007FF7B74F3E20 58 F2 2F A9 F7 7F 00 00                       dq offset fn_UObject_Serialize_7FF7A92FF258 ; 破案了！
.rdata:00007FF7B74F3E28 40 B6 60 A9 F7 7F 00 00                       dq offset TslGame_Win64_Shipping_291
```


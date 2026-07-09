# hello guys in honor of egypt making it to the round of 16 for the first time ever i will leak 3 struct members

Ok first you find luaM_newgco (0x4777cc0) in latest as of writing

You then compare between the pseudocode and source code

```cpp
__int64 __fastcall sub_4777CC0(__int64 a1, __int64 a2, unsigned __int8 a3) 
{
    __int64 v3; // rsi
    __int64 v4; // r14
    __int64 v7; // rbp
    _DWORD *v8; // rax
    void (__fastcall *v9)(__int64, _QWORD, __int64); // rax

    v3 = *(_QWORD *)(a1 + 16);
    v4 = a3;

    if ((unsigned __int64)(a2 - 1) >= 0x400 || byte_78CDE40[a2] < 0) 
    {
        v8 = sub_47781C0(a1, (_QWORD **)(v3 + 792), (int)a2 + 64, a2, 1);
        v7 = (__int64)(v8 + 16);
        v8 -= v8;
        ++v8;
    } 
    else 
    {
        v7 = sub_4778130();
    }

    if (v7 == 0 && a2 != 0) 
    {
        sub_47562A0(a1, 4u);
    }

    *(_QWORD *)(v3 + 56) += a2;
    *(_QWORD *)(v3 + 8 * v4 + 11312) += a2;

    v9 = *(void (__fastcall **)(__int64, _QWORD, __int64))(v3 + 1336);
    if (v9 != nullptr) 
    {
        v9(a1, 0, a2);
    }

    return v7;
}
```

    *(_QWORD *)(v3 + 56) += a2; -> offset 56 decimal which is g->totalbytes 
fromsourcecode:     g->totalbytes += nsize;

       
 v8 = sub_47781C0(a1, (_QWORD **)(v3 + 792), (int)a2 + 64, a2, 1); is offset 792 decimal which is g->allgcopages
fromsourcecode:         lua_Page* page = newpage(L, &g->allgcopages, offsetof(lua_Page, data) + int(nsize), int(nsize), 1);


    *(_QWORD *)(v3 + 8 * v4 + 11312) += a2; is offset 11312 decimal which is g->memcatbytes 
  fromsourcecode:  g->memcatbytes[memcat] += nsize;

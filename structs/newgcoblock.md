# `newgcoblock` 

> Sorry if the write-up is ass. I used chatgpt to help with the writing since im not he best at writing
>
> New write-up in like 30 minutes after this one maybe

## Source Code

```cpp
static void* newgcoblock(lua_State* L, int sizeClass)
{
    global_State* g = L->global;
    lua_Page* page = g->freegcopages[sizeClass];

    // slow path: no page in the freelist, allocate a new one
    if (!page)
        page = newclasspage(L, g->freegcopages, &g->allgcopages, sizeClass, false);

    LUAU_ASSERT(!page->prev);
    LUAU_ASSERT(page->freeList || page->freeNext >= 0);
    LUAU_ASSERT(page->blockSize == kSizeClassConfig.sizeOfClass[sizeClass]);

    void* block;

    if (page->freeNext >= 0)
    {
        block = &page->data + page->freeNext;
        ASAN_UNPOISON_MEMORY_REGION(block, page->blockSize);

        page->freeNext -= page->blockSize;
        page->busyBlocks++;
    }
    else
    {
        block = page->freeList;
        ASAN_UNPOISON_MEMORY_REGION((char*)block + sizeof(GCheader), page->blockSize - sizeof(GCheader));

        // when separate block metadata is not used, free list link is stored inside the block data itself
        page->freeList = freegcolink(block);
        page->busyBlocks++;
    }

    // if we allocate the last block out of a page, we need to remove it from free list
    if (!page->freeList && page->freeNext < 0)
    {
        g->freegcopages[sizeClass] = page->next;

        if (page->next)
            page->next->prev = NULL;

        page->next = NULL;
    }

    return block;
}
```

---

## IDA Output

```cpp
__int64 *__fastcall sub_7ED2C0(__int64 a1, int a2)
{
  __int64 v2;
  __int64 v3;
  __int64 v4;
  __int64 v5;
  __int64 v6;
  __int64 *v7;
  __int64 v8;

  v2 = a2;
  v3 = *(_QWORD *)(a1 + 16);
  v4 = v3 + 8 * v2;
  v5 = *(_QWORD *)(v4 + 464);

  if (!v5)
    v5 = sub_7ED360(a1, (int)v3 + 464, 0, v2, 1);

  v6 = *(int *)(v5 + 48);

  if ((int)v6 < 0)
  {
    v7 = *(__int64 **)(v5 + 40);
    *(_QWORD *)(v5 + 40) = *v7;
  }
  else
  {
    v7 = (__int64 *)(v6 + v5 + 64);
    *(_DWORD *)(v5 + 48) = v6 - *(_DWORD *)(v5 + 36);
  }

  ++*(_DWORD *)(v5 + 52);
  *v7 = v5;

  if (!*(_QWORD *)(v5 + 40) && *(int *)(v5 + 48) < 0)
  {
    *(_QWORD *)(v4 + 464) = *(_QWORD *)(v5 + 24);
    v8 = *(_QWORD *)(v5 + 24);

    if (v8)
      *(_QWORD *)(v8 + 16) = 0;

    *(_QWORD *)(v5 + 24) = 0;
  }

  return v7 + 1;
}
```

---

## Reconstructed Function

Using the decompilation, we can reconstruct the function fairly accurately (thanks to DeepSeek for helping annotate it):

```cpp
static void* newgcoblock(lua_State* L, int sizeClass)
{
    // IDA:
    // a1 = L
    // a2 = sizeClass

    global_State* g = L->global;
    // v3 = *(_QWORD *)(a1 + 16)

    lua_Page* page = g->freegcopages[sizeClass];
    // v5 = *(_QWORD *)(v4 + 464)

    // Slow path
    if (!page)
        page = newclasspage(L, g->freegcopages, &g->allgcopages, sizeClass, false);

    LUAU_ASSERT(!page->prev);
    LUAU_ASSERT(page->freeList || page->freeNext >= 0);
    LUAU_ASSERT(page->blockSize == kSizeClassConfig.sizeOfClass[sizeClass]);

    void* block;

    if (page->freeNext >= 0)
    {
        block = &page->data + page->freeNext;
        ASAN_UNPOISON_MEMORY_REGION(block, page->blockSize);

        page->freeNext -= page->blockSize;
        page->busyBlocks++;
    }
    else
    {
        block = page->freeList;
        ASAN_UNPOISON_MEMORY_REGION(
            (char*)block + sizeof(GCheader),
            page->blockSize - sizeof(GCheader));

        page->freeList = freegcolink(block);
        page->busyBlocks++;
    }

    if (!page->freeList && page->freeNext < 0)
    {
        g->freegcopages[sizeClass] = page->next;

        if (page->next)
            page->next->prev = NULL;

        page->next = NULL;
    }

    return block;
}
```

---

# Struct Recovery

From the decompilation we can recover several useful members of both `global_State` and `lua_Page`.

### `global_State`

```cpp
v5 = *(_QWORD *)(v4 + 464); // 0x1d0 freegcopages
// page = g->freegcopages[sizeClass]
```
```cpp
v3 = *(_QWORD *)(a1 + 16); // 0x10 luastate
```

This identifies the location of:

- `global_State->freegcopages`
- `lua_State->global`

---

### `lua_Page`

```cpp
v8 = *(_QWORD *)(v5 + 24);
// page->next (0x18)

if (v8)
    *(_QWORD *)(v8 + 16) = 0;
// page->next->prev = NULL (0x10)
```

```cpp
*(_DWORD *)(v5 + 48) = v6 - *(_DWORD *)(v5 + 36);
// page->freeNext -= page->blockSize
// freeNext = 0x30
// blockSize = 0x24
```

```cpp
v6 = *(int *)(v5 + 48);
// page->freeNext (0x30)
```

```cpp
++*(_DWORD *)(v5 + 52);
// page->busyBlocks++ (0x34)
```

```cpp
v7 = (__int64 *)(v6 + v5 + 64);
// &page->data + page->freeNext
// data = 0x40
```

---

so:
```cpp
struct lua_Page
{
    lua_Page* listprev;   // notincluded
    lua_Page* listnext;   // 0x08

    lua_Page* prev;       // 0x10
    lua_Page* next;       // 0x18

    int pageSize;         // 0x20
    int blockSize;        // 0x24

    void* freeList;       // 0x28

    int freeNext;         // 0x30
    int busyBlocks;       // 0x34

    char padding[8]; //padding

    char data[1];         // 0x40
};
```

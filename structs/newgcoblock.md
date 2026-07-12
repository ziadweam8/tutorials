# newgcoblock thing sorry if writeup is bad i use deepseek since im not the best at writing 
## new writeup in 30m~ btw after this


Source Code:
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

    Output from IDA:
    __int64 *__fastcall sub_7ED2C0(__int64 a1, int a2)
{
  __int64 v2; // r9
  __int64 v3; // rdx
  __int64 v4; // rbx
  __int64 v5; // r8
  __int64 v6; // rax
  __int64 *v7; // rdx
  __int64 v8; // rax

  v2 = a2;
  v3 = *(_QWORD *)(a1 + 16);
  v4 = v3 + 8 * v2;
  v5 = *(_QWORD *)(v4 + 464);
  if ( !v5 )
    v5 = sub_7ED360(a1, (int)v3 + 464, 0, v2, 1);
  v6 = *(int *)(v5 + 48);
  if ( (int)v6 < 0 )
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
  if ( !*(_QWORD *)(v5 + 40) && *(int *)(v5 + 48) < 0 )
  {
    *(_QWORD *)(v4 + 464) = *(_QWORD *)(v5 + 24);
    v8 = *(_QWORD *)(v5 + 24);
    if ( v8 )
      *(_QWORD *)(v8 + 16) = 0;
    *(_QWORD *)(v5 + 24) = 0;
  }
  return v7 + 1;
}

This allows us to produce (thanks deepseek):

static void* newgcoblock(lua_State* L, int sizeClass)
{
    // IDA: a1 = L, a2 = sizeClass
    // IDA: v2 = sizeClass
    // IDA: v3 = *(_QWORD *)(a1 + 16)  // L->global at offset 16
    
    global_State* g = L->global;  // v3 = *(_QWORD *)(a1 + 16)
    // IDA: v4 = v3 + 8 * v2  // g + (sizeClass * 8) for array indexing
    
    // IDA: v5 = *(_QWORD *)(v4 + 464)  // g->freegcopages[sizeClass] at offset 464
    lua_Page* page = g->freegcopages[sizeClass];  // v5 = page pointer

    // slow path: no page in the freelist, allocate a new one
    // IDA: if (!v5) v5 = sub_7ED360(a1, (int)v3 + 464, 0, v2, 1)
    if (!page)
        page = newclasspage(L, g->freegcopages, &g->allgcopages, sizeClass, false);

    // IDA: offsets checked but not explicitly in decompilation
    LUAU_ASSERT(!page->prev);        // page->prev at offset 16
    LUAU_ASSERT(page->freeList || page->freeNext >= 0);  // offsets 40 and 48
    LUAU_ASSERT(page->blockSize == kSizeClassConfig.sizeOfClass[sizeClass]);  // offset 36

    void* block;
    // IDA: v6 = *(int *)(v5 + 48)  // page->freeNext at offset 48
    // IDA: if ((int)v6 < 0)  // IDA inverted condition for freeNext >= 0

    if (page->freeNext >= 0)  // page->freeNext at offset 48
    {
        // IDA: block = &page->data + page->freeNext
        // IDA: v7 = (__int64 *)(v6 + v5 + 64)  // data starts at offset 64
        block = &page->data + page->freeNext;
        ASAN_UNPOISON_MEMORY_REGION(block, page->blockSize);

        // IDA: *(_DWORD *)(v5 + 48) = v6 - *(_DWORD *)(v5 + 36)
        // page->freeNext -= page->blockSize
        page->freeNext -= page->blockSize;  // page->freeNext at offset 48, blockSize at offset 36
        
        // IDA: ++*(_DWORD *)(v5 + 52)
        page->busyBlocks++;  // page->busyBlocks at offset 52
    }
    else  // freeList path (page->freeNext < 0)
    {
        // IDA: v7 = *(__int64 **)(v5 + 40)  // page->freeList at offset 40
        block = page->freeList;  // page->freeList at offset 40
        ASAN_UNPOISON_MEMORY_REGION((char*)block + sizeof(GCheader), page->blockSize - sizeof(GCheader));

        // when separate block metadata is not used, free list link is stored inside the block data itself
        // IDA: *(_QWORD *)(v5 + 40) = *v7  // page->freeList = next block
        page->freeList = freegcolink(block);  // page->freeList at offset 40
        
        // IDA: ++*(_DWORD *)(v5 + 52)
        page->busyBlocks++;  // page->busyBlocks at offset 52
    }

    // if we allocate the last block out of a page, we need to remove it from free list
    // IDA: if (!*(_QWORD *)(v5 + 40) && *(int *)(v5 + 48) < 0)
    if (!page->freeList && page->freeNext < 0)  // offsets 40 and 48
    {
        // IDA: *(_QWORD *)(v4 + 464) = *(_QWORD *)(v5 + 24)
        // g->freegcopages[sizeClass] = page->next
        g->freegcopages[sizeClass] = page->next;  // page->next at offset 24
        
        // IDA: v8 = *(_QWORD *)(v5 + 24)
        // IDA: if (v8) *(_QWORD *)(v8 + 16) = 0
        if (page->next)
            page->next->prev = NULL;  // page->next at offset 24, prev at offset 16
            
        // IDA: *(_QWORD *)(v5 + 24) = 0
        page->next = NULL;  // page->next at offset 24
    }

    // IDA: return v7 + 1  // v7 points to block, +1 skips GC header
    return block;
}


We will be able to extract some sturct members like freegcopages, L->global, and almost the entirety of lua_Page struct.

v5 = *(_QWORD *)(v4 + 464);  // page = g->freegcopages[sizeClass] 

v8 = *(_QWORD *)(v5 + 24);     // page->next // 0x18
if ( v8 )
    *(_QWORD *)(v8 + 16) = 0;  // page->next->prev = NULL // 0x10

*(_DWORD *)(v5 + 48) = v6 - *(_DWORD *)(v5 + 36);  // page->freeNext -= page->blockSize // 0x24

v6 = *(int *)(v5 + 48);           // freeNext = page->freeNext
if ( (int)v6 < 0 )                // if (page->freeNext < 0) freenext is 0x30

++*(_DWORD *)(v5 + 52);           // page->busyBlocks++ is 0x34

v7 = (__int64 *)(v6 + v5 + 64);  // block = &page->data + page->freeNext 0x40

struct lua_Page {
    lua_Page* listprev; //no here
    lua_Page* listnext;     // 0x08
    lua_Page* prev;         // 0x10 
    lua_Page* next;         // 0x18 
    int pageSize;           // 0x20
    int blockSize;          // 0x24 
    void* freeList;         / 0x28 
    int freeNext;           // 0x30 
    int busyBlocks;         // 0x34 
    char padding[8];// padding
    char data[1];           // 0x40 
};

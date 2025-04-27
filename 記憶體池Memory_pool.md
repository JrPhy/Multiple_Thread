雖然現在記憶體價錢已經很低，但在一些 IOT 或是傳感器上記憶體的量還是很少，所以除了算法要去優化外，記憶體也不可能一直申請在釋放，通常一次要就會要一大塊，然後在整個一起還回去來降低開銷，也可以避免記憶體碎片化。當然我們也可以直接在 STACK 上直接開一塊很大的記憶體，但通常 STACK 預設很小，所以一般會在 HEAP 上去申請。而實際情況還需要考慮多線程的情況，這邊就先從單線程的例子開始，在漸漸地往多線程。
 
## 一、單線程記憶體池
雖然我們是一次要一大塊 Memory，但這一大塊中還會再切分成很多小塊 block_count，其中每一小塊大小為 block_size，最後並將頭尾連接，這樣就可以比較方便去控管每一小塊的使用情況，其中 block_size 會要求是 2 的整數次方，例如 8, 16, 32, 等數字。如果新進來的資料還可以放進可用塊中，那就先使用可用塊，否則就往下一小塊中去尋找。
#### 1. 資料結構
```C
typedef struct MemoryBlock {
    struct MemoryBlock* next;
} MemoryBlock;

typedef struct {
    MemoryBlock* free_list;  // 自由列表（可用塊）
    char* pool_start;        // 內存池起始指針
    size_t block_size;       // 內存塊大小（可由使用者決定）
    size_t block_count;      // 內存池內存塊數量
    size_t offset;           // 當前分配位置（用於線性分配）
} MemoryPool;
```
所以我們申請的總大小為 block_size*block_count，offset 則是看目前申請的記憶體位置是否還足夠用，如果不夠就需要再去申請另外一塊記憶體，並且將申請的這兩塊用 linked list 的方式連接起來
#### 2. 初始化
```C
bool init_pool(MemoryPool* mp, size_t block_size, size_t block_count) {
    mp->pool_start = (char*)malloc(block_size * block_count);
    if (!mp->pool_start) return false;

    mp->block_size = block_size;
    mp->block_count = block_count;
    mp->offset = 0;
    mp->free_list = NULL;
    return true;
}
```
其中 ```mp->free_list = NULL;``` 是因為還沒使用，當有已申請的記憶體被釋放後，free_list 就會指向該塊被釋放的記憶體，並將多塊被釋放的記憶體串接起來。
#### 3. 將資料放入 POOL
```C
void* pool_alloc(MemoryPool* mp) {
    if (mp->free_list) {
        // 使用自由列表中的可用塊
        MemoryBlock* allocated = mp->free_list;
        mp->free_list = allocated->next;
        return allocated;
    } else if (mp->offset + mp->block_size <= mp->block_size * mp->block_count) {
        // 線性分配新的塊
        void* ptr = mp->pool_start + mp->offset;
        mp->offset += mp->block_size;
        return ptr;
    }
    return NULL; // 內存池已滿
}
```
如果得到 NULL，那就必須去在要一塊，否則就可以一直將資料放入。
#### 4. 釋放某一塊記憶體
```C
void pool_free(MemoryPool* mp, void* block) {
    if (!block) return;
    MemoryBlock* freed = (MemoryBlock*)block;
    freed->next = mp->free_list;
    mp->free_list = freed; // 把釋放的塊放回自由列表
}
```
當某一塊記憶體已用完時我們換需要將其釋放，所以必須把該塊記憶體的位置傳入，清空之後就可以重新使用該塊記憶體。
#### 5. 重置與銷毀
```C
void pool_reset(MemoryPool* mp) {
    mp->free_list = NULL; // 清空自由列表
    mp->offset = 0;       // 重置線性分配
}

void destroy_pool(MemoryPool* mp) {
    free(mp->pool_start);
}
```
最後的使用就是
```C
int main() {
    MemoryPool mp;
    size_t user_defined_block_size = 8; // 由使用者決定塊大小
    size_t user_defined_block_count = 32; // 由使用者決定塊數量
    if(!init_pool(&mp, user_defined_block_size, user_defined_block_count)) {
        printf("內存池初始化失敗\n");
        return 1;
    };

    int *a = (int *)pool_alloc(&mp);
    if (a) {
        *a = 42; // 賦值
        printf("Allocated value: %d, %p\n", *a, a);
    }
    int *b = (int *)pool_alloc(&mp);
    if (b) {
        *b = 42; // 賦值
        printf("Allocated value: %d, %p\n", *b, b);
    }
    int *c = (int *)pool_alloc(&mp);
    if (c) {
        *c = 42; // 賦值
        printf("Allocated value: %d, %p\n", *c, c);
    }
    pool_free(&mp, b); // **釋放 a**
    void* d = pool_alloc(&mp); // **重新使用 a 的位置**
    printf("重新分配的內存: %p\n", d);
    
    pool_reset(&mp);
    destroy_pool(&mp);

    return 0;
}
```
## 二、有鎖多線程記憶體池
多數情況會用到多線程來申請記憶體，在此可以來時做有鎖與無鎖的版本。對於有鎖的情況就會使用互斥鎖，畢竟要先等某個線程申請完並釋放鎖後，下一個線程才能去申請。所以在結構與初始化就先加入一個鎖
```C
...
    pthread_mutex_t lock;    // 互斥鎖
} MemoryPool;
```
```C
bool init_pool(MemoryPool* mp, size_t block_size, size_t block_count) {
    ...
    pthread_mutex_init(&mp->lock, NULL);  // 初始化互斥鎖
    return true;
}
```
當有線程要申請時，要把申請的部分都當成臨界區來保護
```C
void* pool_alloc(MemoryPool* mp) {
    pthread_mutex_lock(&mp->lock);
    ... // 原程式碼
    pthread_mutex_unlock(&mp->lock);
    return ptr;
}
```
而釋放時如果不會有兩線程同時去取用同一段記憶體，那麼就不需要保護，否則也是跟申請一樣需要將整段保護起來
```C
void pool_reset(MemoryPool* mp) {
    pthread_mutex_lock(&mp->lock);
    ... // 原程式碼
    pthread_mutex_unlock(&mp->lock);
}
```
最後銷毀記憶體池時記得也要銷毀互斥鎖
```C
void destroy_pool(MemoryPool* mp) {
    pthread_mutex_lock(&mp->lock);
    free(mp->pool_start);
    pthread_mutex_unlock(&mp->lock);
    pthread_mutex_destroy(&mp->lock);  // 銷毀互斥鎖
}
```

## 三、無鎖多線程記憶體池
相較於有鎖的版本，無鎖的是使用原子操作來實作，大部分情況會有比較高的效能，但若要保護的區段太複雜，則無法用原子操作來達成。首先我們需要先看結構中的變數那些有可能會被同時存取，當有使用者要來申請時，就會需要改變 offset，而當池滿了就需要再開一個池，所以需要改成原子變數的即為 offset 與 free_list，其餘仍保持一般變數。
```C
typedef struct MemoryBlock {
    struct MemoryBlock* next;
} MemoryBlock;

typedef struct {
    atomic_uintptr_t free_list;// 自由列表（可用塊）
    char* pool_start;        // 內存池起始指針
    size_t block_size;       // 內存塊大小（可由使用者決定）
    size_t block_count;      // 內存池內存塊數量
    atomic_size_t offset;   // 當前分配位置（用於線性分配）
} MemoryPool;
```
初始化時這兩個變數就需要用 atomic_store 初始化
```C
bool init_pool(MemoryPool* mp, size_t block_size, size_t block_count) {
    ...
    atomic_store(&mp->offset, 0);
    atomic_store(&mp->free_list, (uintptr_t)NULL);
    return true;
}
```
重置記憶體池時也是使用 atomic_store
```C
void pool_reset(MemoryPool* mp) {
    atomic_store(&mp->free_list, (uintptr_t)NULL);
    atomic_store(&mp->offset, 0);
}
```
而申請時會比較麻煩，因為有可能在申請同時也有被釋放，所以會使用 ```atomic_compare_exchange_weak``` 來去判斷 free_list，如果有新的則會去更新 free_list，offset 也是一樣
```C
void* pool_alloc(MemoryPool* mp) {
    MemoryBlock* old_head = (MemoryBlock*)atomic_load(&mp->free_list);
    
    // 嘗試使用自由列表中的可用塊
    while (old_head &&
            !atomic_compare_exchange_weak(&mp->free_list, 
                                         (uintptr_t*)&old_head, 
                                         (uintptr_t)old_head->next)) {
        old_head = (MemoryBlock*)atomic_load(&mp->free_list);
    }
    
    if (old_head) return old_head;
    // 嘗試線性分配新的塊
    size_t current_offset = atomic_load(&mp->offset);
    size_t new_offset = current_offset + mp->block_size;

    if (new_offset <= mp->block_size * mp->block_count &&
        atomic_compare_exchange_weak(&mp->offset, &current_offset, new_offset)) {
        return mp->pool_start + current_offset;
    }
    return NULL; // 內存池已滿
}
```
釋放時也是一樣
```C
void pool_free(MemoryPool* mp, void* block) {
    if (!block) return;
    MemoryBlock* freed = (MemoryBlock*)block;

    MemoryBlock* old_head = (MemoryBlock*)atomic_load(&mp->free_list);
    do {
        freed->next = old_head;
    } while (!atomic_compare_exchange_weak(&mp->free_list, (uintptr_t*)&old_head, (uintptr_t)freed));
}
```
若 ```atomic_compare_exchange_weak``` 反覆失敗，線程可能會陷入忙等待，浪費 CPU 資源。
## 四、鎖與原子操作的比較
|  | mutex | atomic |
| --- | --- | --- |
| 優點 | 容易理解和使用 | 效能較高 |
| 優點 | 適用於複雜的臨界區 | 無死鎖問題 |
| 優點 | 防止忙等待 |  |
|  |  |  |
| 缺點 | 開銷較高 | 程式碼較複雜 |
| 缺點 | 可能有死鎖 | 不適合複雜的臨界區 |
| 缺點 |  | 可能導致忙等待 |

主要在於臨界區是否複雜，若不複雜則使用 atomic，反之則用 mutex

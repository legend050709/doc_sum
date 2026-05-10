```table-of-contents
```
# 介绍
LRU（Least Recently Used，最近最少使用）是一种经典的缓存淘汰策略。当缓存空间满时，优先淘汰最近最少被访问的数据项。
**核心思想**：如果一个数据最近被访问过，那么它将来被访问的概率也更高；反之，很久没被访问的数据，将来被访问的概率更低。

# 为什么需要 LRU

在实际系统中，**缓存容量总是有限**的。LRU 提供了一种高效、直觉合理的淘汰策略：

- **命中率较高**：局部性原理，符合"最近访问的数据更可能再次被访问"的时间局部性原理
    
- **实现高效**：O(1) 时间完成所有操作（查找、插入、更新、淘汰）
    
- **通用性强**：广泛应用于 CPU 缓存、数据库缓冲池、Web 缓存、操作系统页面置换、redis/memcache等

# 原理
# 目标
查找快，插入快，删除快，有顺序之分。

# 设计
因为显然 cache 必须有顺序之分，以区分最近使用的和久未使用的数据；而且我们要在 cache 中查找键是否已存在；如果容量满了要删除最后一个数据；每次访问还要把数据插入到队头。

那么，什么数据结构同时符合上述条件呢？哈希表查找快，但是数据无固定顺序；
链表有顺序之分，插入删除快，但是查找慢。

所以结合一下，形成一种新的数据结构：**哈希表 + 双向链表**。
```bash
hash表：查找快；
双向链表：插入快，删除快，有顺序之分。
```

![](attachments/Pasted%20image%2020260427105655.png)

如果存储满了，可以通过 O(1) 的时间淘汰掉双向链表的尾部，每次新增和访问数据，都可以通过 O(1)的效率把新的节点增加到对头，或者把已经存在的节点移动到队头。

# 流程
## 哈希表 + 双向链表

![](attachments/Pasted%20image%2020260427121636.png)

哨兵节点标注：
- DH = dummy_head（左哨兵，蓝色）：不存数据，是所有真实节点的"左墙"
- DT = dummy_tail（右哨兵，紫色）：不存数据，是所有真实节点的"右墙"    
- S = 哈希桶哨兵（每桶一个）：不存有效数据，是冲突链的"左墙"
- 绿色节点 = 最近访问（MRU），红色节点 = 最久未访问（LRU，淘汰候选）


关键关系：
- 最近访问节点 = dummy_head->next（第一个真实节点）
- 最久未访问节点 = dummy_tail->prev（最后一个真实节点，淘汰候选）
- 空链表时：dummy_head->next == dummy_tail（两个哨兵紧挨）

## lru_get(cache, key) — 查找流程

![](attachments/Pasted%20image%2020260427113806.png)

范例：
![](attachments/Pasted%20image%2020260427121841.png)

## lru_put(cache, key, value) — 插入/更新流程

![](attachments/Pasted%20image%2020260427113939.png)

示例（容量=3，已满后 put(4, 400)）：
![](attachments/Pasted%20image%2020260427121927.png)

# 实现
## 数据结构
```c
#ifndef LRU_CACHE_H
#define LRU_CACHE_H

#include <stddef.h>

/**
 * 双向链表节点
 *
 * 在双哨兵设计中：
 *   dummy_head 是链表左端哨兵（不存数据）
 *   dummy_tail 是链表右端哨兵（不存数据）
 *   所有真实节点位于两个哨兵之间：
 *     [dummy_head] ⇄ [real_node_1] ⇄ ... ⇄ [real_node_N] ⇄ [dummy_tail]
 *
 *   最近访问节点 = dummy_head->next（第一个真实节点）
 *   最久未访问节点 = dummy_tail->prev（最后一个真实节点，淘汰候选）
 */
typedef struct ListNode {
    int key;                /* 缓存键（哨兵节点此字段无意义） */
    int value;              /* 缓存值（哨兵节点此字段无意义） */
    struct ListNode *prev;  /* 前驱指针 */
    struct ListNode *next;  /* 后继指针 */
} ListNode;

/**
 * 哈希表节点
 *
 * 在哨兵头设计中：
 *   每个桶的第一个 HashNode 是哨兵（不存有效数据）
 *   所有真实 HashNode 在哨兵之后：
 *     bucket[i] → [dummy_HashNode] → [real_HashNode_1] → ... → NULL
 *
 *   优点：删除操作不再需要判断目标是否为冲突链首节点
 *         prev->next = cur->next 始终有效（prev 至少是哨兵）
 */
typedef struct HashNode {
    int key;                /* 哈希键（哨兵节点此字段无意义） */
    ListNode *list_node;    /* 指向双向链表中对应的节点；相当于是value */
    struct HashNode *next;  /* 哈希冲突链 */
} HashNode;

/**
 * LRU Cache 主结构
 */
typedef struct LRUCache {
    int capacity;           /* 缓存容量上限 */
    int size;               /* 当前缓存中的元素数量 */
    ListNode *dummy_head;   /* 链表左哨兵（最近访问端） */
    ListNode *dummy_tail;   /* 链表右哨兵（最久未访问端） */
    HashNode **hash_table;  /* 哈希表数组（每个桶带头哨兵） */
    int hash_size;          /* 哈希表桶数量 */
} LRUCache;


/* 公共 API */
LRUCache *lru_create(int capacity);
void lru_destroy(LRUCache *cache);
int lru_get(LRUCache *cache, int key);
int lru_put(LRUCache *cache, int key, int value);
void lru_print(LRUCache *cache);
int lru_size(LRUCache *cache);

#endif /* LRU_CACHE_H */
```

## 链表操作

```c
static ListNode *list_create_node(int key, int value)
{
    ListNode *node = (ListNode *)malloc(sizeof(ListNode));
    if (!node) {
        fprintf(stderr, "[LRU] malloc failed for ListNode (key=%d)\n", key);
        return NULL;
    }
    node->key   = key;
    node->value = value;
    node->prev  = NULL;
    node->next  = NULL;
    return node;
}


/**
 * list_insert_after - 在指定节点之后插入新节点
 * 操作示意：
 *   [target] ⇄ [next]
 *   => [target] ⇄ [node] ⇄ [next]
 */
static void list_insert_after(ListNode *target, ListNode *newNode)
{
    newNode->prev       = target;
    newNode->next       = target->next;
    target->next->prev  = newNode;
    target->next        = newNode;
}


/**
 * list_remove_node - 从链表中摘除指定节点
 *
 * 操作示意：
 *   [prev] ⇄ [node] ⇄ [next]
 *   => [prev] ⇄ [next]，node 被摘除
 */
static void list_remove_node(ListNode *node)
{
    node->prev->next = node->next;
    node->next->prev = node->prev;
    node->prev = NULL;
    node->next = NULL;
}

/**
 * list_move_to_head - 将已有节点移到链表头部
 */
static void list_move_to_head(LRUCache *cache, ListNode *node)
{
    list_remove_node(node);
    list_insert_after(cache->dummy_head, node);
}

/**
 * list_remove_tail - 淘汰链表尾部节点（最久未访问）
 *
 * 返回：被摘除的节点指针（调用方负责释放内存）
 */
static ListNode *list_remove_tail(LRUCache *cache)
{
    ListNode *tail_node = cache->dummy_tail->prev;

    /* 如果 tail_node == dummy_head，说明链表为空（两个哨兵紧挨） */
    if (tail_node == cache->dummy_head) {
        return NULL;
    }

    list_remove_node(tail_node);
    return tail_node;
}
```

## hash 操作
```c
static int hash_func(int key, int hash_size)
{
    return ((key < 0) ? (-key) : key) % hash_size;
}


/**
 * hash_find - 在哈希表中查找 key 对应的节点
 *
 * 哨兵设计下的查找流程：
 *   1. 计算 hash 桶编号
 *   2. 从该桶的哨兵节点之后开始遍历（跳过哨兵）
 *   3. 在冲突链中逐个比较 key
 *   4. 找到则返回 HashNode 指针，否则返回 NULL
 */
static HashNode *hash_find(LRUCache *cache, int key)
{
    int idx = hash_func(key, cache->hash_size);
    /* 从哨兵之后的第一个真实节点开始遍历 */
    HashNode *cur = cache->hash_table[idx]->next;

    while (cur) {
        if (cur->key == key) {
            return cur;
        }
        cur = cur->next;
    }
    return NULL;
}

/**
 * hash_insert - 在哈希表中插入新节点
 *
 * 哨兵设计下的插入操作：
 *   直接在桶哨兵之后插入（头插法）
 *   不需要判断桶是否为空，因为桶哨兵始终存在
 *
 * 操作示意：
 *   [dummy] → [old_first] → ...
 *   => [dummy] → [new_node] → [old_first] → ...
 */
static int hash_insert(LRUCache *cache, int key, ListNode *list_node)
{
    HashNode *hnode = (HashNode *)malloc(sizeof(HashNode));
    if (!hnode) {
        fprintf(stderr, "[LRU] malloc failed for HashNode (key=%d)\n", key);
        return -1;
    }
    hnode->key       = key;
    hnode->list_node = list_node;

    int idx = hash_func(key, cache->hash_size);
    /* 在桶哨兵之后插入（头插法） */
    HashNode *dummy = cache->hash_table[idx];
    hnode->next   = dummy->next;
    dummy->next   = hnode;

    return 0;
}

/**
 * hash_remove - 从哈希表中移除指定 key
 *
 * 哨兵设计下的删除操作：
 *   prev 初始化为桶哨兵，遍历时 prev 前移
 *   找到目标后，prev->next = cur->next 始终有效
 *   不需要判断 cur 是否为冲突链首节点
 */
static int hash_remove(LRUCache *cache, int key)
{
    int idx = hash_func(key, cache->hash_size);
    HashNode *prev = cache->hash_table[idx];  /* prev 初始化为桶哨兵 */
    HashNode *cur  = prev->next;              /* cur 为哨兵之后的第一个真实节点 */

    while (cur) {
        if (cur->key == key) {
            /* prev->next = cur->next 始终有效 */
            prev->next = cur->next;
            free(cur);
            return 0;
        }
        prev = cur;
        cur  = cur->next;
    }
    return -1; /* key 不存在 */
}
```
## LRU Cache 创建
```c
/**
 * lru_create - 创建 LRU Cache
 *
 * 哨兵初始化：
 *   1. 创建 dummy_head 和 dummy_tail 哨兵节点
 *   2. dummy_head->next = dummy_tail, dummy_tail->prev = dummy_head
 *      形成初始的双哨兵环：[DH] ⇄ [DT]
 *   3. 为哈希表每个桶创建哨兵 HashNode
 */
LRUCache *lru_create(int capacity)
{
    if (capacity <= 0) {
        fprintf(stderr, "[LRU] capacity must be > 0 (got %d)\n", capacity);
        return NULL;
    }

    LRUCache *cache = (LRUCache *)malloc(sizeof(LRUCache));
    if (!cache) {
        fprintf(stderr, "[LRU] malloc failed for LRUCache\n");
        return NULL;
    }

    cache->capacity = capacity;
    cache->size     = 0;

    /* 创建双向链表的哨兵节点 */
    cache->dummy_head = list_create_node(-1, -1);  /* key/value 无意义 */
    cache->dummy_tail = list_create_node(-2, -2);  /* key/value 无意义 */
    if (!cache->dummy_head || !cache->dummy_tail) {
        free(cache->dummy_head);
        free(cache->dummy_tail);
        free(cache);
        return NULL;
    }

    /* 初始状态：两个哨兵紧挨，中间无真实节点
     * [dummy_head] ⇄ [dummy_tail]
     */
    cache->dummy_head->next = cache->dummy_tail;
    cache->dummy_head->prev = NULL;
    cache->dummy_tail->prev = cache->dummy_head;
    cache->dummy_tail->next = NULL;

    /* 创建哈希表（每个桶带哨兵） */
    cache->hash_size = capacity * 2 + 1;
    cache->hash_table = (HashNode **)calloc(cache->hash_size, sizeof(HashNode *));
    if (!cache->hash_table) {
        fprintf(stderr, "[LRU] calloc failed for hash_table (size=%d)\n",
                cache->hash_size);
        free(cache->dummy_head);
        free(cache->dummy_tail);
        free(cache);
        return NULL;
    }

    /* 为每个桶创建哨兵 HashNode */
    for (int i = 0; i < cache->hash_size; i++) {
        HashNode *dummy_hn = (HashNode *)malloc(sizeof(HashNode));
        if (!dummy_hn) {
            /* 创建失败，释放已创建的哨兵 */
            for (int j = 0; j < i; j++) {
                free(cache->hash_table[j]);
            }
            free(cache->hash_table);
            free(cache->dummy_head);
            free(cache->dummy_tail);
            free(cache);
            return NULL;
        }
        dummy_hn->key       = -1;      /* 哨兵的 key 无意义 */
        dummy_hn->list_node = NULL;     /* 哨兵不指向任何 ListNode */
        dummy_hn->next      = NULL;     /* 哨兵之后的冲突链初始为空 */
        cache->hash_table[i] = dummy_hn;
    }

    return cache;
}
```

## LRU get实现：查找
```c
/**
 * lru_put - 插入或更新缓存项
 *
 * CASE 1: key 已存在 → 更新 value，移到头部
 * CASE 2: key 不存在 → 若满则淘汰尾部，创建新节点插入头部
 */
int lru_put(LRUCache *cache, int key, int value)
{
    if (!cache) {
        return -1;
    }

    HashNode *hnode = hash_find(cache, key);


    if (hnode) {
        /* CASE 1: key 已存在，更新 value 并移到头部 */
        ListNode *lnode = hnode->list_node;
        lnode->value = value;
        list_move_to_head(cache, lnode);
        return 0;
    }

    /* CASE 2: key 不存在，需要插入新节点 */

    /* 缓存已满，先淘汰尾部节点 */
    if (cache->size >= cache->capacity) {
        ListNode *tail_node = list_remove_tail(cache);
        if (tail_node) {
            hash_remove(cache, tail_node->key);
            free(tail_node);
            cache->size--;
        }
    }

    /* 创建新 ListNode 并插入 dummy_head 之后 */
    ListNode *new_node = list_create_node(key, value);
    if (!new_node) {
        return -1;
    }
    list_insert_after(cache->dummy_head, new_node);

    /* 在哈希表中插入映射 */
    if (hash_insert(cache, key, new_node) != 0) {
        /* 哈希插入失败，回滚链表操作 */
        list_remove_node(new_node);
        free(new_node);
        return -1;
    }

    cache->size++;
    return 0;
}
```

# 扩展思考
## 线程安全

当前实现不含锁机制。多线程环境下使用需外部加锁，常见方案：

- 全局锁：简单但性能受限
- 分段锁：将哈希表分为多个段，每段独立锁，提高并发度
- 读写锁：get 操作用读锁，put 操作用写锁
    
## 性能优化
- 哈希函数：可替换为更均匀的哈希函数（如 MurmurHash）
- 预分配内存池：预分配 ListNode/HashNode 池，避免频繁 malloc/free
- NUMA 友好：按 NUMA 节点分配内存，减少跨节点访问延迟

# 参考
```bash
```
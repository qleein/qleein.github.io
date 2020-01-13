---
layout: post
title:  "PHP7中HashTable实现"
date:   2016-11-06 23:04:00 +0800
tags: PHP 哈希
categories: 日常记录
---

## 结构定义

HashTable结构定义在zend\_types.h中 

```c
typedef struct _zend_array HashTable;

struct _zend_array {
	zend_refcounted_h gc;
	union {
		struct {
			ZEND_ENDIAN_LOHI_4(
				zend_uchar    flags,
				zend_uchar    nApplyCount,
				zend_uchar    nIteratorsCount,
				zend_uchar    reserve)
		} v;
		uint32_t flags;
	} u;
	uint32_t          nTableMask;
	Bucket           *arData;
	uint32_t          nNumUsed;
	uint32_t          nNumOfElements;
	uint32_t          nTableSize;
	uint32_t          nInternalPointer;
	zend_long         nNextFreeElement;
	dtor_func_t       pDestructor;
};
```

重要的几个字段：

nTableSize: 哈系表最多容纳元素数量，归整为2的幂。

nTableMask: 掩码， 对key求哈系得到的值h， h | nTableMask得到偏移量。

arData:存储实际数据。

arData指向的数据实际分为两部分，Hash部分和Data部分。

                                                            ht->arData

-----------------Hash部分------------------------|-------------------Data部分--------------------

{Hash部分}{arData指针}{Data部分}。

哈希运算不可避免会产生冲突，有两个以上的key映射到同一位置， PHP采用的是链地址法，即把冲突的键值用链表连接起来。链地址法会有内存碎片，还需要不断分配释放内存，所以这里直接分配了nTableSize个Bucket空间，存放所有的key-val对。

所以这里分配了两个空间，Data部分存储key-val对，Hash部分存储映射对某个位置的元素链表的头部。

## 初始化

zend\_hash\_init对HashTable进行初始化赋值。这是一个宏，实际调用\_zend\_hash\_init.

ht->nTableSize存放取整后的nSize,此时没有为哈希表元素分配内存。 添加元素时通过ht->u.flags & HASH_FLAG_INITIALIZED判断为false时分配内存。

zend\_hash\_check\_size(nSize)  将nsize取整为2的幂。

```c
#define zend_hash_init(ht, nSize, pHashFunction, pDestructor, persistent)   \
     _zend_hash_init((ht), (nSize), (pDestructor), (persistent) ZEND_FILE_LINE_CC)

ZEND_API void ZEND_FASTCALL _zend_hash_init(HashTable *ht, uint32_t nSize, dtor_func_t pDestructor, zend_bool persistent ZEND_FILE_LINE_DC)
{
	GC_REFCOUNT(ht) = 1;
	GC_TYPE_INFO(ht) = IS_ARRAY;
	ht->u.flags = (persistent ? HASH_FLAG_PERSISTENT : 0) | HASH_FLAG_APPLY_PROTECTION | HASH_FLAG_STATIC_KEYS;
	ht->nTableSize = zend_hash_check_size(nSize);
	ht->nTableMask = HT_MIN_MASK;
	HT_SET_DATA_ADDR(ht, &uninitialized_bucket);
	ht->nNumUsed = 0;
	ht->nNumOfElements = 0;
	ht->nInternalPointer = HT_INVALID_IDX;
	ht->nNextFreeElement = 0;
	ht->pDestructor = pDestructor;
}
```

第一次向表中添加元素时调用， 分配空间。

```c
//第一次向表中添加元素时调用， 分配空间。
static void zend_always_inline zend_hash_real_init_ex(HashTable *ht, int packed)
{
	HT_ASSERT(GC_REFCOUNT(ht) == 1);
	ZEND_ASSERT(!((ht)->u.flags & HASH_FLAG_INITIALIZED));
	if (packed) {
		HT_SET_DATA_ADDR(ht, pemalloc(HT_SIZE(ht), (ht)->u.flags & HASH_FLAG_PERSISTENT));
		(ht)->u.flags |= HASH_FLAG_INITIALIZED | HASH_FLAG_PACKED;
		HT_HASH_RESET_PACKED(ht);
	} else {
		(ht)->nTableMask = -(ht)->nTableSize;
		HT_SET_DATA_ADDR(ht, pemalloc(HT_SIZE(ht), (ht)->u.flags & HASH_FLAG_PERSISTENT));
		(ht)->u.flags |= HASH_FLAG_INITIALIZED;
		if (EXPECTED(ht->nTableMask == -8)) {
			Bucket *arData = ht->arData;

			HT_HASH_EX(arData, -8) = -1;
			HT_HASH_EX(arData, -7) = -1;
			HT_HASH_EX(arData, -6) = -1;
			HT_HASH_EX(arData, -5) = -1;
			HT_HASH_EX(arData, -4) = -1;
			HT_HASH_EX(arData, -3) = -1;
			HT_HASH_EX(arData, -2) = -1;
			HT_HASH_EX(arData, -1) = -1;
		} else {
			HT_HASH_RESET(ht);
		}
	}
}
```

## 查找元素

HashTable如何查找元素：

```c
static zend_always_inline Bucket *zend_hash_find_bucket(const HashTable *ht, zend_string *key)
{
	zend_ulong h;
	uint32_t nIndex;
	uint32_t idx;
	Bucket *p, *arData;

    //计算hash。
	h = zend_string_hash_val(key);
	arData = ht->arData;
    //根据nTableMask取哈希值的后几位作为有效值，决定最终散列位置。
	nIndex = h | ht->nTableMask;
    /* #define HT_HASH_EX(data, idx)    ((uint32_t*)(data))[(int32_t)(idx)]
    //散列到（-nIndex）的元素组成一个链表，idx为此链表头的位置索引。
	idx = HT_HASH_EX(arData, nIndex);
	while (EXPECTED(idx != HT_INVALID_IDX)) {
        // p == arData[idx], p为散列到同一bucket的元素列表的首元素。通过while循环遍历查找。
		p = HT_HASH_TO_BUCKET_EX(arData, idx);
		if (EXPECTED(p->key == key)) { /* check for the same interned string */
			return p;
		} else if (EXPECTED(p->h == h) &&
		     EXPECTED(p->key) &&
		     EXPECTED(ZSTR_LEN(p->key) == ZSTR_LEN(key)) &&
		     EXPECTED(memcmp(ZSTR_VAL(p->key), ZSTR_VAL(key), ZSTR_LEN(key)) == 0)) {
			return p;
		}
		idx = Z_NEXT(p->val);
	}
	return NULL;
}
```


## 添加元素

通过\_zend\_hash\_add\_or\_update\_i添加元素。

```c
static zend_always_inline zval *_zend_hash_add_or_update_i(HashTable *ht, zend_string *key, zval *pData, uint32_t flag ZEND_FILE_LINE_DC);
```
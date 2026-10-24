---
layout: post
title: "Exploring the implementation of PHP array Part 1"
date: 2026-10-24 18:45
---

I decided to go have a look at how arrays are implemented in PHP. The source code is available here: <https://github.com/php/php-src>{:target="_blank" rel="noopener"} if you want to have a poke around yourself. 

There are two questions I want answered: how value lookup happens and how values are set. Since the answer turned out to be a little bit more complicated than I originally expected, this will be a two-part blog post with the first part covering packed arrays and the second covering non-packed arrays.

Packed arrays have 2 characteristics:
1. the keys are only integers
2. the keys are inserted in a growing order

In practice it would look something like this: 
```php
$array = [
	0 => 'foo',
	1 => 'foo2',
	2 => 'foo3',
];
```
Gaps to a certain size are allowed; that logic lives in the `_zend_hash_index_add_or_update_i` function but exploring that is a little out of scope for now.

## How a packed PHP array stores data
The array in PHP is represented by a struct called `HashTable`.
```c
typedef struct _zend_array HashTable;

struct _zend_array {
	zend_refcounted_h gc;
	union {
		struct {
			ZEND_ENDIAN_LOHI_4(
				uint8_t    flags,
				uint8_t    _unused,
				uint8_t    nIteratorsCount,
				uint8_t    _unused2)
		} v;
		uint32_t flags;
	} u;
	uint32_t          nTableMask;
	union {
		uint32_t     *arHash;   /* hash table (allocated above this pointer) */
		Bucket       *arData;   /* array of hash buckets */
		zval         *arPacked; /* packed array of zvals */
	};
	uint32_t          nNumUsed;
	uint32_t          nNumOfElements;
	uint32_t          nTableSize;
	uint32_t          nInternalPointer;
	zend_long         nNextFreeElement;
	dtor_func_t       pDestructor;
};
```

Inspecting the struct, you can see that our values are stored in a struct array of zvals called `arPacked` (since PHP version 8.2). 
```c
zval         *arPacked; /* packed array of zvals */
```
and our array of foos would be a regular C array of zvals with their index as the C array's index. Looking something like this (abbreviated):
```c
[0](value = pointer to a zend_string "foo", u1.type_info = IS_STRING)
[1](value = pointer to a zend_string "foo2", u1.type_info = IS_STRING)
[2](value = pointer to a zend_string "foo3", u1.type_info = IS_STRING)
```

## Lookup
When we write code like this: 
```php
echo($array[2]);
```

Packed array value lookup happens in the `ZEND_HASH_INDEX_FIND` macro.

```c
#define ZEND_HASH_INDEX_FIND(_ht, _h, _ret, _not_found) do { \
		if (EXPECTED(HT_IS_PACKED(_ht))) { \
			if (EXPECTED((zend_ulong)(_h) < (zend_ulong)(_ht)->nNumUsed)) { \
				_ret = &_ht->arPacked[_h]; \
				if (UNEXPECTED(Z_TYPE_P(_ret) == IS_UNDEF)) { \
					goto _not_found; \
				} \
			} else { \
				goto _not_found; \
			} \
		} else { \
			_ret = _zend_hash_index_find(_ht, _h); \
			if (UNEXPECTED(_ret == NULL)) { \
				goto _not_found; \
			} \
		} \
	} while (0)
```

Since this is a macro, not a function, it does not return a value; the mechanism used here is setting the value to `_ret`.
1.  `EXPECTED(HT_IS_PACKED(_ht))` checks if the HashTable (array) is packed
2. `EXPECTED((zend_ulong)(_h) < (zend_ulong)(_ht)->nNumUsed)` checks if index is in bounds
3.  `_ret = &_ht->arPacked[_h];` we get the value from arPacked array at our index

Every other code path here gotos to `_not_found`, which gives us the beloved error `Warning: Undefined array key 5` or calls `_zend_hash_index_find` if we have a non-packed array.

(You can't debug macros, so under watch you can see our `_ret`, set to `foo3`, and under variables our `_h`, `2`.)
![VS Code debugger stopped after ZEND_HASH_INDEX_FIND, with the watch showing "foo3"](/assets/images/php-array-zend-hash-index-find-debugger.png)
## Setting values
When we write code to set a value to an array:
```php
$array[] = "foo4";
```

From `zend_hash.c` the `_zend_hash_index_add_or_update_i` function is called. This function is rather large, and reading it from a blog post might be a bit annoying, so here's a GitHub link to it.
<https://github.com/php/php-src/blob/12ef787293fc0e234535fdbf72ce838156923331/Zend/zend_hash.c#L1271>{:target="_blank" rel="noopener"}

We drop into the `HT_IS_PACKED` branch:
```c
if (HT_IS_PACKED(ht)) {
```

We fail the first if check, because `h = 3`, and `nNumUsed = 3`, `3 < 3 = false`
```c
(flag & (HASH_ADD_NEW|HASH_ADD_NEXT)) != (HASH_ADD_NEW|HASH_ADD_NEXT)
		 && h < ht->nNumUsed
```

Then we pass this if, which basically checks if our index is in size of the allocated `HashTable`:
```c
} else if (EXPECTED(h < ht->nTableSize)) {
```

Then we calculate the position of our zval in the arPacked array:
```c
zv = ht->arPacked + h;
```

Then we fill any gaps that might have been created with undefined zvals:
```c
if ((flag & (HASH_ADD_NEW|HASH_ADD_NEXT)) != (HASH_ADD_NEW|HASH_ADD_NEXT)) {
	if (h > ht->nNumUsed) {
		zval *q = ht->arPacked + ht->nNumUsed;
		while (q != zv) { ZVAL_UNDEF(q); q++; }
	}
}
```

Then we update the `nNextFreeElement`, `nNumUsed` and increment the `nNumOfElements`:
```c
ht->nNextFreeElement = ht->nNumUsed = h + 1;
ht->nNumOfElements++;
```

Finally, we either set a null zval or copy our data into the zval:
```c
if (flag & HASH_LOOKUP) {
	ZVAL_NULL(zv);
} else {
	ZVAL_COPY_VALUE(zv, pData);
}

return zv;
```

In our case, we drop into the `ZVAL_COPY_VALUE` branch and return the zv or if we would call:
```php
$array[4] = 'foo5';
```
it would create the null zval and return it; the value is set later.

Now there are other cases that this function handles, for example:
1. The TableSize is too small, so the table needs to be resized before a value can be set.
2. There is already a value present, so we replace the value.

I encourage you to check it out yourself with a debugger. It's cool :D I created a small GitHub repo with Claude that contains a small README to help you quickly set it up.
<https://github.com/Kaurikallaste/php-src-playground>{:target="_blank" rel="noopener"}

tl;dr: packed arrays are not that complicated in PHP; it's just a C array of zvals.

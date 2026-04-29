---
title: "LRU 캐시: 해시 테이블 + 이중 연결 리스트의 조합"
date: 2026-01-07
tags: [data-structure, CS, cache, linked-list, hash-table]
permalink: /lru-cache-internals/
excerpt: "LRU 캐시는 왜 해시 테이블과 이중 연결 리스트를 동시에 쓰는가? O(1) get/put을 달성하는 내부 구조를 C 코드로 파헤친다."
---

> 이 글은 [자료구조 1편: 선형 구조](/ds-1-linear/)(연결 리스트)와 [2편: 해시와 트리](/ds-2-hash-tree/)(해시 테이블)의 심화 문서다.

## LRU 캐시란

LRU(Least Recently Used) 캐시는 용량이 가득 차면 **가장 오래 사용되지 않은 항목을 제거**하는 캐시다. 운영체제의 페이지 교체, Redis의 메모리 정책, CDN 캐시 등 어디에서나 쓰인다.

두 가지 연산이 필요하다:
- **get(key):** 키에 해당하는 값을 반환. 해당 항목을 "최근 사용"으로 갱신
- **put(key, value):** 키-값 저장. 용량 초과 시 가장 오래된 항목 제거

**둘 다 O(1)**이어야 한다. 이 요구사항이 자료구조 선택을 결정한다.

## 왜 단일 자료구조로는 안 되는가

**배열**은 인덱스 접근은 O(1)이지만, 키로 탐색하려면 O(n)이고 가장 오래된 항목을 제거하는 것도 O(n)이다. **해시 테이블**은 get/put이 O(1)이지만 "가장 오래 사용되지 않은 것"이 무엇인지 알 방법이 없어서 제거가 O(n)이다. **큐/연결 리스트**는 가장 오래된 것의 제거는 O(1)이지만, 키로 특정 항목을 찾는 게 O(n)이다. **BST**는 모든 연산이 O(log n)이지만, 요구사항은 O(1)이다.

어떤 단일 자료구조도 세 연산(get, put, 제거)을 모두 O(1)에 해결하지 못한다.

## 해법: 두 자료구조의 조합

**해시 테이블**은 키로 O(1) 탐색을 제공하고, **이중 연결 리스트**는 O(1) 삽입/삭제/순서 관리를 제공한다. 둘을 결합하면:

```
해시 테이블                     이중 연결 리스트 (사용 순서)
┌────────┐
│ key: A ─┼──────→ [A] ← head (가장 최근)
│ key: B ─┼──────→ [B]
│ key: C ─┼──────→ [C]
│ key: D ─┼──────→ [D] ← tail (가장 오래됨)
└────────┘
```

- **get(key):** 해시 테이블에서 O(1)로 노드를 찾고, 그 노드를 연결 리스트의 head로 옮긴다 (O(1))
- **put(key, value):** 해시 테이블에 삽입(O(1)), 연결 리스트 head에 추가(O(1))
- **용량 초과 시:** 연결 리스트의 tail(가장 오래된 것)을 제거(O(1)), 해시 테이블에서도 삭제(O(1))

**모든 연산이 O(1)**이다.

## 핵심: 연결 리스트 노드를 옮기는 것이 O(1)인 이유

이중 연결 리스트에서 **노드의 포인터를 이미 알고 있으면** 삭제와 삽입이 O(1)이다. 해시 테이블의 value가 바로 그 노드의 포인터다. 탐색 없이 곧바로 포인터 4개만 조작하면 된다:

```
삭제: node->prev->next = node->next;
      node->next->prev = node->prev;

head 앞에 삽입: node->next = head->next;
               node->prev = head;
               head->next->prev = node;
               head->next = node;
```

이것이 이중 연결 리스트가 **다른 자료구조의 부품으로** 쓰이는 전형적인 패턴이다.

## C 구현

```c
#include <stdlib.h>
#include <string.h>

/* --- 이중 연결 리스트 노드 --- */
typedef struct LRUNode {
    int key;
    int value;
    struct LRUNode *prev;
    struct LRUNode *next;
} LRUNode;

/* --- 해시 테이블 엔트리 (체이닝) --- */
typedef struct HTNode {
    int key;
    LRUNode *lru_node;      /* 연결 리스트 노드의 포인터 */
    struct HTNode *next;     /* 해시 체인 */
} HTNode;

/* --- LRU 캐시 --- */
typedef struct {
    int capacity;
    int size;
    LRUNode head;            /* 센티널: 가장 최근 쪽 */
    LRUNode tail;            /* 센티널: 가장 오래된 쪽 */
    HTNode **buckets;        /* 해시 테이블 */
    int ht_cap;
} LRUCache;

LRUCache *lru_create(int capacity) {
    LRUCache *c = calloc(1, sizeof(LRUCache));
    c->capacity = capacity;
    c->size = 0;
    c->head.next = &c->tail;
    c->tail.prev = &c->head;
    c->ht_cap = capacity * 2;
    c->buckets = calloc(c->ht_cap, sizeof(HTNode *));
    return c;
}

/* --- 연결 리스트 조작 (O(1)) --- */

static void dll_remove(LRUNode *node) {
    node->prev->next = node->next;
    node->next->prev = node->prev;
}

static void dll_push_front(LRUCache *c, LRUNode *node) {
    node->next = c->head.next;
    node->prev = &c->head;
    c->head.next->prev = node;
    c->head.next = node;
}

static void dll_move_to_front(LRUCache *c, LRUNode *node) {
    dll_remove(node);
    dll_push_front(c, node);
}

/* --- 해시 테이블 조작 --- */

static int ht_idx(LRUCache *c, int key) {
    return ((unsigned int)key) % c->ht_cap;
}

static LRUNode *ht_find(LRUCache *c, int key) {
    int idx = ht_idx(c, key);
    for (HTNode *h = c->buckets[idx]; h; h = h->next)
        if (h->key == key) return h->lru_node;
    return NULL;
}

static void ht_put(LRUCache *c, int key, LRUNode *node) {
    int idx = ht_idx(c, key);
    HTNode *h = malloc(sizeof(HTNode));
    h->key = key;
    h->lru_node = node;
    h->next = c->buckets[idx];
    c->buckets[idx] = h;
}

static void ht_remove(LRUCache *c, int key) {
    int idx = ht_idx(c, key);
    HTNode **pp = &c->buckets[idx];
    while (*pp) {
        if ((*pp)->key == key) {
            HTNode *del = *pp;
            *pp = del->next;
            free(del);
            return;
        }
        pp = &(*pp)->next;
    }
}

/* --- 공개 API --- */

int lru_get(LRUCache *c, int key) {
    LRUNode *node = ht_find(c, key);
    if (!node) return -1;            /* 캐시 미스 */
    dll_move_to_front(c, node);      /* "최근 사용"으로 갱신 */
    return node->value;
}

void lru_put(LRUCache *c, int key, int value) {
    LRUNode *node = ht_find(c, key);

    if (node) {
        /* 이미 존재 → 값 갱신 + 앞으로 이동 */
        node->value = value;
        dll_move_to_front(c, node);
        return;
    }

    /* 용량 초과 → tail(가장 오래된 것) 제거 */
    if (c->size == c->capacity) {
        LRUNode *victim = c->tail.prev;   /* 센티널 바로 앞 = 가장 오래된 것 */
        ht_remove(c, victim->key);
        dll_remove(victim);
        free(victim);
        c->size--;
    }

    /* 새 노드 생성 → head(가장 최근) 위치에 삽입 */
    node = malloc(sizeof(LRUNode));
    node->key = key;
    node->value = value;
    dll_push_front(c, node);
    ht_put(c, key, node);
    c->size++;
}
```

## 동작 흐름 예시

capacity = 3인 LRU 캐시에서:

```
put(1,10)  리스트: [1] ← head
put(2,20)  리스트: [2] → [1]
put(3,30)  리스트: [3] → [2] → [1]

get(1)     1을 head로 이동
           리스트: [1] → [3] → [2]

put(4,40)  용량 초과! tail인 [2]를 제거
           리스트: [4] → [1] → [3]

get(2)     → 캐시 미스 (-1 반환). 이미 제거됨
```

## 실무에서의 LRU

- **Linux 페이지 캐시:** 커널이 디스크 블록을 메모리에 캐시할 때 LRU 근사 알고리즘 사용
- **Redis:** `maxmemory-policy allkeys-lru` 설정 시 LRU로 키를 제거
- **CPU TLB:** 가상→물리 주소 변환 캐시. LRU 근사로 교체
- **웹 브라우저 캐시:** 가장 오래 사용하지 않은 리소스부터 제거

## LRU의 한계와 변형

LRU는 **빈도(frequency)**를 고려하지 않는다. 한 번만 접근된 대량의 데이터가 자주 접근되는 소수의 데이터를 밀어낼 수 있다 (cache pollution).

- **LFU (Least Frequently Used):** 접근 횟수가 가장 적은 항목을 제거한다.
- **ARC (Adaptive Replacement Cache):** LRU와 LFU를 적응적으로 조합한다. ZFS 파일 시스템에서 사용된다.
- **2Q / SLRU:** 최근 접근 리스트와 빈번 접근 리스트를 분리하여 관리한다.
- **Clock (Second Chance):** LRU 근사 알고리즘이다. 원형 버퍼와 reference bit를 사용하며, OS 페이지 교체에 사용된다.

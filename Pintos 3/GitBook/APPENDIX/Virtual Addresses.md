64비트 가상 주소는 다음과 같이 구성됩니다:

```
63          48 47            39 38            30 29            21 20         12 11         0
+-------------+----------------+----------------+----------------+-------------+------------+
| Sign Extend |    Page-Map    | Page-Directory | Page-directory |  Page-Table |  Physical  |
|             | Level-4 Offset |    Pointer     |     Offset     |   Offset    |   Offset   |
+-------------+----------------+----------------+----------------+-------------+------------+
              |                |                |                |             |            |
              +------- 9 ------+------- 9 ------+------- 9 ------+----- 9 -----+---- 12 ----+
                                          Virtual Address
```

헤더 `include/threads/vaddr.h` 와 `include/threads/mmu.h` 는 가상 주소를 다루기 위한 다음의 함수와 매크로들을 정의합니다:

```c
#define PGSHIFT { /* Omit details */ }
#define PGBITS  { /* Omit details */ }
```

각각 가상 주소의 오프셋 부분의 **비트 시작 인덱스(0)** 와 **비트 수(12)** 를 의미합니다.

```c
#define PGMASK  { /* Omit details */ }
```

페이지 오프셋에 해당하는 비트들이 **1**이고, 나머지는 **0**인 비트 마스크(**0xfff**).

```c
#define PGSIZE  { /* Omit details */ }
```

페이지 크기(바이트 단위, **4,096**).

```c
#define pg_ofs(va) { /* Omit details */ }
```

가상 주소 `va`에서 **페이지 오프셋**을 추출하여 반환합니다.

```c
#define pg_no(va)  { /* Omit details */ }
```

가상 주소 `va`에서 **페이지 번호**를 추출하여 반환합니다.

```c
#define pg_round_down(va) { /* Omit details */ }
```

`va`가 속한 **가상 페이지의 시작 주소**(즉, 페이지 오프셋을 0으로 만든 주소)를 반환합니다.

```c
#define pg_round_up(va) { /* Omit details */ }
```

`va`를 **가장 가까운 페이지 경계로 올림**하여 반환합니다.

---

Pintos의 가상 메모리는 **사용자 가상 메모리**와 **커널 가상 메모리**의 두 영역으로 나뉩니다(“Virtual Memory Layout” 참조).

두 영역의 경계는 **`KERN_BASE`** 입니다:

```c
#define KERN_BASE { /* Omit details */ }
```

커널 가상 메모리의 **기준(base) 주소**입니다. 기본값은 **0x8004000000** 입니다. 사용자 가상 메모리는 가상 주소 **0**부터 `KERN_BASE` **바로 아래**까지의 범위입니다. 커널 가상 메모리는 **나머지 가상 주소 공간**을 점유합니다.

```c
#define is_user_vaddr(vaddr)  { /* Omit details */ }
#define is_kernel_vaddr(vaddr){ /* Omit details */ }
```

각각 `va`가 **사용자/커널 가상 주소**이면 **true**, 그렇지 않으면 **false**를 반환합니다.

x86-64는 **물리 주소**가 주어졌을 때 그 메모리를 **직접 접근하는 방법**을 제공하지 않습니다. 운영체제 커널에서는 이 능력이 종종 필요하므로, Pintos는 **커널 가상 메모리를 물리 메모리에 1:1로 매핑**하는 방식으로 이를 우회합니다. 즉, **가상 주소 > `KERN_BASE`** 는 **물리 주소 0**을 접근하고, **가상 주소 `KERN_BASE + 0x1234`** 는 **물리 주소 `0x1234`** 를 접근하며, 이는 머신의 물리 메모리 크기까지 이어집니다. 따라서 **물리 주소에 `KERN_BASE`를 더하면** 해당 주소를 접근하는 **커널 가상 주소**를 얻고, 반대로 **커널 가상 주소에서 `KERN_BASE`를 빼면** 대응하는 **물리 주소**를 얻습니다.

헤더 `include/threads/vaddr.h` 는 이러한 변환을 수행하는 **두 개의 함수**를 제공합니다:

```c
#define ptov(paddr) { /* Omit details */ }
```

물리 주소 `pa`(0 이상, 물리 메모리 총 바이트 수 이하)에 대응하는 **커널 가상 주소**를 반환합니다.

```c
#define vtop(vaddr) { /* Omit details */ }
```

**커널 가상 주소** `va`에 대응하는 **물리 주소**를 반환합니다.

헤더 `include/threads/mmu.h` 는 **페이지 테이블 연산**을 제공합니다:

```c
#define is_user_pte(pte) { /* Omit details */ }
#define is_kern_pte(pte) { /* Omit details */ }
```

해당 **페이지 테이블 엔트리(PTE)** 가 각각 **사용자** 소유인지, **커널** 소유인지 질의합니다.

```c
#define is_writable(pte) { /* Omit details */ }
```

해당 **페이지 테이블 엔트리(PTE)** 가 가리키는 **가상 주소가 쓰기 가능**한지 질의합니다.

```c
typedef bool pte_for_each_func (uint64_t *pte, void *va, void *aux);
bool pml4_for_each (uint64_t *pml4, pte_for_each_func *func, void *aux);
```

PML4 하위의 **유효한 각 엔트리**에 대해, 보조 값 `AUX`와 함께 **`FUNC`** 를 적용합니다. `VA`는 그 엔트리의 **가상 주소**를 나타냅니다. 만약 `pte_for_each_func`가 **false**를 반환하면, **순회를 중단**하고 **false**를 반환합니다.

아래는 `pml4_for_each`에 전달할 수 있는 **예시 함수**입니다:

```c
static bool
stat_page (uint64_t *pte, void *va,  void *aux) {
        if (is_user_vaddr (va))
                printf ("user page: %llx\n", va);
        if (is_writable (va))
                printf ("writable page: %llx\n", va);
        return true;
}
```
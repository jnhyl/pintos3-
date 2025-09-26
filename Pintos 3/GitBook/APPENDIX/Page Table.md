# Page Table
`threads/mmu.c`의 코드는 x86_64 하드웨어 **페이지 테이블**에 대한 **추상 인터페이스**입니다. Pintos의 페이지 테이블(여러분이 프로젝트에서 사용할 테이블)은 인텔 프로세서 문서에서 **PML4 (Page-Map-Level-4)** 라고 불리며, 이는 테이블이 **4단계** 구조이기 때문입니다. 페이지 테이블 인터페이스는 내부 구조에 접근하기 편하도록 **`uint64_t *`** 를 사용해 페이지 테이블을 표현합니다. 아래 섹션에서는 페이지 테이블 **인터페이스와 내부 동작**을 설명합니다.

---

## Creation, Destruction, and Activation

다음 함수들은 페이지 테이블을 **생성**, **파괴**, **활성화**합니다. **기본 Pintos 코드**가 필요한 곳에서 이미 이 함수들을 호출하므로, 일반적으로 여러분이 직접 호출할 필요는 없습니다.

```c
uint64_t * pml4_create (void);
```

새 페이지 테이블을 **생성하고 반환**합니다. 새 페이지 테이블에는 Pintos의 **일반적인 커널 가상 페이지 매핑**은 포함되지만, **사용자 가상 매핑은 없습니다**. 메모리를 확보할 수 없으면 **NULL 포인터**를 반환합니다.

```c
void pml4_destroy (uint64_t *pml4);
```

`pml4`가 보유한 **모든 리소스**(페이지 테이블 자체와, 그것이 매핑한 **프레임** 포함)를 **해제**합니다. 모든 레벨의 리소스를 해제하기 위해 **`pdpe_destroy`**, **`pgdir_destory`**, **`pt_destroy`** 를 **재귀적으로 호출**합니다.

```c
void pml4_activate (uint64_t *pml4);
```

`pml4`를 **활성화**합니다. **활성 페이지 테이블**은 CPU가 **메모리 참조를 변환**할 때 사용하는 테이블입니다.

---

## Inspection and Updates

다음 함수들은 페이지 테이블에 캡슐화된 **페이지→프레임 매핑을 조회/갱신**합니다. 이 함수들은 **활성/비활성** 페이지 테이블(즉, 실행 중/중단된 프로세스의 테이블) **모두에서 작동**하며, 필요할 경우 **TLB를 플러시**합니다.

```c
bool pml4_set_page (uint64_t *pml4, void *upage, void *kpage, bool rw);
```

`pml4`에 사용자 페이지 `upage` → **커널 가상 주소** `kpage`가 가리키는 **프레임**으로의 **매핑을 추가**합니다. `rw`가 **true**면 **읽기/쓰기**, 아니면 **읽기 전용**으로 매핑합니다. 사용자 페이지 `upage`는 `pml4`에 **이미 매핑되어 있어서는 안 됩니다**. 커널 페이지 `kpage`는 **`palloc_get_page(PAL_USER)`로 사용자 풀에서 얻은 커널 가상 주소**여야 합니다. 성공 시 **true**, 실패 시 **false**를 반환합니다. **실패**는 페이지 테이블에 필요한 **추가 메모리를 확보하지 못했을 때** 발생합니다.

```c
void * pml4_get_page (uint64_t *pml4, const void *uaddr);
```

`pml4`에서 `uaddr`에 매핑된 **프레임을 조회**합니다. 매핑되어 있다면 그 **프레임의 커널 가상 주소**를 반환하고, 아니라면 **NULL**을 반환합니다.

```c
void pml4_clear_page (uint64_t *pml4, void *upage);
```

`pml4`에서 해당 페이지를 **“not present”** 로 표시합니다. 이후 이 페이지에 대한 접근은 **폴트**를 일으킵니다. 페이지 테이블의 **다른 비트들은 보존**되어, **accessed/dirty 비트**(다음 섹션 참조)를 확인할 수 있습니다. 페이지가 **매핑되어 있지 않다면** 아무 효과가 없습니다.

---

## Accessed and Dirty Bits

x86_64 하드웨어는 각 페이지의 **PTE(페이지 테이블 엔트리)** 에 있는 **두 비트**를 통해 페이지 교체 알고리즘 구현을 **도와줍니다**. 어떤 페이지에 **읽기 또는 쓰기**가 발생하면 CPU는 그 페이지의 PTE의 **accessed 비트**를 **1**로 설정하고, **쓰기**가 발생하면 **dirty 비트**를 **1**로 설정합니다. CPU는 이 비트들을 **0으로 되돌리지 않지만**, **OS는 재설정할 수 있습니다**.

이 비트들을 올바르게 해석하려면 **alias(별칭)** 개념을 이해해야 합니다. 즉, **같은 프레임을 참조하는 둘 이상의 페이지**가 있는 경우입니다. **별칭 프레임**이 접근되면, **accessed/dirty 비트는 접근에 사용된 페이지의 PTE에만** 업데이트됩니다. **다른 별칭들의 PTE 비트는 갱신되지 않습니다**.

```c
bool pml4_is_dirty (uint64_t *pml4, const void *vpage);
bool pml4_is_accessed (uint64_t *pml4, const void *vpage);
```

`pml4`가 `vpage`에 대한 **PTE를 가지고 있고**, 그 엔트리가 **dirty(또는 accessed)** 로 표시되어 있으면 **true**, 아니면 **false**를 반환합니다.

```c
void pml4_set_dirty (uint64_t *pml4, const void *vpage, bool dirty);
void pml4_set_accessed (uint64_t *pml4, const void *vpage, bool accessed);
```

`pml4`가 해당 페이지에 대한 **PTE를 가지고 있다면**, 그 **dirty(또는 accessed) 비트**를 **주어진 값**으로 설정합니다.
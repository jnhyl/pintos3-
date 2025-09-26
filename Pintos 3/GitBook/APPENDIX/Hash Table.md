# Hash Table

Pintos는 `lib/kernel/hash.c`에 해시 테이블 자료구조를 제공합니다. 이를 사용하려면 `#include <hash.h>`로 헤더 파일 `lib/kernel/hash.h`를 포함해야 합니다. Pintos에 제공된 코드 중 해시 테이블을 사용하는 코드는 없습니다. 따라서 원하시는 대로 그대로 사용해도 되고, 목적에 맞게 구현을 수정해도 되며, 무시해도 됩니다.

가상 메모리 프로젝트의 대부분 구현에서는 페이지에서 프레임으로의 변환(매핑)에 해시 테이블을 사용합니다. 해시 테이블을 다른 용도로 활용할 수도 있습니다.

## Data Types

해시 테이블은 `struct hash`로 표현됩니다.

```
struct hash;
```

해시 테이블 전체를 나타냅니다. `struct hash`의 실제 멤버는 “불투명(opaque)”합니다. 즉, 해시 테이블을 사용하는 코드는 `struct hash` 멤버에 직접 접근해서는 안 되며, 그럴 필요도 없습니다. 대신 해시 테이블 함수와 매크로를 사용하십시오.

해시 테이블은 `struct hash_elem` 타입의 요소(element) 위에서 동작합니다.

```
struct hash_elem;
```

해시 테이블에 포함시키려는 구조체 안에 `struct hash_elem` 멤버를 내장(embed)하세요. `struct hash_elem` 역시 `struct hash`와 마찬가지로 불투명합니다. 해시 테이블 요소를 다루는 모든 함수는 실제로는 여러분의 실제 요소 타입의 포인터가 아니라 `struct hash_elem`에 대한 포인터를 인자로 받고 반환합니다.

해시 테이블의 실제 요소로부터 `struct hash_elem`을 얻어야 하거나, 그 반대로 해야 할 때가 자주 있습니다. 실제 요소로부터는 `&` 연산자를 써서 그 요소 안의 `struct hash_elem` 포인터를 얻을 수 있습니다. 반대 방향(즉, `struct hash_elem`으로부터 실제 구조체의 포인터를 얻는 것)은 `hash_entry()` 매크로를 사용하십시오.

```
#define hash_entry (elem, type, member) { /* Omit details */ }
```

`elem`이 가리키는 `struct hash_elem`이 내장돼 있는 구조체의 포인터를 반환합니다. `type`에는 `elem`이 들어 있는 구조체의 이름을, `member`에는 그 구조체 안에서 `elem`이 가리키는 멤버의 이름을 제공해야 합니다.

예를 들어, `h`가 `struct thread`의 멤버(타입은 `struct hash_elem`)인 `h_elem`을 가리키는 `struct hash_elem *` 변수라고 가정합시다. 그러면 `hash_entry (h, struct thread, h_elem)`은 `h`가 가리키는 내부의 `struct thread`의 주소를 돌려줍니다.

각 해시 테이블 요소는 키(key)를 포함해야 합니다. 키란 요소를 식별하고 구별하는 데이터이며, 해시 테이블 내 요소들 사이에서 반드시 유일해야 합니다. (요소는 또한 키가 아닌 데이터도 포함할 수 있으며, 이는 유일할 필요가 없습니다.) 요소가 해시 테이블 안에 있는 동안에는 그 요소의 키 데이터를 변경해서는 안 됩니다. 필요하다면, 먼저 요소를 해시 테이블에서 제거하고 키를 수정한 뒤, 그 요소를 다시 삽입해야 합니다.

각 해시 테이블에 대해, 키를 대상으로 동작하는 두 개의 함수를 작성해야 합니다: 해시 함수와 비교 함수. 이 함수들은 다음 프로토타입과 일치해야 합니다:

```
typedef unsigned hash_hash_func (const struct hash_elem *e, void *aux);
```

요소의 데이터에 대한 해시 값을 `unsigned int` 범위 어디든 올 수 있는 값으로 반환합니다. 요소의 해시는 요소의 키에 대한 의사난수 함수여야 합니다. 요소의 키가 아닌 데이터나, 키 외의 비상수 데이터에 의존해서는 안 됩니다. Pintos는 해시 함수의 적절한 기반으로 사용할 수 있는 다음 함수를 제공합니다.

```
unsigned hash_bytes (const void *buf, size t *size)
```

`buf`에서 시작하는 `size` 바이트에 대한 해시 값을 반환합니다. 구현은 32비트 워드를 위한 범용 Fowler–Noll–Vo 해시입니다.
[http://en.wikipedia.org/wiki/Fowler_Noll_Vo_hash](http://en.wikipedia.org/wiki/Fowler_Noll_Vo_hash)

```
unsigned hash_string (const char *s)
```

널 종료 문자열 `s`의 해시를 반환합니다.

```
unsigned hash_int (int i)
```

정수 `i`의 해시를 반환합니다.

키가 적절한 타입의 단일 데이터 조각이라면, 해시 함수가 위 함수들 중 하나의 출력을 그대로 반환하는 것이 합리적입니다. 데이터가 여러 조각으로 이루어져 있다면, 이들 함수를 여러 번 호출한 결과를 ‘^’(배타적 OR) 연산자 등으로 결합할 수 있습니다. 마지막으로, 이 함수들을 완전히 무시하고 해시 함수를 처음부터 직접 작성해도 되지만, 여러분의 목표는 해시 함수를 설계하는 것이 아니라 운영체제 커널을 구축하는 것임을 기억하십시오. `aux`에 대한 설명은 [Hash Auxiliary Data] 절을 참조하세요.

```
bool hash_less_func (const struct hash_elem *a, const struct hash_elem *b, void *aux)
```

요소 `a`와 `b`에 저장된 키를 비교합니다. `a`가 `b`보다 작으면 true를, 크거나 같으면 false를 반환합니다. 두 요소가 비교하여 같다고 판단되면, 그 둘은 동일한 해시 값으로 해시되어야 합니다.

`aux`에 대한 설명은 [Hash Auxiliary Data] 절을, 해시와 비교 함수의 예시는 [Hash Table Example] 절을 참고하세요. 몇몇 함수는 인자로 세 번째 종류의 함수에 대한 포인터를 받습니다:

```
void hash_action_func (struct hash_elem *element, void *aux)
```

호출자가 선택한 어떤 동작을 `element`에 대해 수행합니다. `aux`에 대한 설명은 [Hash Auxiliary Data] 절을 참고하세요.

## Basic Functions

다음 함수들은 해시 테이블을 생성, 파괴, 검사합니다.

```
bool hash_init (struct hash *hash, hash_hash_func *hash_func,
                    hash_less_func *less_func, void *aux)
```

`hash`를 해시 함수 `hash_func`, 비교 함수 `less_func`, 보조 데이터 `aux`를 갖는 해시 테이블로 초기화합니다. 성공 시 true, 실패 시 false를 반환합니다. `hash_init()`은 `malloc()`을 호출하며, 메모리를 할당할 수 없으면 실패합니다. `aux`에 대한 설명은 [Hash Auxiliary Data]를 참조하세요. 대개 `aux`는 널 포인터입니다.

```
void hash_clear (struct hash *hash, hash_action_func *action)
```

이전에 `hash_init()`으로 초기화된 `hash`에서 모든 요소를 제거합니다. `action`이 널이 아니면, 해시 테이블의 각 요소마다 한 번씩 호출되어, 호출자가 요소가 사용하는 메모리나 기타 자원을 해제할 기회를 제공합니다. 예를 들어 해시 테이블 요소가 `malloc()`으로 동적으로 할당된 경우, `action`은 그 요소를 `free()`할 수 있습니다. 이는 안전합니다. 왜냐하면 `hash_clear()`는 특정 요소에 대해 `action`을 호출한 이후에는 그 요소의 메모리에 접근하지 않기 때문입니다. 그러나 `action`은 `hash_insert()`나 `hash_delete()`처럼 해시 테이블을 수정할 수 있는 어떤 함수도 호출해서는 안 됩니다.

```
void hash_destroy (struct hash *hash, hash_action_func *action);
```

`action`이 널이 아니면, `hash_clear()` 호출과 동일한 의미로 해시의 각 요소에 대해 `action`을 호출합니다. 그런 다음 `hash`가 보유한 메모리를 해제합니다. 이후, `hash`는 중간에 `hash_init()`을 다시 호출하지 않는 한 어떤 해시 테이블 함수에도 전달되어서는 안 됩니다.

```
size_t hash_size (struct hash *hash);
```

현재 `hash`에 저장된 요소의 개수를 반환합니다.

```
bool hash_empty (struct hash *hash);
```

`hash`가 현재 요소를 하나도 포함하지 않으면 true, 하나 이상 포함하면 false를 반환합니다.

## Search Functions

아래 각 함수는 제공된 요소와 비교하여 같은 요소를 해시 테이블에서 검색합니다. 검색 성공 여부에 따라, 해시 테이블에 새 요소를 삽입하는 등의 동작을 수행하거나, 단순히 검색 결과를 반환합니다.

```
struct hash_elem *hash_insert (struct hash *hash, struct hash elem *element);
```

`hash`에서 `element`와 동일한 요소를 검색합니다. 찾지 못하면 `element`를 `hash`에 삽입하고 널 포인터를 반환합니다. 이미 `hash`에 `element`와 같은 요소가 있으면, 해시를 수정하지 않고 그 요소를 반환합니다.

```
struct hash_elem *hash_replace (struct hash *hash, struct hash elem *element);
```

`element`를 `hash`에 삽입합니다. 이미 `hash`에 `element`와 같은 요소가 있다면 제거합니다. 제거된 요소를 반환하며, 만약 동일한 요소가 없다면 널 포인터를 반환합니다.

반환된 요소에 연관된 자원(예: 동적으로 할당된 메모리)을 적절히 해제하는 책임은 호출자에게 있습니다. 예를 들어 해시 테이블 요소가 `malloc()`으로 동적으로 할당되었다면, 더 이상 필요 없을 때 호출자가 그 요소를 `free()`해야 합니다.

다음 함수들에 전달되는 `element`는 오직 해시와 비교 목적에만 사용됩니다. 실제로 해시 테이블에 삽입되지는 않습니다. 따라서 그 요소에서는 키 데이터만 초기화하면 되며, 다른 데이터는 사용되지 않습니다. 흔히 요소 타입의 인스턴스를 지역 변수로 선언하고 키 데이터를 초기화한 뒤, 그 지역 변수 안의 `struct hash_elem`의 주소를 `hash_find()` 또는 `hash_delete()`에 전달하는 것이 합리적입니다. 예시는 [Hash Table Example]을 참조하세요. (큰 구조체를 지역 변수로 할당해서는 안 됩니다. 자세한 내용은 `struct thread`를 참조하세요.)

```
struct hash_elem *hash_find (struct hash *hash, struct hash elem *element);
```

`hash`에서 `element`와 동일한 요소를 검색합니다. 찾으면 그 요소를 반환하고, 없으면 널 포인터를 반환합니다.

```
struct hash_elem *hash_delete (struct hash *hash, struct hash elem *element);
```

`hash`에서 `element`와 동일한 요소를 검색합니다. 찾으면 `hash`에서 제거하고 그 요소를 반환합니다. 찾지 못하면 널 포인터를 반환하며 `hash`는 변경되지 않습니다.

반환된 요소에 연관된 자원(예: 동적으로 할당된 메모리)을 적절히 해제하는 책임은 호출자에게 있습니다. 예를 들어 해시 테이블 요소가 `malloc()`으로 동적으로 할당되었다면, 더 이상 필요 없을 때 호출자가 그 요소를 `free()`해야 합니다.

## Iteration Functions

다음 함수들은 해시 테이블의 요소들을 순회(iterate)할 수 있게 해줍니다. 두 가지 인터페이스가 제공됩니다. 첫 번째는 각 요소에 대해 수행할 `hash_action_func`를 작성하여 제공하는 방식입니다.

```
void hash_apply (struct hash *hash, hash action func *action);
```

`hash`의 각 요소에 대해 한 번씩 `action`을 호출합니다(순서는 임의). `action`은 `hash_insert()`나 `hash_delete()`처럼 해시 테이블을 수정할 수 있는 어떤 함수도 호출해서는 안 됩니다. `action`은 요소의 키 데이터를 수정해서는 안 되지만, 그 외의 데이터는 수정할 수 있습니다.

두 번째 인터페이스는 “이터레이터(iterator)” 자료형에 기반합니다. 관용적으로 이터레이터는 다음과 같이 사용합니다:

```
struct hash_iterator i;
hash_first (&i, h);
while (hash_next (&i)) {
    struct foo *f = hash_entry (hash_cur (&i), struct foo, elem);
    . . . do something with f . . .
}
```

```
struct hash_iterator;
```

해시 테이블 내의 한 위치를 나타냅니다. `hash_insert()`나 `hash_delete()`처럼 해시 테이블을 수정할 수 있는 어떤 함수를 호출하면, 해당 해시 테이블 내의 모든 이터레이터는 무효화됩니다.

`struct hash`와 `struct hash_elem`처럼, `struct hash_elem`(원문과 동일)을 포함해 불투명합니다.

```
void hash_first (struct hash iterator *iterator, struct hash *hash);
```

`iterator`를 `hash`의 첫 번째 요소 바로 이전 위치로 초기화합니다.

```
struct hash_elem *hash_next (struct hash iterator *iterator);
```

`iterator`를 `hash`의 다음 요소로 진행시키고, 그 요소를 반환합니다. 더 이상 요소가 없다면 널 포인터를 반환합니다. `hash_next()`가 `iterator`에 대해 널을 반환한 이후에 이를 다시 호출하는 것은 정의되지 않은 동작입니다.

```
struct hash_elem *hash_cur (struct hash iterator *iterator);
```

`iterator`에 대해 가장 최근 `hash_next()`가 반환한 값을 돌려줍니다. `hash_first()`가 `iterator`에 대해 호출된 후, `hash_next()`가 처음 호출되기 전까지 이 함수를 호출하는 것은 정의되지 않은 동작을 야기합니다.

## Hash Table Example

해시 테이블에 넣고 싶은 `struct page`라는 구조체가 있다고 가정합시다. 먼저 `struct page`에 `struct hash_elem` 멤버를 포함하도록 정의합니다:

```
struct page {
    struct hash_elem hash_elem; /* Hash table element. */
    void *addr; /* Virtual address. */
    /* . . . other members. . . */
};
```

키로 `addr`을 사용하여 해시 함수와 비교 함수를 작성합니다. 포인터는 그 바이트들에 기반하여 해시할 수 있고, 포인터 비교에는 `<` 연산자가 잘 동작합니다:

```
/* Returns a hash value for page p. */
unsigned
page_hash (const struct hash_elem *p_, void *aux UNUSED) {
  const struct page *p = hash_entry (p_, struct page, hash_elem);
  return hash_bytes (&p->addr, sizeof p->addr);
}
```

```
/* Returns true if page a precedes page b. */
bool
page_less (const struct hash_elem *a_,
           const struct hash_elem *b_, void *aux UNUSED) {
  const struct page *a = hash_entry (a_, struct page, hash_elem);
  const struct page *b = hash_entry (b_, struct page, hash_elem);

  return a->addr < b->addr;
}
```

(이 함수 프로토타입에서 `UNUSED`를 사용한 이유는 `aux`가 사용되지 않는다는 경고를 억제하기 위해서입니다. `UNUSED`에 대한 정보는 Function and Parameter Attributes를 참조하세요. `aux`에 대한 설명은 [Hash Auxiliary Data] 절을 참고하세요.)

그런 다음, 다음과 같이 해시 테이블을 생성할 수 있습니다:

```
struct hash pages;
hash_init (&pages, page_hash, page_less, NULL);
```

이제 생성한 해시 테이블을 조작할 수 있습니다. `p`가 `struct page`에 대한 포인터라면, 다음과 같이 해시 테이블에 삽입할 수 있습니다:

```
hash_insert (&pages, &p->hash_elem);
```

`pages`가 이미 같은 `addr`을 가진 페이지를 포함하고 있을 가능성이 있다면, `hash_insert()`의 반환값을 확인해야 합니다.

해시 테이블에서 요소를 검색하려면 `hash_find()`를 사용합니다. 이 함수는 비교 대상으로 사용할 요소를 받기 때문에 약간의 준비가 필요합니다. 아래는 전역(파일 스코프)에 `pages`가 정의되어 있다고 가정할 때, 가상 주소를 기반으로 페이지를 찾아 반환하는 함수입니다:

```
/* Returns the page containing the given virtual address, or a null pointer if no such page exists. */
struct page *
page_lookup (const void *address) {
  struct page p;
  struct hash_elem *e;

  p.addr = address;
  e = hash_find (&pages, &p.hash_elem);
  return e != NULL ? hash_entry (e, struct page, hash_elem) : NULL;
}
```

여기서 `struct page`는 꽤 작다는 가정하에 지역 변수로 할당됩니다. 큰 구조체는 지역 변수로 할당해서는 안 됩니다.

유사한 함수로, `hash_delete()`를 사용하여 주소로 페이지를 삭제할 수도 있습니다.

## Auxiliary Data

위의 단순한 예제처럼, `aux` 파라미터가 필요 없는 경우도 있습니다. 이런 경우 `hash_init()`에 `aux`로 널 포인터를 전달하고, 해시 함수와 비교 함수에서 전달되는 `aux` 값을 무시하면 됩니다. (컴파일러가 `aux` 파라미터를 사용하지 않는다고 경고할 수 있는데, 예제에서처럼 `UNUSED` 매크로로 경고를 끄거나 그냥 무시해도 됩니다.)

`aux`는 해시 테이블 안의 데이터에서 어떤 속성이 “상수이면서” 해시나 비교에 필요하지만, 데이터 항목 자체에는 저장되어 있지 않을 때 유용합니다. 예를 들어, 해시 테이블의 항목들이 고정 길이 문자열이지만, 각 항목 자체에는 그 고정 길이가 무엇인지 표시되어 있지 않은 경우, 그 길이를 `aux` 파라미터로 전달할 수 있습니다.

## Synchronization

해시 테이블은 내부적으로 동기화를 수행하지 않습니다. 해시 테이블 함수 호출의 동기화를 책임지는 것은 호출자입니다. 일반적으로, 해시 테이블을 수정하지 않는 함수들(예: `hash_find()`나 `hash_next()`)은 동시에 여러 개가 실행되어도 안전합니다. 그러나 특정 해시 테이블에 대해, 수정할 수 있는 함수(예: `hash_insert()`나 `hash_delete()`)가 실행되는 동안에는 이러한 함수들과 동시에 실행되어서는 안전하지 않으며, 수정 가능한 함수들끼리도 동시에 두 개 이상 실행되어서는 안전하지 않습니다.

또한 해시 테이블 요소의 데이터 접근을 동기화하는 책임도 호출자에게 있습니다. 이 데이터에 대한 동기화 방법은, 다른 어떤 자료구조와 마찬가지로, 그 데이터가 어떻게 설계되고 조직되어 있는지에 따라 달라집니다.

---

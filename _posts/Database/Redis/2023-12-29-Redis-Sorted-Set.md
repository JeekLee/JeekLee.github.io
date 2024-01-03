---
title: "Redis Sorted Set"
date: 2023-12-29 09:00:00 +0900
categories: [Database]
tags: [redis, sorted_set]
render_with_liquid: false
math: true
---


# I. Data structure
## 1. Zset
![redis-zset-structure](https://github.com/JeekLee/JeekLee.github.io/assets/72681875/7ab02688-1983-402f-a0d4-3feea33100ed)
_(그림 I-1-1) Redis Zset 예시_

`Redis`에서 sorted set은 `Zset`이라 표현한다.
`Zset`은 **ZipList**와 **SkipList**라는 두 가지 데이터 구조로 구성되어 있으며, `Redis`는 데이터에 따라 저장할 데이터 구조를 선택한다.
`Redis`가 데이터 구조를 결정하는 조건은 다음과 같다.

> 1. value의 멤버 수가 128개 이하인 경우 ZipList, 아닐 경우 SkipList 사용
> 2. value의 멤버 중 값(value)이 64byte 미만인 경우만 존재할 경우 ZipList, 아닐 경우 SkipList

위 조건은 `redis.conf`{: .filepath}의 ADVANCED CONFIG 부분에서 수정할 수 있다.
```conf
zset-max-ziplist-entries 128
zset-max-ziplist-value 64
```
{: file="redis.conf" }

### 1) ZipList
Redis는 인메모리 방식의 데이터 저장소이다. 메모리(RAM)는 디스크와 비교했을 때 매우 비싼 자원이며, 메모리 사용 시 그에 따른 최적화가 필수적이다.
데이터 타입 혹은 자료 구조에 따라 실질적인 데이터 외에 메모리 오버헤드가 발생하게 되는데, **경우에 따라 실 데이터의 몇 배에 달하는 자원이 필요**하다.
사실 다른 DBMS에서 역시 메모리 오버헤드 문제는 동일하지만, 메모리(RAM)에서 발생하진 않기 때문에 상대적으로 중요도가 떨어지는 것에 가깝다.

ZipList는 이러한 메모리 오버헤드를 최소화하기 위해 도입되었으며, 구조는 다음과 같다.

![redis-ziplist](https://github.com/JeekLee/JeekLee.github.io/assets/72681875/b3f5ed06-9348-467d-974d-6ed665f945dd)
_(그림 I-1-2) Zip List 구조_

- Header
  - `zlbytes`: ZipList의 총 Byte 수, `unsigned int 4 bytes`, 0 ~ 4,294,967,295(4GB)
  - `zllen`: ZipList의 총 Entry 수, `unsigned int 2 bytes`, 0 ~ 65,535 (64KB)
  - `zltail`: 마지막 Entry의 시작 지점(offset), `unsigned int 4 bytes`, 0 ~ 4,294,967,295(4GB)
- ZIP_END
  - `zlend`: ZipList의 끝을 나타내는 바이트(`11111111`), `1 byte`

따라서 ZipList의 오버헤드는 **총 11byte(Header + ZIP_END)**이다.

ZipList 내부의 Entry는 다음과 같은 구조로 구성된다.

![redis-ziplist-entry](https://github.com/JeekLee/JeekLee.github.io/assets/72681875/c3b74d1c-dee4-4ee1-bb76-26f047e35202)
_(그림 I-1-3) Zip List Entry 구조_

- `prevlen`: 이전 Entry의 길이, 역방향 탐색에 사용(1 byte or 5 byte)
- `itselflen`: 현재 Entry Value의 길이, 순방향 탐색 및 Value 조회에 사용(Value가 문자인 경우 1,2,5 byte, 숫자인 경우 1 byte)
- `value`: 문자와 정수로 구분해 저장. 문자는 문자 그대로 저장하며 정수는 4, 8, 16, 24, 32, 64bit로 구분해 저장.

#### 1. `prevlen` 구성
이전 Entry의 길이가 `ZIP_BIGLEN`(254)보다 작으면 1 byte를 할당해 다음과 같이 길이를 표현한다.

![redis-ziplist-entry-prevlen-1](https://github.com/JeekLee/JeekLee.github.io/assets/72681875/7127f295-507d-4194-8d9c-ab23d4a0df9b)
_(그림 I-1-4) Zip List Entry prevlen(1byte)_

이전 Entry의 길이가 `ZIP_BIGLEN` 이상일 경우 다음과 같이 첫 바이트에 `ZIP_BIGLEN`을 넣고 나머지 4 byte에 길이를 표현한다.
![redis-ziplist-entry-prevlen-2](https://github.com/JeekLee/JeekLee.github.io/assets/72681875/f942d4ce-9d24-4218-8834-5ce6b80f0bd2)
_(그림 I-1-5) Zip List Entry prevlen(5byte)_

#### 2. `itselflen` 구성
> 맨 앞 2개 비트가 00, 01, 10이면 문자, 11은 숫자를 표현한다.
{: .prompt-tip }

문자열의 길이가 63byte 이하일 땐 1byte를 사용하며 아래 예시처럼 길이를 저장하는 데 6bit를 사용한다.

![redis-ziplist-entry-itselflen-1](https://github.com/JeekLee/JeekLee.github.io/assets/72681875/fc510649-f0a6-4c80-ac77-e8e8449c3756)
_(그림 I-1-6) Zip List Entry itselflen string(1byte)_

문자열의 길이가 64 ~ 16,383 byte일 경우, 길이를 저장하는데 14bit를 사용한다.
![redis-ziplist-entry-itselflen-2](https://github.com/JeekLee/JeekLee.github.io/assets/72681875/36ad5ab5-cb1e-4ae3-87d5-ef2e3973d506)
_(그림 I-1-7) Zip List Entry itselflen string(2byte)_

문자열의 길이가 16,384 byte 이상일 경우, 길이를 지정하는데 4byte를 사용한다.
![redis-ziplist-entry-itselflen-3](https://github.com/JeekLee/JeekLee.github.io/assets/72681875/1fdf1e5c-d156-4bad-98cc-a63dd631de06)
_(그림 I-1-8) Zip List Entry itselflen string(5byte)_

숫자의 경우 1byte로 고정이지만, 2개의 sign bit가 추가된다. 숫자는 값의 길이를 표현할 필요가 없기 때문에, 수의 범위를 표현하는 데 `itselflen`을 사용한다.
다시말해 1byte 전체가 수의 범위를 표현하는 sign byte로 사용되며, 앞의 4bit를 사용한다.
- ZIP_INT_16B(11000000), -32,768 ~ 32,767, `value int 2 byte`, 약 3만
- ZIP_INT_32B(11010000), -2,147,483,648 ~ 2,147,483,647, `value int 4 byte`, 약 21억 
- ZIP_INT_64B(11100000), 약 900경, `value int 8 byte`
- ZIP_INT_24B(11110000), 약 800만, `value int 3 byte`
- ZIP_INT_8B(11111110), 약 150만, `value int 1 byte`

![redis-ziplist-operation](https://github.com/JeekLee/JeekLee.github.io/assets/72681875/e608b0cb-ff84-486b-8c8a-d8da6fb8a35c)
_(그림 I-1-9) Zip List Operation Example_

### 2) SkipList
**SkipList**는 정렬된 상태를 유지하면서 데이터를 삽입, 삭제, 조회할 수 있는 데이터 구조이다.
SkipList는 LinkedList의 단점을 보완하기 위해 구현되었다. 정렬된 LinkedList가 존재한다고 가정할 때,
특정 값을 지니는 노드를 찾으려면 최악의 경우 모든 노드와 일치 여부를 확인해야 한다.
아래 예시에서 파란색으로 표시된 80이라는 값을 조회하기 위해선, 순서대로 확인할 경우 총 9번의 연산을 필요로 한다.

![linked-list-example](https://github.com/JeekLee/JeekLee.github.io/assets/72681875/72a5ba96-fdb2-4dac-bd00-c766421881f1)
_(그림 I-1-10) Linked List_

절대적인 비교 횟수를 줄일 수 있다면, 탐색 시간을 단축할 수 있다. 비교 횟수를 줄이기 위해 등장한 방법이 Level의 적용이다.
아래 예시에서 각 List의 노드는 홀수 번째일 경우 한개, 짝수 번째일 경우 두 개의 포인터를 지닌다.
이 때, **각 노드의 레벨에 따라 2^n번째 앞에 존재하는 노드를 가르키는 포인터가 생성**된다.
이처럼 구현할 경우 노드를 하나씩 생략하면서 비교할 수 있기 때문에, 비교 횟수가 1/2 수준으로 떨어지게 된다.
실제 파란색으로 표시된 80이라는 값을 조회하기 위해서 연산 횟수는 6번으로 감소한다.

![linked=list-example-with-level](https://github.com/JeekLee/JeekLee.github.io/assets/72681875/b1e70033-075a-4052-933d-2b722f321f27)
_(그림 I-1-11) Linked List with Level_

이번엔 Level을 4개 적용해보자. 값이 70인 노드에 포인터 4개(Level 4)를 적용하면, 80이라는 값을 찾는 과정은 더욱 단순해진다.
하지만 단순 조회가 아닌 삽입 및 삭제 케이스를 고려해보자. 
노드가 삽입 혹은 삭제되면, 해당 노드 이후에 존재하는 모든 노드의 순서가 변경된다.
순서가 변경됨에 따라 각 노드별 레벨을 수정해야하며, 레벨을 많이 사용할 수록 **삽입/삭제 시 레벨 수정에 사용되는 비용이 증가**한다. 

![linked-list-example-with-4-levels](https://github.com/JeekLee/JeekLee.github.io/assets/72681875/9e57822d-03ed-4844-9ee6-9d6aa590fedc)
_(그림 I-1-12) Linked List with 4 Levels_

각 노드의 순서(위치)에 레벨이 **의존적**인 현재 형태에선 삽입/삭제 시 발생하는 비용을 핸들링할 방법이 없다.
다시 말해, 각 노드의 레벨을 독립적으로 설정할 수 있는 방법이 필요하다. 이에 대해 Redis는 확률에 따른 노드별 레벨 설정 방식을 사용한다.
노드가 생성될 때, 레벨은 1로 고정된다. 25% 확률로 레벨은 증가될 수 있으며, 레벨이 증가되지 않은 경우 노드의 레벨은 해당 레벨로 픽스된다.
문장으로 표현하면 복잡하지만 간단한 예시를 통해 쉽게 이해할 수 있다.

정사면체 주사위가 존재한다고 가정하자. 정사면체 주사위를 던져 나온 값에 따라 레벨의 수치가 정해진다.
1. 첫 번째 주사위를 던질 때 레벨은 1이다. $$1$$
2. 두 번째 주사위를 던져 같은 레벨이 나오면 레벨 2가되고, 아닐 경우 레벨 1로 마무리한다. $$1/4$$
3. 세 번째 주사위를 던져 같은 레벨이 나오면 레벨 3이되고, 아닐 경우 레벨 2로 마무리한다. $$1/4^2$$
4. 위 과정을 레벨이 고정될 때까지 반복한다. $$1/4^{n-1}$$

확률에 따른 레벨 설정 방식을 통해 순서에 대한 의존성을 제거했지만, 정작 SkipList의 목적 중 하나인 빠른 순서 확인은 불가능해졌다.
때문에 Redis는 레벨에 포인터와 순서를 지니는 필드를 만들어 저장하고, 해당 필드명을 SPAN으로 지정한다. 이를 **Indexed SkipList**라 한다.
아래 예시처럼, 탐색 중 지나친 레벨의 SPAN 값을 모두 더하면 해당 노드가 몇 번째에 위치해있는 지 알 수 있다.

![indexed-skip-list](https://github.com/JeekLee/JeekLee.github.io/assets/72681875/c3c4eab2-c4af-4f59-980f-948290ebeeae)
_(그림 I-1-13) Indexed Skip List_

앞서 예시로 든 주사위의 형태(정사면체, 정육면체 등)에 따라 Indexed SkipList의 메모리 오버헤드와 탐색 시간이 달라진다. 
Redis는 $$1/4$$ 확률을 사용하며, `server.h`{: .filepath} 파일에 다음과 같이 정의되어 있다.
```c
#define ZSKIPLIST_P   0.25       /* Skiplist P = 1/4 */
```
{: file="server.h" }

### 3) Redis SkipList
- `LEX`(Lexicographic sorting, 사전순 정렬) 명령을 사용하기 위해 동일한 Score가 반복되는 것을 허용한다. 당연히 Value는 달라야 한다.
- 역탐색을 위해 이전 노드를 가르키는 포인터(`*backward`)를 사용한다.
- **최대 레벨을 32레벨로 제한**한다.
- SkipList를 정의하는 구조체(`zskiplist`)에 최대 레벨, 노드 수, 헤더 및 테일 노드의 포인터를 지니고 있다.

![zskiplist-structure](https://github.com/JeekLee/JeekLee.github.io/assets/72681875/0a1ec4c9-174f-404c-8391-1b8ed1c500e1)
_(그림 I-1-14) zsiplist structure_

노드 삽입 과정을 도식화하면 다음과 같다.

![zskiplist-node1](https://github.com/JeekLee/JeekLee.github.io/assets/72681875/2fcf535d-7ba4-4def-827f-b0e7fd70567e)
_(그림 I-1-15) zsiplist node addition 1_

두 번째 노드 삽입 과정을 도식화하면 다음과 같다.

![zskiplist-node1](https://github.com/JeekLee/JeekLee.github.io/assets/72681875/cc25d026-135e-4188-a98f-16d2f162c40a)
_(그림 I-1-16) zsiplist node addition 2_


# X. References
1. [Redis Docs](https://redis.io/docs/data-types/)
2. [Redis Zip List](https://songhayoung.github.io/2021/06/04/Redis/zset-vs-list/#%EA%B0%9C%EC%9A%94)

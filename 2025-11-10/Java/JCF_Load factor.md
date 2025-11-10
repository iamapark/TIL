# Java Collection Framework - Load factor

## 이 주제를 공부하는 이유
- 매일메일 공부 주제: https://www.maeil-mail.kr/question/235

## 배운 내용

JCF 자료구조의 초기 용량 지정은 "동적으로 크기가 변하는 자료구조가 리사이징 하는 횟수를 줄여 성능을 향상시키는 최적화 기법" 입니다.

### 💡 왜 필요한가?
해결하는 문제:
ArrayList나 HashMap 같은 동적 자료구조는 데이터가 추가될 때마다 내부 배열이 가득차면 자동으로 크기를 늘립니다. 이 과정이 매우 비용이 큰 작업이기 때문에 이 문제를 제대로 다루는 것이 중요합니다.

리사이징이 일어날 때 무슨 일이 발생하나요?
```
1. 새로운 더 큰 배열 생성 (보통 2배 크기)
2. 기존 배열의 모든 요소를 새 배열로 복사
3. 기존 배열은 가비지 컬렉션 대상이 됨
```

비용 분석:
  - 시간 복잡도: O(n) - 모든 요소 복사
  - 메모리: 순간적으로 2배 사용 (구 배열 + 신 배열)
  - GC 압력: 버려지는 배열로 인한 가비지 증가

실제 문제 시나리오:
```java
// ❌ 나쁜 예: 초기 용량 없음
List<User> users = new ArrayList<>(); // 기본 용량: 10

// 1만 명의 사용자 추가
for (int i = 0; i < 10000; i++) {
    users.add(new User(i)); // 리사이징이 여러 번 발생!
}

// 리사이징 횟수: 약 14번!
// 10 → 15 → 22 → 33 → 49 → 73 → 109 → 163 → 244 → 366 → 549 → 823 → 1234 → 1851 → ...
```

### 🔑 핵심 원리 5가지
1. 리사이징 비용의 주거
ArrayList는 capacity가 가득 차면 `1.5배`씩 증가합니다:
    ```java
    // ArrayList의 실제 grow 메커니즘 (JDK 8+)
    int newCapacity = oldCapacity + (oldCapacity >> 1); // 1.5배
    ```

    **문제점:**
      - 10 -> 15 -> 22 -> 33 -> 49 ... 계속 복사 작업 발생
      - 각 리사이징마다 O(n) 시간 소요
      - 총 비용은 O(n log n)에 근접

2. Amortized Time의 차이
  - 초기 용량 없이 추가: 분할상환 O(1) 이지만 최악은 O(n)
  - 초기 용량 지정 후 추가: 진짜 O(1)

    **구체적인 차이:**
      - 초기 용량 없음: 10,000개 추가 시 14번 리사이징
      - 초기 용량 10,000: 10,000개 추가 시 0번 리사이징

3. 메모리 효율성
  - **과도한 리사이징의 메모리 낭비:**
    상황: 1000개 요소 저장
      - 초기 용량 없음: 최종 capacity = 1234(234개 낭비)
      - 초기 용량 1000: 최종 capacity = 1000(0개 낭비)

  - **순간 메모리 사용 피크:**
    리사이징 시점: 기존 배열(n) + 새 배열(1.5n) = 2.5n
    => 순간적으로 2.5배 메모리 사용!


4. HashMap의 Load Factor와 Threadhold
    HashMap은 더 복잡합니다:
    ```java
    // HashMap 기본값
    int capacity = 16;           // 초기 용량
    float loadFactor = 0.75f;    // 로드 팩터
    int threshold = capacity * loadFactor; // 12

    // threshold를 넘으면 리사이징 (capacity * 2)
    ```

    예시:
    ```java
    Map<String, Integer> map = new HashMap<>(); // capacity=16, threshold=12

    // 13번째 요소 추가 시 리사이징 발생!
    for (int i = 0; i < 13; i++) {
        map.put("key" + i, i);
    }
    ```

    **리사이징 과정의 추가 비용:**
      1. 새 배열 (capacity * 2) 생성
      2. 모든 엔트리 rehashing (해시 재계산)
      3. 새 배열에 재배치
      -> ArrayList 보다 훨씬 비쌈!

5. **실무 최적화 지표**
**벤치마크 결과 (10,000개 요소 기존):**
  ArrayList 추가 시간:
    - 초기 용량 없음: ~3.2ms (14번 리사이징)
    - 초기 용량 10,000: ~0.8ms (0번 리사이징)
  
  HashMap 추가 시간:
    - 초기 용량 없음: ~8.5ms (10번 리사이징 + rehashing)
    - 초기 용량 계산: ~2.1ms (0번 리사이징)

### ADT: Map

#### Map
  - Key-value pair들을 저장하는 ADT
  - 같은 key를 가지는 pair는 최대 한 개만 존재
  - associative array, dictionay 라고 불리기도 함

#### Map은 언제 쓸까?
  - 전화번호, 이름 쌍을 저장할 때
  - 어떤 개념을 '카운트' 할 때 (ex: 영화 별 인기 투표)

#### Map 구현체 
  - hash table
  - tree-based

#### Hash table (hash map)
  - 배열과 hash function을 사용하여 map을 구현한 자료 구조
  - (일반적으로) 상수 시간으로 데이터에 접근 가능

#### hash function
  - 임의의 크기를 가지는 type의 데이터를 고정된 크기를 가지는 type의 데이터로 변환하는 함수
  - (hash table에서) 임의의 데이터를 정수로 변환하는 함수

#### hash table은 어떻게 동작하는가?
  - put("010-2222-2222", "홍진호")
    - key 값(010-2222-2222)을 hash function의 input으로 넣는다. 
    - output으로 나오는 값(ex: 202)을 capacity(ex: 8)로 나눈다. (Modular 연산)
    - Modular 연산의 결과값에 해당하는 index 위치에 value("홍진호") 값을 저장한다. 이 때, Key와 Value를 같이 저장한다.
  - get("010-1010-1001")
    - key 값으로 hash function을 실행 그리고 capacity로 Modular 연산을 한다. 이 때 연산의 결과값의 index 위치에서 이미 저장된 값의 Key와 비교해서 'Equal' 하면 해당 Value를 사용하고 만약 다른 경우에는 계속 찾는다.

#### hash collision
  - key는 다른데 hash가 같을 때
    - "010-2222-2222", "010-1010-1001"의 hash 값이 같을 수도 있음
    - input의 범위 대비 hash function의 결과값 범위는 좁기 때문에 다른 값을 hash 하더라도 같은 값이 나올 수도 있음
  - key도 hash도 다른데 `hash % map_capa` 결과가 같을 때 (Modular 연산하고 나니 값이 같음)

#### hash collision 해결 방법
  - open addressing (linear probing)
    - hash collision이 발생하면, bucket의 비어있는 공간에 데이터를 저장
  - separate chaining
    - hash collision이 발생하면, bucket의 item 내부에 linked list를 만들어서 관리

## 학습 후 설명
Java Collection Framework(JCF)에서 초기 용량 지정은 매우 중요합니다. 그 이유는 JCF는 동적으로 내부 자료 구조의 크기를 조절하는데 이 때, 초기 용량에 따라서 성능이 저하될 수 있기 때문입니다. 따라서 예상되는 자료 구조의 크기에 따라서 초기 용량을 적절하게 지정하는 방식으로 코드를 작성해야 합니다.

### ArrayList
  - ArrayList는 내부적으로 배열을 갖고 있으며 ArrayList의 초기 용량 지정에 따라서 해당 배열 크기가 결정됩니다. 초기 용량이 명시적으로 지정되어 있지 않으면 기본값(Default capacity:10)을 사용합니다.
  - 가정: 초기 용량을 지정하지 않은 상황 (Capacity: 10)
    - 만약, 코드가 실행되면서 10개 미만으로 ArrayList에 item이 추가되면 상관 없지만 10개 이상의 item이 추가되면, 해당 시점에 capacity가 재조정 됩니다.(resizing)
    - ArrayList는 capacity를 1.5배씩 증가시키기 때문에 10개의 capacity를 15개로 늘립니다. 이 과정에서 새로운 배열을 만들고 구 배열 데이터를 신규 배열로 옮깁니다. 이 때, 메모리 점유(신 배열 + 구 배열)이 발생하게 됩니다.
  - 따라서 ArrayList의 초기 용량 지정은 매우 중요하며 expected size에 따라서 이 초기값을 계산하여 지정하는 것이 필요합니다.

### HashMap
  - HashMap 역시 내부적으로 배열을 갖고 있습니다. 이것은 Bucket 이라고 부르며 크기를 자유롭게 조절할 수 있어야 합니다.
  - 다만 HashMap은 ArrayList 보다 조금 더 복잡합니다. HashMap의 특성상 내부적으로 데이터를 저장/조회/삭제 할 때, hash function을 사용해야 하기 때문입니다.
    - HashMap은 item을 저장할 때, item의 key를 hash function의 input으로 하여 실행하고 계산된 결과값을 다시 Modular 연산(capacity length에 맞게)하여 최종 연산 값을 bucket의 index로 지정합니다. 그래서 item을 `bucket[index]`에 저장하는 것입니다.
  - 이와 같은 배경 하에, HashMap은 threadhold와 load factor 라는 2가지 수치를 가집니다.
    - threadhold = capacity x load factor(ex: 0.75)
    - 그리고 HashMap에 초기 용량을 지정하지 않으면 기본값은 16입니다.
    - 즉, 초기 용량을 지정하지 않으면 threadhold = 16 x 0.75 = 12
    - HashMap에 item을 추가할 때, threadhold를 넘어서는 경우 resizing이 발생합니다.
  - Resizing 발생 시 ArrayList와 다른 점
    - 배열의 크기를 조정하는 것은 동일
    - 그러나, HashMap의 특성상 item의 Key를 hash 계산하여 modular 연산까지 한 값이 bucket의 index이기 때문에 bucket의 크기가 늘어나게 되면 자연스럽게 modular 연산을 다시 해야 합니다. (= rehashing). 다만 이 때, hash 값 자체는 기존에 계산된 값을 재사용하며 `index = hash % capacity`를 재계산 하는 것입니다. 그리고 재계산 한 위치에 Entry를 재배치 하는 작업까지 수행해야 합니다.
    - 즉, `1. 배열 크기 조정` + `2. rehasing` 작업을 해야 하는 것입니다.
  - HashMap에서 resizing 할 때, Entry를 재배치 하는 이유
    - HashMap의 성능은 각 Entry가 얼마나 '균등'하게 배치되어 있는가에 달려있습니다.
    - capacity 16에서 capacity 32로 resizing 될 때 Entry 재배치를 하지 않는다면 bucket의 앞쪽 0~15 index 부분에만 데이터가 저장되어 있을 것이기 때문에 균등하게 배치되어 있지 않게 됩니다.

### 성능
  - 최적의 성능을 발휘하기 위해 ArrayList, HashMap 모두 적절한 초기 용량을 지정하는 것이 매우 중요합니다.
  - 초기 용량 계산 방법
    - ArrayList: expected item 개수 만큼 초기 용량을 지정
    - HashMap: (expected item 개수 / load factor(0.75)) + 1
  - 초기 용량 개수가 큰 경우
    - 만약 초기 용량이 매우 큰 경우에는 다른 방식으로 접근해야 합니다.
    - 예를 들어, 초기 용량이 1,000,000,000 이라면 이것은 다른 성능 저하 요소를 발생시킬 수 있습니다.
      - 초기 메모리 점유가 매우 큼
      - Iteration 하는 데 많은 시간이 소모됨
      - 내부 배열의 크기가 큰 경우 CPU cache(L1/L2)가 아니라 RAM에서 직접 읽어야 할 수 있습니다. 이 경우 성능 저하가 발생합니다.

## 참고 자료
  - [Claude 학습 자료](https://claude.ai/share/161f78f8-9c2f-4685-a166-ced29616a957)
  - [쉬운 코드 - BJ.23 맵(map)과 해시 테이블(hash table) 혹은 해시 맵(hash map)의 핵심만 모아보기! 20분간 아주아주아주 알차게 설명합니다!!](https://www.youtube.com/watch?v=ZBu_slSH5Sk)
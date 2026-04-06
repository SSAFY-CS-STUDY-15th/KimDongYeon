# 인덱스 스캔 방식의 이해 — Range, Full, Loose

분류: CS, DataBase

## 1. 발표 개요

오늘은 MySQL에서 인덱스를 "어떻게 읽는가"에 대해 이야기하려 합니다.

우리가 인덱스를 설계하고 쿼리를 작성하면, MySQL 옵티마이저는 그 인덱스를 다양한 방식으로 읽습니다. 같은 인덱스라 하더라도 쿼리 조건에 따라 **레인지 스캔**이 될 수도 있고, **풀 스캔**이 될 수도 있고, 때로는 **루스 인덱스 스캔**이라는 독특한 방식이 선택되기도 합니다.

EXPLAIN을 찍었을 때 `type` 컬럼에 `range`가 나오면 "아 범위 검색이구나" 정도로 넘기는 경우가 많은데, 실제로 그 안에서 B+Tree의 어디를 어떻게 읽고 있는지를 이해하면 쿼리 최적화에 대한 감각이 달라집니다.

이번 발표에서는 인덱스 레인지 스캔, 인덱스 풀 스캔, 루스 인덱스 스캔 세 가지 방식을 다루며, 각각이 **B+Tree 구조 위에서 어떻게 동작하는지**, **EXPLAIN에서 어떻게 보이는지**, 그리고 **왜 성능 차이가 나는지**까지 파고들어 보겠습니다.

예시 테이블은 다음과 같이 사용하겠습니다.

```sql
CREATE TABLE orders (
    id INT PRIMARY KEY AUTO_INCREMENT,
    user_id INT NOT NULL,
    product_id INT NOT NULL,
    amount DECIMAL(10,2) NOT NULL,
    status VARCHAR(20) NOT NULL,
    created_at DATETIME NOT NULL,
    INDEX idx_user_created (user_id, created_at),
    INDEX idx_status (status),
    INDEX idx_product_amount (product_id, amount)
);
-- 약 100만 건의 데이터가 들어있다고 가정합니다.
```

## 2. B+Tree 인덱스 구조 복습

세 가지 스캔 방식을 이해하려면, B+Tree 인덱스가 어떻게 생겼는지를 먼저 짚고 넘어가야 합니다.

InnoDB의 인덱스는 B+Tree 구조입니다. 핵심적인 특징은 다음과 같습니다.

1. **리프 노드에만 실제 데이터(또는 PK)가 존재**합니다. 내부 노드(브랜치 노드)는 탐색을 위한 키 값만 가지고 있습니다.
2. **리프 노드끼리는 Linked List로 연결**되어 있어서, 한 리프에서 다음 리프로 순차 이동이 가능합니다.
3. 클러스터링 인덱스(=PK 인덱스)의 리프 노드에는 실제 행 데이터가 저장되고, 세컨더리 인덱스의 리프 노드에는 **인덱스 키 값 + PK 값**이 저장됩니다.

이 구조를 머릿속에 그려두면 각 스캔 방식이 B+Tree의 어디를 타고 다니는지 자연스럽게 이해됩니다.

```
[브랜치 노드]      [10 | 50 | 90]
                  /     |     \
[리프 노드]  [1,3,7,10] → [12,20,35,50] → [55,70,88,90] → [91,95,99]
              ↑ Linked List로 순차 연결
```

## 3. 인덱스 레인지 스캔 (Index Range Scan)

### 3-1. 정의

인덱스 레인지 스캔은 **인덱스 트리에서 특정 범위의 시작점을 찾아 리프 노드를 따라 순차적으로 스캔**하는 방식입니다.

가장 효율적이고 일반적인 인덱스 활용 방식이며, B+Tree의 장점을 가장 잘 살리는 스캔입니다.

### 3-2. 동작 과정

```sql
SELECT * FROM orders
WHERE user_id = 100 AND created_at BETWEEN '2025-01-01' AND '2025-03-31';
```

이 쿼리가 `idx_user_created (user_id, created_at)` 인덱스를 사용한다면, 동작 과정은 다음과 같습니다.

1. **B+Tree 루트에서 시작하여 `user_id = 100, created_at = '2025-01-01'` 에 해당하는 리프 노드 위치를 탐색**합니다. 이 과정이 Index Seek입니다.
2. **해당 리프 노드에서부터 Linked List를 따라 순차적으로 스캔**합니다.
3. `created_at > '2025-03-31'` 인 키를 만나면 **스캔을 중단**합니다.
4. 스캔한 각 엔트리에서 PK를 얻어 **클러스터링 인덱스에서 실제 행 데이터를 조회**합니다. (`SELECT *` 이므로)

핵심은 **읽어야 할 범위만 정확히 읽고 멈춘다**는 점입니다.

### 3-3. EXPLAIN 결과

```sql
EXPLAIN SELECT * FROM orders
WHERE user_id = 100 AND created_at BETWEEN '2025-01-01' AND '2025-03-31';
```

```
type: range
key: idx_user_created
rows: 245 (예시)
Extra: Using index condition
```

- `type = range` : 인덱스에서 범위 스캔을 수행한다는 의미입니다.
- `key = idx_user_created` : 사용된 인덱스입니다.
- `rows` : 옵티마이저가 예측한 스캔 건수입니다.
- `Extra: Using index condition` : ICP(Index Condition Pushdown)가 적용되어 스토리지 엔진 레벨에서 인덱스 조건을 먼저 필터링합니다.

### 3-4. 왜 빠른가?

레인지 스캔이 빠른 이유는 **불필요한 데이터를 아예 읽지 않기 때문**입니다.

B+Tree에서 시작점까지 내려가는 데 O(log N)이 소요되고, 이후 결과 건수만큼만 리프 노드를 따라 이동합니다. 100만 건 테이블에서 245건만 읽으면 되니까,,, 매우 빠릅니다.

또한 리프 노드가 Linked List로 연결되어 있으므로 순차 I/O에 가깝게 동작하여 디스크 I/O 측면에서도 효율적입니다.

### 3-5. 등호 조건도 레인지 스캔이 될 수 있다

```sql
SELECT * FROM orders WHERE user_id = 100;
```

이 쿼리는 내부적으로는 **"user_id = 100인 범위의 시작점~끝점"을 스캔하는 레인지 스캔과 유사한 메커니즘**입니다.

`ref`와 `range`는 B+Tree에서 시작점을 찾아 리프를 따라 읽는다는 물리적 동작은 유사하지만, 옵티마이저 내부에서는 별개의 접근 방식으로 분류되긴 합니다.

- `ref`의 경우 인덱스에서 해당 키 값의 첫 위치를 찾고, 같은 키 값을 가진 레코드가 끝나는 지점까지 순차적으로 읽습니다. 종료 조건은 "키 값이 달라지는 시점"입니다.
- `range`의 경우 옵티마이저가 범위 조건을 분석하여 시작 키와 종료 키를 미리 계산한 뒤, 해당 구간을 명시적으로 스캔합니다. 종료 조건은 "미리 계산된 종료 키에 도달하는 시점"입니다.

무튼 `ref`는 등호 조건에 특화된 방식이고, `range`는 범위 조건을 위한 방식입니다. MySQL은 `ref`를 `range`보다 효율적인 접근 방식으로 취급합니다.

- MySQL의 EXPLAIN type 컬럼 성능 순위

```
system > const > eq_ref > ref > range > index > ALL
```

## 4. 인덱스 풀 스캔 (Index Full Scan)

### 4-1. 정의

인덱스 풀 스캔은 **인덱스의 리프 노드 전체를 처음부터 끝까지 순차적으로 읽는** 방식입니다.

레인지 스캔이 "필요한 범위만 읽는" 것이라면, 풀 스캔은 "인덱스 전체를 읽는" 것입니다.

### 4-2. 언제 발생하는가?

인덱스 풀 스캔은 주로 다음 상황에서 발생합니다.

1. **쿼리에 필요한 모든 컬럼이 인덱스에 포함되어 있지만, 인덱스의 선행 컬럼 조건이 없는 경우**

```sql
-- idx_user_created (user_id, created_at) 에서
-- user_id 조건 없이 created_at만으로 조회
SELECT user_id, created_at FROM orders
WHERE created_at BETWEEN '2025-01-01' AND '2025-03-31';
```

이 경우 `idx_user_created` 인덱스만으로 결과를 만들 수 있지만(커버링 인덱스), `user_id` 조건이 없기 때문에 인덱스에서 범위의 시작점을 특정할 수 없습니다. 그래서 **인덱스 리프 노드 전체를 순회하면서 `created_at` 조건에 맞는 것만 필터링**합니다.

1. **ORDER BY나 GROUP BY가 인덱스와 일치하지만 WHERE 조건이 없는 경우**

```sql
SELECT user_id, created_at FROM orders
ORDER BY user_id, created_at;
```

왜냐면 인덱스가 어차피 “user_id , created_at” 순서대로 정렬 되어있으니까, 그냥 인덱스 풀 스캔을 수행한 결과가 해당 쿼리의 결과입니다.

옵티마이저는 내부적으로 "테이블 풀 스캔 + 정렬"과 비용을 비교하지만, 커버링 인덱스가 가능한 경우 인덱스 풀 스캔이 정렬 비용까지 제거할 수 있으므로 일반적으로 인덱스 풀 스캔이 선택됩니다.

만약 커버링 인덱스가 적용되지 않는 경우(`SELECT *` 등), 인덱스를 전부 읽은 뒤 테이블까지 조회해야 하므로 옵티마이저가 테이블 풀 스캔을 선택할 가능성이 높습니다. ORDER BY 순서가 인덱스와 다른 경우에도 정렬 제거의 이점이 사라지므로 마찬가지입니다.

### 4-3. 테이블 풀 스캔과의 차이

"어차피 전체를 읽는 거면 테이블 풀 스캔이랑 뭐가 다르지?" 라는 의문이 들 수 있습니다.

핵심적인 차이는 **읽는 데이터의 크기**입니다.

인덱스 풀 스캔은 인덱스 리프 노드만 읽습니다. 인덱스 레코드는 (인덱스 키 + PK) 만 저장하므로, 테이블 레코드보다 훨씬 작습니다.

예를 들어 `orders` 테이블 한 행이 약 100바이트라면, `idx_user_created`의 인덱스 레코드는 (user_id 4B + created_at 8B + PK 4B) = 약 16바이트입니다. 즉, **인덱스 풀 스캔이 테이블 풀 스캔 대비 약 6배 적은 데이터를 읽습니다.**

디스크에서 읽어야 할 페이지 수가 줄어드니 I/O 비용이 크게 줄어드는 것이죠.

### 4-4. EXPLAIN 결과

```sql
EXPLAIN SELECT user_id, created_at FROM orders
WHERE created_at BETWEEN '2025-01-01' AND '2025-03-31';
```

```
type: index
key: idx_user_created
rows: 1000000
Extra: Using where; Using index
```

- `type = index` : 인덱스 풀 스캔입니다.
- `Extra: Using index` : **커버링 인덱스**입니다. 테이블에 접근하지 않고 인덱스만으로 결과를 반환합니다.
- `Extra: Using where` : 인덱스를 읽은 후 WHERE 조건으로 필터링합니다.
- `rows = 1000000` : 인덱스 전체를 스캔한다는 의미입니다.

### 4-5. 왜 레인지 스캔보다 느린가?

원인은 명확합니다. **읽는 건수 자체가 다릅니다.**

레인지 스캔은 조건에 맞는 범위만 읽지만, 풀 스캔은 인덱스 전체를 읽은 후 조건으로 필터링합니다.

100만 건 인덱스에서 실제 결과가 5만 건이라면, 레인지 스캔은 5만 건만 읽고, 풀 스캔은 100만 건을 읽은 후 95만 건을 버리는 것입니다.

그래서 인덱스 풀 스캔이 EXPLAIN에 나타나면, **"이 인덱스의 선행 컬럼 조건이 빠졌구나"** 또는 **"인덱스 설계를 재검토해볼 필요가 있겠다"** 라는 신호로 받아들이는 것이 좋습니다.

## 5. 루스 인덱스 스캔 (Loose Index Scan)

### 5-1. 정의

루스 인덱스 스캔은 **인덱스의 리프 노드를 연속적으로 읽지 않고, 필요한 그룹의 첫 번째 값만 읽고 건너뛰는** 방식입니다.

이름 그대로 "느슨하게(Loose)" 인덱스를 읽는다는 뜻이며, `GROUP BY` + 집계 함수 조합에서 주로 발생합니다.

### 5-2. 동작 과정

```sql
SELECT product_id, MIN(amount) FROM orders
GROUP BY product_id;
```

`idx_product_amount (product_id, amount)` 인덱스가 있다면, 이 인덱스의 리프 노드에는 `(product_id, amount)` 쌍이 `product_id` 순, 같은 `product_id` 내에서는 `amount` 순으로 정렬되어 있습니다.

```
인덱스 리프 노드 (정렬 상태)
[product_id=1, amount=100] → [1, 200] → [1, 350] → ... →
[product_id=2, amount=50]  → [2, 120] → [2, 800] → ... →
[product_id=3, amount=30]  → [3, 90]  → [3, 150] → ...
```

`MIN(amount)`를 구하려면, 각 `product_id` 그룹에서 **첫 번째 엔트리만 읽으면** 됩니다. 이미 `amount` 순으로 정렬되어 있으니까요!

동작 과정은 이렇습니다.

1. `product_id = 1` 의 첫 번째 엔트리 → `amount = 100` → MIN 획득
2. `product_id = 2` 의 첫 번째 엔트리로 **점프** → `amount = 50` → MIN 획득
3. `product_id = 3` 의 첫 번째 엔트리로 **점프** → `amount = 30` → MIN 획득
4. … 반복

**중간의 수많은 엔트리를 모두 건너뛰므로**, 데이터 양에 비해 극적으로 빠릅니다.

### 5-3. EXPLAIN 결과

```sql
EXPLAIN SELECT product_id, MIN(amount) FROM orders
GROUP BY product_id;
```

```
type: range
key: idx_product_amount
rows: 1001 (예: distinct product_id 수 + 1)
Extra: Using index for group-by
```

- `Extra: Using index for group-by` : 이것이 루스 인덱스 스캔을 선택했음을 의미합니다.
- `type = range` : 레인지 스캔과 같은 type으로 표시되지만, Extra를 보면 구분됩니다.
- `rows` : 전체 100만 건이 아니라, distinct한 `product_id` 수 정도만 스캔합니다.

### 5-4. 루스 인덱스 스캔이 가능한 조건

루스 인덱스 스캔은 아무 때나 발생하지 않습니다. MySQL 문서에 따르면 다음 조건을 만족해야 합니다.

1. `GROUP BY`가 인덱스의 **왼쪽 접두사(leftmost prefix)**와 일치해야 합니다.
2. 집계 함수는 `MIN()`, `MAX()`, `COUNT(DISTINCT)`, `SUM(DISTINCT)`, `AVG(DISTINCT)` 만 가능합니다.
    1. `COUNT(DISTINCT)`, `SUM(DISTINCT)`, `AVG(DISTINCT)` 는 왜 가능한가??
        
        인덱스는 키 값 기준으로 정렬되어있습니다. 그래서 같은 값들은 연속으로 모여있습니다.
        
        즉, 같은 값의 첫 번째 엔트리만 읽고, 다음 다른 값으로 점프"하는 방식으로 동작할 수 있습니다.
        
        100, 100, 150, 150, 150, 200, 200, 300 이라면?
        100 → 150 → 200 → 300 으로, 이동이 진행됩니다.
        
3. SELECT 절에 `GROUP BY`에 포함되지 않은 컬럼이 있으면, 그 컬럼이 `MIN()`이나 `MAX()` 안에 있어야 합니다.
4. MIN()/MAX()에 사용되는 컬럼은 인덱스에서 GROUP BY 컬럼 **바로 다음에 위치**해야 합니다.
    
    ex) 인덱스가 `(c1, c2, c3)`이고 `GROUP BY c1`인 경우, `MIN(c3)`은 안 되고 `MIN(c2)`만 가능
    

즉, **인덱스의 정렬 특성을 활용해서 그룹별 첫 번째/마지막 값을 O(1)로 얻을 수 있을 때만** 루스 인덱스 스캔이 선택됩니다.

### 5-5. 루스 인덱스 스캔이 안 되는 경우

```sql
-- SUM(amount)는 그룹 전체를 읽어야 하므로 루스 스캔 불가
SELECT product_id, SUM(amount) FROM orders
GROUP BY product_id;
```

`SUM()`의 경우 각 그룹의 모든 값을 합산해야 하므로, 첫 번째 값만 읽고 건너뛸 수 없습니다. 이 경우에는 **타이트 인덱스 스캔(Tight Index Scan)** 또는 인덱스 풀 스캔이 선택됩니다.

## 6. 세 방식 비교표

| 구분 | 인덱스 레인지 스캔 | 인덱스 풀 스캔 | 루스 인덱스 스캔 |
| --- | --- | --- | --- |
| B+Tree 탐색 | 시작점 Seek → 범위만 순차 | 리프 전체 순차 | 그룹별 첫 값만 점프 |
| EXPLAIN type | range / ref | index | range |
| EXPLAIN Extra | Using index condition 등 | Using index / Using where | Using index for group-by |
| 읽는 건수 | 조건 매칭 건수만 | 인덱스 전체 | distinct 그룹 수 |
| I/O 패턴 | 순차 I/O (부분) | 순차 I/O (전체) | 점프 (그룹 간 건너뛰기, 데이터 분포에 따라 순차에 가까울 수도 있음) |
| 성능 | 가장 효율적 | 테이블 풀스캔보다는 나음 | GROUP BY + MIN/MAX에서 최적 |
| 발생 조건 | WHERE에 인덱스 선행 컬럼 존재 | 커버링 가능 + 선행 컬럼 조건 없음 | GROUP BY + 집계 + 인덱스 정렬 활용 가능 |

## 7. 실행계획 예시와 해석

실제로 EXPLAIN을 찍어보면서 어떤 스캔이 선택되는지 확인해봅시다.

**예시 1: 레인지 스캔**

```sql
EXPLAIN SELECT * FROM orders WHERE user_id = 100;
```

```
type: ref
key: idx_user_created
rows: 980
Extra: NULL
```

`ref`는 non-unique 인덱스에서 등호 조건으로 검색할 때 사용되는 접근 방식입니다. `user_id = 100`인 시작점을 찾고, 같은 값이 끝나는 지점까지 읽습니다.

**예시 2: 인덱스 풀 스캔 vs 테이블 풀 스캔**

```sql
-- 커버링 인덱스 가능 → 인덱스 풀 스캔
EXPLAIN SELECT user_id FROM orders ORDER BY user_id;
-- type: index, Extra: Using index

-- 커버링 불가 → 테이블 풀 스캔
EXPLAIN SELECT * FROM orders ORDER BY user_id;
-- type: ALL (옵티마이저 판단에 따라)
```

두 번째 쿼리에서 `SELECT *`로 인해 인덱스만으로는 결과를 만들 수 없고, 인덱스를 읽은 후 100만 번 랜덤 I/O로 테이블을 조회하는 것보다 차라리 테이블을 순차적으로 풀 스캔하는 게 낫다고 옵티마이저가 판단할 수 있습니다.

**예시 3: 루스 인덱스 스캔**

```sql
EXPLAIN SELECT product_id, MAX(amount) FROM orders GROUP BY product_id;
-- type: range
-- key: idx_product_amount
-- Extra: Using index for group-by
```

`Extra: Using index for group-by`가 보이면 루스 인덱스 스캔입니다. 각 `product_id` 그룹에서 마지막(=MAX) 값만 읽고 다음 그룹으로 점프합니다.

**예시 4: 루스 스캔이 기대되지만 안 되는 경우**

```sql
EXPLAIN SELECT product_id, AVG(amount) FROM orders GROUP BY product_id;
-- type: index
-- key: idx_product_amount
-- Extra: Using index
```

`AVG()`는 그룹 전체 합계와 건수를 알아야 하므로, 각 그룹을 전부 읽어야 합니다. 루스 스캔이 아니라 인덱스 풀 스캔(타이트 인덱스 스캔)이 선택됩니다.

## 8. 실무 관점 정리

### 8-1. 인덱스 설계 시 고려할 점

**선행 컬럼이 핵심이다**

복합 인덱스에서 WHERE 절에 선행 컬럼 조건이 빠지면 레인지 스캔이 불가능합니다. `idx_user_created (user_id, created_at)` 인덱스에서 `user_id` 조건 없이 `created_at`만 검색하면 풀 스캔으로 밀려납니다.

따라서 **가장 자주 사용되는 조건의 컬럼을 인덱스 앞에 배치**하는 것이 중요합니다.

**커버링 인덱스를 활용하자**

인덱스 풀 스캔이더라도 커버링 인덱스라면 테이블 풀 스캔보다 훨씬 낫습니다. 자주 사용하는 쿼리의 SELECT 절에 필요한 컬럼을 인덱스에 포함시키는 전략을 고려할 수 있습니다.

**GROUP BY + MIN/MAX 최적화**

GROUP BY 쿼리에서 `MIN()`이나 `MAX()`를 사용한다면, 해당 컬럼이 GROUP BY 컬럼 바로 다음에 오는 복합 인덱스를 설계하면 루스 인덱스 스캔의 이점을 얻을 수 있습니다.

### 8-2. 헷갈리기 쉬운 포인트 정리

1. **`type = index`는 좋은 게 아니다**
    
    `type = index`가 EXPLAIN에 나오면 "인덱스 쓰고 있으니 괜찮겠지" 라고 생각할 수 있습니다. 하지만 이는 인덱스 **전체**를 읽고 있다는 의미이므로, 데이터가 많으면 성능 문제가 될 수 있습니다.
    
2. **`type = range`인데 루스 스캔일 수 있다**
    
    루스 인덱스 스캔도 `type = range`로 표시됩니다. 반드시 `Extra` 컬럼의 `Using index for group-by`를 확인해야 구분 가능합니다.
    
3. **인덱스 풀 스캔과 테이블 풀 스캔의 선택 기준**
    
    옵티마이저는 "인덱스를 읽고 + 랜덤 I/O로 테이블 접근" vs "테이블을 순차 풀 스캔" 의 비용을 비교합니다. 결과 건수가 전체의 20~30%를 넘으면, 인덱스를 통한 랜덤 I/O보다 테이블 풀 스캔이 더 빠르다고 판단하는 경우도 존재할 수 있다고 합니다.
    
4. **ORDER BY와 인덱스 스캔**
    
    ORDER BY가 인덱스 순서와 일치하면 별도 정렬 없이 인덱스 순서 그대로 결과를 반환합니다. 이때 `Extra: Using filesort`가 사라지는 것을 확인할 수 있습니다. 반대로 인덱스 순서와 다른 정렬을 요구하면 filesort가 발생합니다.
    

## 9. 마치며

오늘 세 가지 인덱스 스캔 방식을 살펴보았습니다.

정리하면, **레인지 스캔**은 B+Tree에서 필요한 범위만 정확히 읽는 가장 효율적인 방식이고, **인덱스 풀 스캔**은 인덱스 전체를 읽지만 테이블 풀 스캔보다는 작은 데이터를 읽으므로 차선의 방식이며, **루스 인덱스 스캔**은 GROUP BY + MIN/MAX 같은 특정 패턴에서 그룹별 첫 값만 읽어 극적인 성능 향상을 제공하는 방식입니다.

결국 이 세 방식 모두 **B+Tree의 정렬 특성과 리프 노드 연결 구조**를 어떻게 활용하느냐의 차이입니다.

EXPLAIN을 찍었을 때 `type`과 `Extra` 컬럼을 함께 읽는 습관을 들이면, 지금 이 쿼리가 인덱스를 얼마나 효율적으로 사용하고 있는지 바로 판단할 수 있을 것입니다.

감사합니다!
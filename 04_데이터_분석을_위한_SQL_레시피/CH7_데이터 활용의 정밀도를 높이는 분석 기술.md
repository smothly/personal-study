# CH7 데이터 활용의 정밀도를 높이는 분석 기술
---

## 17강 데이터를 조합해서 새로운 데이터 만들기

---

### 17-1 ㅇㅇㅇ














## 22강 데이터 마이닝

---

- 대량의 데이터에서 특정 패턴 또는 규칙 등 유용한 지식을 추출하는 방법을 전반적으로 나타내는 용어
- 데이터 마이닝에는 `상관 규칙 추출` `클러스터링` `상관 분석` 등이 있는데 SQL만으로 처리가 힘듬
- 상관 규칙 추출 방법 중 하나인 `어소시에이션 분석` 을 주로 다룸

### 22-1 어소시에이션 분석

- '상품 A를 구매했으면 상품 B를 구매한다'를 찾아내기
- 주요 지표
  - 지지도(support)
    - 상관 규칙이 어느 정도의 확률로 발생하는지 나타내는 값
    - 상품 X와 상품 Y를 같이 구매한 확률
  - 확신도 신뢰도
    - 어떤 결과가 어느 정도의 확률로 발생하는지 나타내는 값
    - 총 구매로그 100개 / Y 구매 20 / X 구매 50 = 40%
  - 리프트(lift)
    - '어떤 조건을 만족하는 경우의 확률' / '사전 조건 없이 해당 결과가 일어날 확률'
    - 총 구매로그 100개 / X만 구매 50개, X Y 같이 구매하는 로그 20개, Y만 구매 50개
    - 확신도(40%) / 상품 Y 구매확률(20%) = 2.0
    - 1.0 이상이면 좋은 규칙이라고 판단
- 두 상품의 연관성을 어소시에이션 분석으로 찾기
  -  구매 로그 수와 상품별 구매 수를 세는 쿼리
    
    ```sql
    WITH
    purchase_id_count AS (
    -- 구매 상세 로그에서 유니크한 구매 로그 수 계산하기
    SELECT COUNT(DISTINCT purchase_id) AS purchase_count
    FROM purchase_detail_log
    )
    , purchase_detail_log_with_counts AS (
    SELECT
        d.purchase_id
        , p.purchase_count
        , d.product_id
        -- 상품별 구매 수 계산하기
        , COUNT(1) OVER(PARTITION BY d.product_id) AS product_count)
    FROM
        purchase_detail_log AS d
        CROSS JOIN
        -- 구매 로그 수를 모든 레코드 수와 결합하기
        purchase_id_count AS p
    )
    SELECT
    *
    FROM
    purchase_detail_log_with_counts
    ORDER BY
    product_id, purchase_id
    ;
    ```

    - 상품 조합별로 구매 수를 세는 쿼리

    ```sql
    WITH
    purchase_id_count AS (
    -- CODE.22.1.
    )
    , purchase_detail_log_with_counts AS (
    -- CODE.22.1.
    )
    , product_pair_with_stat AS (
    SELECT
        l1.product_id AS p1
        , l2.product_id AS p2
        , l1.product_count AS p1_count
        , l2.product_count AS p2_count
        , COUNT(1) AS p1_p2_count
        , l1.purchase_count AS purchase_count
    FROM
        purchase_detail_log_with_counts AS l1
        JOIN
        purchase_detail_log_with_counts AS l2
        ON l1.purhcase_id = l2.purchase_id
    WHERE
        -- 같은 상품 조합 제외하기
        l1.product_id <> l2.product_id
    GROUP BY
        l1.product_id
        , l2.product_id
        , l1.product_count
        , l2.product_count
        , l1.purchase_count
    )
    SELECT
    *
    FROM
    product_pair_with_stat
    ORDER BY
    p1, p2
    ;
    ```

    - 지지도, 확산도, 리프트를 계산하는 쿼리

    ```sql
    WITH
    purchase_id_count AS (
    -- CODE.22.1.
    )
    , purchase_detail_log_with_counts AS (
    -- CODE.22.1.
    )
    , product_pair_with_stat AS (
    -- CODE.22.2.
    )
    SELECT
    p1
    , p2
    , 100.0 * p1_p2_count / purchase_count AS support
    , 100.0 * p1_p2_count / p1_count AS confidence
    , (100.0 * p1_p2_count / p1_count)
        / (100.0 * p2_count / purchase_count) AS lift
    FROM
    product_pair_with_stat
    ORDER BY
    p1, p2
    ;
    ```

- 어소시에이션 분석은 두 상품의 상관 규칙만을 주목한 한정적인 분석 방법
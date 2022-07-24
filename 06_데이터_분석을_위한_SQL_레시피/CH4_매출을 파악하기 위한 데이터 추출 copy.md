# CH4_매출을 파악하기 위한 데이터 추출

---

## 9강 시계열 기반으로 데이터 집계하기

- 시계열로 매출을 집계함녀 규칙성이나, 기간간의 비교 등을 할 수 있음

#### 9-1 날짜별 매출 집계하기

- 가로 축에 날짜, 세로 축에 금액
  - 날짜별 매출과 평균 구매액을 집계하는 쿼리

  ```SQL
  SELECT
    dt
    , COUNT(*) AS purchase_count
    , SUM(purchase_amount) AS total_amount
    , AVG(purchase_amount) AS avg_amount
  FROM purchase_log
  GROUP BY dt
  ORDER BY dt
  ```

#### 9-2 이동평균을 사용한 날짜별 추이 보기

- 매출이 상승하는 경향인지 하락하는 경향인지 파악하기 위함
  - 날짜별 매출과 7일 이동평균을 집계하는 쿼리
    - seven_day_avg는 7일간의 데이터가 없으면 현재데이터로 보정이 들어감

  ```SQL
  SELECT
  dt
  , SUM(purchase_amount) AS total_amount

  -- 최근 최대 7일 동안의 평균 계산
  , AVG(SUM(purchase_amount))
    OVER(ORDER BY dt ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)
  AS seven_day_avg

  -- 최근 7일 동안의 평균을 확실히 계산
  , CASE
    WHEN
      7 = COUNT(*)
      OVER(ORDER BY dt ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)
    THEN
      AVG(SUM(purchase_amount))
      OVER(ORDER BY dt ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)
    END
    AS seven_day_avg_strict
  FROM purchase_log
  GROUP BY dt
  ORDER BY dt
  ;
  ```
  
#### 9-3 당월 매출 누계 구하기

- 윈도 함수 사용하여 매출의 누계 구하기
  - 날짜별 매출과 당월 누계 매출을 집계하는 쿼리
    - 연월을 기준으로 파티션을 생성해서 이전까지 합을 누계하는 식
    - 가독성을 높일려면 연월일 빼는 함수들을 WITH절로 빼면 됨

  ```SQL
  SELECT
    dt
    -- PostgreSQL, Hive, Redshift, SparkSQL의 경우 substring로 연-월 추출
    , substring(dt, 1, 7) AS year_month

    -- PostgreSQL, Hive, Bigquery, SparkSQL의 경우 substr 함수 사용
    , substr(dt, 1, 7) AS year, month

    , SUM(purchase_amount) AS total_amount
    , SUM(SUM(purchase_amount))
      -- PostgreSQL, Hive, Redshift, SparksQL
      OVER(PARTITION BY substring(dt, 1, 7) ORDER BY dt ROWS UNBOUNDED PRECEDING)

      -- BigQuery의 경우 substring을 substr로 수정
      OVER(PARTITION BY substr(dt, 1, 7) ORDER BY dt ROWS UNBOUNDED PRECEDING)
      AS agg_amount
    FROM purchase_log
  GROUP BY dt
  ORDER BY dt
  ;
  ```

  - 날짜별 매출을 일시 테이블로 만드는 쿼리

  ```SQL
  WITH
  daily_purchase AS (
    SELECT
      dt
      -- 연, 월, 일을 각각 추출
      -- PostgreSQL, Hive, Redshift, SparkSQL은 다음과 같이 작성
      -- BigQuery의 경우 substring -> substr
      , substring(dt, 1, 4) AS year
      , substring(dt, 6, 2) AS month
      , substring(dt, 9, 2) AS date
      , SUM(purchase_amount) AS purchase_amount
      , COUNT(order_id) AS orders
    FROM purchase_log
    GROUP BY dt
  )

  SELECT *
  FROM daily_uprchase
  ORDER BY dt;
  ```

  - daily_purchase 테이블에 대해 당월 누계 매출을 집계하는 쿼리(최종본!!)

  ```SQL
  WITH
  daily_purchase AS (
    SELECT
      dt
      -- 연, 월, 일을 각각 추출
      -- PostgreSQL, Hive, Redshift, SparkSQL은 다음과 같이 작성
      -- BigQuery의 경우 substring -> substr
      , substring(dt, 1, 4) AS year
      , substring(dt, 6, 2) AS month
      , substring(dt, 9, 2) AS date
      , SUM(purchase_amount) AS purchase_amount
      , COUNT(order_id) AS orders
    FROM purchase_log
    GROUP BY dt
  )
  SELECT
    dt
    , concat(year, '-', month) AS year_month
    
    --Redshift의 경우, concat 함수를 조합해서 사용하거나 || 연산자 사용
    , concat(concat(year, '-'), month) AS year_month
    , year || '-' || month AS year_month

    , purchase_amount
    , SUM(purchase_amount)
        OVER(PARTITION BY year, month ORDER BY dt ROWS UNBOUNDED PRECEDING)
      AS agg_amount
    FROM daily_purchase
    ORDER BY dt;
  ```

#### 9-4 월별 매출의 작대비 구하기

- 작년의 해당 월의 매출과 비교
- JOIN을 사용하지 않고 작대비를 계산하는 방법 소개
  - 월별 매출과 작대비를 계산하는 쿼리

  ```SQL
  WITH
  daily_purchase AS (
    ...(이전 코드와 같음)
  )

  SELECT
    month
    , SUM(CASE year WHEN '2014' THEN purchase_amount END) AS amount_2014
    , SUM(CASE year WHEN '2015' THEN purchase_amount END) AS amount_2015
    , 100.0
      * SUM(CASE year WHEN '2015' THEN purchase_amount END)
      / SUM(CASE year WHEN '2014' THEN purchase_amount END)
    AS rate
  FROM
    daily_purchase
  GROUP BY month
  ORDER BY month
  ;
  ```

#### 9-5 Z차트로 업적의 추이 확인하기

- `월차매출` `매출누계` `이동년계` 라는 3개의 지표로 구성하여 계절 변동의 영향을 배제하고 트렌드를 분석하는 방법
  - ![Z차트에 대한 설명](https://online.fliphtml5.com/hkuy/msoq/files/large/52.jpg)
  - Z차트 분석할 때
    - 매출누계
      - 월차매출이 일정하면 직선이되고 기울기가 급해지면 상승 완만하면 하락
    - 이동년계
      - 작년과 올해의 매출이 일정하면 직선
      - 오른쪽 위로 올라가면 매출 오름, 내려가면 매출 감소
    - ![Z차트 예시](https://mblogthumb-phinf.pstatic.net/20141127_136/socialmedia_1417097904019q7BqE_PNG/%BD%BA%C5%A9%B8%B0%BC%A6_2014-11-27_%BF%C0%C8%C4_11.18.07.png?type=w2)
  - 2015년 매출에 대한 Z차트를 작성하는 쿼리

  ```SQL
  WITH
  daily_purchase AS (
    ...(이전 코드와 같음)
  )
  , monthly_amount AS (
    -- 월별 매출 집계
    SELECT
      year
      , month
      , SUM(purchase_amount) AS amount
    FROM daily_purhase
    GROUP BY year, month
  )
  , calc_index AS (
    SELECT
      year
      , month
      , amount
      -- 2015년의 누계 매출 집계
      , SUM(CASE WHEN year='2015' THEN amount END)
        OVER(ORDER BY year, month ROWS UNBOUNDED PRECEDING)
        AS agg_amount

      -- 당월부터 11개월 이전까지의 매출 합계(이동년계) 집계
      , SUM(amount)
        OVER(ORDER BY year, month ROWS BETWEEN 11 PRECEDING AND CURRENT ROW)
        AS year_avg_amount
      FROM
        monthly_purchase
      ORDER BY
        year, month
  )

  -- 2015년의 데이터 압축
  SELECT
    concat('year', '-', month) AS year_month
    -- Redshift의 경우 concat 함수를 조합하거나, || 연산자 사용
    concat(concat(year, '-'), month) AS year_month
    year || '-' || month AS year_month

    , amount
    , agg_amount
    , year_avg_amount
  FROM cal_index
  WHERE year = '2015'
  ORDER BY year_month
  ;
  ```

#### 9-6 매출을 파악할 때 중요 포인트

- 주변 데이터를 함께 포함해서 리포트를 만드는게 좋음 ex) 판매 횟수, 평균 구매액, 구매 단가 등
  - 매출과 관련된 지표를 집계하는 쿼리

  ```SQL
  WITH
  daily_purchase AS (
    ...
  )
  , monthly_purchase AS (
    SELECT
      year
      , month
      , SUM(orders) AS orders
      , AVG(purchase_amount) AS avg_amount
      , SUM(purchase_amount) AS monthly
    FROM daily_purchase
    GROUP BY year, month
  )

  SELECT
    concat(year, '-', month) AS year_month
    -- Redshift의 경우 concat 함수를 조합하거나, || 연산자 사용
    concat(concat(year, '-'), month) AS year_month
    year || '-' || month AS year_month

    , orders
    , avg_amount
    , monthly
    , SUM(monthly)
      OVER(PARTITION BY year ORDER BY month ROWS UNBOUNDED PRECEDING) AS agg_amount

    -- 12개월 전의 매출 구하기
    , LAG(monthly, 12)
      OVER(ORDER BY year, month)
    -- sparkSQL의 경우 다음과 같이 사용
      OVER(ORDER BY year, month ROWS BETWEEN 12 PRECEDING AND 12 PRECEDING)
      AS last_year
    , 100.0
      * monthly
      / LAG(monthly, 12)
        OVER(ORDER BY year, month)
      -- sparkSQL의 경우 다음과 같이 사용
      OVER(ORDER BY year, month ROWS BETWEEN 12 PRECEDING AND 12 PRECEDING)
      AS rate
  FROM monthly_purchase
  ORDER BY year, month
  ;

  -- PRECEDING : 이전
  -- LAG : 현재 행 기준 n번 전
  ```

---

## 10강 다면적인 축을 사용해 데이터 집약하기

- 매출의 시계열뿐만 아니라 상품의 카테고리, 가격 등을 조합해서 데이터의 특징을 추출해 리포팅

#### 10-1 카테고리별 매출과 소계 계산하기

- 드릴다운 형식으로 출력하기
  - 카테고리별 매출과 소계를 동시에 구하는 쿼리
    - `UNION ALL` 을 사용하는 건 테이블을 여러 번 불러오고 결합하는 비용이 발생하여 좋지 않음

  ```SQL
  WITH
  sub_category_amount AS (
    -- 소카테고리 매출 집계
    SELECT
      category AS category
      , sub_category AS sub_category
      , SUM(price) AS amount
    FROM
      purchase_detail_log
    GROUP BY
      category, sub_category
  ),
  category_amount AS (
  -- 대카테고리 매출 집계
  SELECT
    category
    , 'all' AS sub_category
    , SUM(price) AS amount
  FROM
    purchase_detail_log
  GROUP BY
    category
  )
  , total_amount AS (
    -- 전체 매출 집계
    SELECT
      'all' AS category
      , 'all' AS sub_category
      , SUM(price) AS amount
    FROM
      purchase_datail_log
  )
            SELECT category, sub_category, amount FROM sub_category_amount
  UNION ALL SELECT category, sub_category, amount FROM category_amount
  UNION ALL SELECT category, sub_category, amount FROM total_amount
  ;
  ```

  - ROLLUP을 사용해서 카테고리별 매출과 소계를 동시에 구하는 쿼리
    - PostgreSQL, Hive SparkSQL에 성능이 좋음
    - 레코드 집계 키가 NULL이 돼서 COALESCE함수로 'all'로 변환

  ```SQL
  SELECT
    COALESCE(category, 'all') AS category
    , COALESCE(sub_category, 'all') AS sub_category
    , SUM(price) AS amount
  FROM
    purchase_detail_log
  GROUP BY
    ROLLUP(category, sub_category)
    -- HIVE 에서는 다음과 같이 사용
    category, sub_category WITH ROLLUP
  ```

#### 10-2 ABC 분석으로 잘 팔리는 상품 판별하기

- ![ABC분석 예시](https://velog.velcdn.com/images%2Fjaytiger%2Fpost%2Fc4163f2b-79d4-4ebb-bdb3-2ddec1ea1e39%2Fabc_chart.png)
- 재고 관리에서 사용하는 분석 방법
- 매출 구성을 파악하기에 용이
- 데이터를 작성하는 방법
  1. 매출이 높은 순서로 정렬
  2. 매출 합계 집계
  3. 매출 합계를 기반으로 각 데이터가 차지하는 비율 계산과 구성비를 구함
  4. 구성비 누계를 구함
  - 매출 구성비 누계와 ABC 등급을 계산하는 쿼리

  ```SQL
  WITH
  monthly_sales AS (
    SELECT
      category1
      -- 항목별 매출 계산
      , SUM(amount) AS amount
    FROM
      purchase_log
    -- 대상 1개월 동안의 로그를 조건으로 걸기
    WHERE
      dt BETWEEN '2015-12-01' AND '2015-12-31'
    GROUP BY
      category1
  )
  , sales_composition_ratio AS (
    SELECT
      category1
      , amount

      -- 구성비 : 100.0 * 항목별 매출 / 전체 매출
      , 100*0 * amount / SUM(amount) OVER() AS composition_ratio

      -- 구성비누계 : 100.0 * 항목별 구계 매출 / 전체 매출
      , 100*0 * SUM(amount) OVER(ORDER BY amount DESC
      ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
        / SUM(amount) OVER() AS cumulative_ratio
    FROM
      monthly_sales
  )
  SELECT
    *
    -- 구성비누계 범위에 따라 순위 붙이기
    , CASE
      WHEN cumulative_ratio BETWEEN 0 AND 70 THEN 'A'
      WHEN cumulative_ratio BETWEEN 70 AND 90 THEN 'B'
      WHEN cumulative_ratio BETWEEN 90 AND 100 THEN 'C'
    END AS abc_rank
  FROM
    sales_composition_ratio
  ORDER BY
    amount DESC
  ;
  ```

#### 10-3 팬 차트로 상품의 매출 증가율 확인하기

- 변화를 백분율로하여 작은 변화도 쉽게 인지하고 상황을 판단할 수 있음
- ![팬차트 예시](https://velog.velcdn.com/images%2Fjaytiger%2Fpost%2Fcbbe9d74-aa80-4e72-b9c9-6d058dc87da1%2Ffan_chart2.png)
  - 팬 차트 작성 때 필요한 데이터를 구하는 쿼리
    - `FIRST_VALUE` 함수를 사용해서 2014년 1월 매출 기준의 비율을 계산

  ```SQL
  WITH
  daily_category_amount AS (
    SELECT
    dt
    , cateogry
    -- PostgreSQL, Hive, Redshift, SparkSQL은 다음과 같잉 구성
    -- BigQuery의 경우 substring을 sdubstr로 수정
    , substring(dt, 1, 4) AS year
    , substring(dt, 6, 2) AS month
    , substring(dt, 9, 2) AS date
    , SUM(price) AS amount
  FROM purchase_detail_log
  GROUP BY dt, category
  )
  , monthly_category_amount AS (
    SELECT
      concat(year, '-', month) AS year_month
      -- Redshift의 경우 concat 함수를 조합해서 사용하거나 || 연산자 사용
      -- concat(concat(year, '-'), month) AS year_month
      -- year || '-' || month AS year_month
      , category
      , SUM(amount) AS amount
    FROM
      daily_category_amount
    GROUP BY
      year, month, category
  )

  SELECT
    year_month
    , category
    , amount
    -- amount의 첫번째 값을 구한다
    -- CATEGORY로 구분하여, year_month, category로 정렬하며, 이전행 전부를 대상으로 함
    , FIRST_VALUE(amount)
      OVER(PARTITION BY category ORDER BY year_month, category ROWS UNBOUNDED PRECEDING) AS base_amount
    , 100.0 * amount / FIRST_VALUE(amount)
        OVER(PARTITION BY category ORDER BY year_month, category ROWS UNBOUNDED PRECEDING) AS rate
  FROM
    monthly_category_amount
  ORDER BY
    year_month, category
  ;
  ```

#### 10-4 히스토그램으로 구매 가격대 집계하기

- 상품의 가격에 주목해 데이터 분포를 확인할 수 있는 히스토그램 작성 방법
- ![히스토그램 예시](https://t3.daumcdn.net/thumb/R720x0.fpng/?fname=http://t1.daumcdn.net/brunch/service/user/QI8/image/EKo2Q-ejY7T7-DTX9zIVbGRAh1A.png)
- 히스토그램 만드는 방법
  1. 최댓값, 최솟값 범위를 구한다.
  2. 몇 개의 계급으로 나눌지, 각 계급의 하한과 상한을 구한다.
  3. 각 계급에 들어가는 데이터 개수를 구한다.
  - 최댓값, 최솟값, 범위를 구하는 쿼리
    - 대부분 히스토그램 작성하는 함수를 표준으로 제공함 ex) psql의 width_bucket함수

  ```SQL
  WITH
  stats AS (
    SELECT
      MAX(price) AS max_price
      , MIN(price) AS min_price
      , MAX(price) - MIN(price) AS range_price
      -- 계층 수
      , 10 AS bucket_num
    FROM
      purchase_detail_log
  )

  SELECT * FROM stats;
  ```

  - 데이터의 계층을 구하는 쿼리
    - 계급 범위를 10으로 지정해 최댓값인 35000이 계급 11로 지정됨

  ```SQL
  WITH
  stats AS (
    -- ...
  )
  , purchase_log_with_bucket AS (
    SELECT
      price
      , min_price
      -- 정규화 금액: 대상 금액 - 최소 금액
      , price - min_price AS diff
      -- 계층 범위 : 금액 범위를 계층 수로 분할
      , 1.0 * range_price / bucket_num AS bucket_range

      -- 계층 판정 : FLOOR(정규화 금액 / 계층 범위)
      , FLOOR(
          1.0 * (price - min_price) / (1.0 * range_price / bucket_num)
          -- index가 1부터 시작하므로, 1을 더함
      ) + 1 AS bucket

      -- PostgreSQL, width_bucket 함수 사용
      , width_bucket(price, min_price, max_price, bucket_num) AS bucket
    FROM
      purchase_Detail_log, stats
  )
  SELECT *
  FROM purchase_log_with_bucket
  ORDER BY amount
  ;
  ```

  - 계급 상한 값을 조정한 쿼리
    - 금액의 최댓값 + 1 하여 상한 미만에 속하도록 변경

  ```SQL
  WITH
  stats AS (
    SELECT
      -- 금액의 최댓값 + 1
      MAX(price) + 1 AS max_price
      -- 금액의 최솟값
      , MIN(price) AS min_price
      -- 금액의 범위 + 1(실수)
      , MAX(price) + 1 - MIN(price) AS range_price
      -- 계층 수
      , 10 AS bucket_num
    FROM
      purchase_detail_log
  )
  purchase_log_with_bucket AS (
    -- 이전과 동일 
    ...
  )
  SELECT *
  FROM purchase_log_with_bucket
  ORDER BY price
  ;
  ```

- 히스토그램을 구하는 쿼리

```SQL
WITH
stats AS (
  ...
)
, purchase_log_with_bucket AS (
  ...
)
SELECT
  bucket
  -- 계층의 하한과 상한 계산
  , min_price + bucket_range * (bucket - 1) AS lower_limit
  , min_price + bucket_range * bucket AS upper_limit
  -- 도수 세기
  , COUNT(price) AS num_purchase
  -- 합계 금액 계산
  , SUM(price) AS total_amount
FROM
  purchase_log_with_bucket
GROUP BY
  bucket, min_price, bucket_range
ORDER BY bucket
;
```

- 히스토그램의 상한과 하한을 수동으로 조정한 쿼리

```SQL
WITH
stats AS (
  SELECT
  -- max, min, range, range_count
  50000 AS max_price
  , 0 AS min_price
  , 5000 AS range_price
  , 10 AS bucket_num
  FROM
    purchase_detail_log
)
, purchase_log_with_bucket AS (
  ...
)
SELECT
  bucket
  -- 계층의 상한, 하한 계산
  , min_price + bucket_range * (bucket -1) AS lower_limit
  , min_price + bucket_range * bucket AS upper_limit
  -- 도수 세기
  , COUNT(price) AS num_purchase
  -- 합계 금액 계산
  , SUM(price) AS total_amount
FROM
  purchase_log_with_bucket
GROUP BY
  bucket, min_price, bucket_range
ORDER BY
  bucket
;
```

- 히스토그램이 2개의 산으로 나누어진 경우 필터링을 걸어 어떤 부분에서 차이가 있는지 확인하기!

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
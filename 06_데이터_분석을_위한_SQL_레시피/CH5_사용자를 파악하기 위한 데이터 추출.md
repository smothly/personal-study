# CH5_사용자를 파악하기 위한 데이터 추출

---

## 11강 사용자 전체의 특징과 경향 찾기

- 사용자의 속성(나이, 성별, 주소 등)과 행동(구매한 상품, 기능, 사용 빈도 등 )을 조사하여 서비스를 개선

### 11-1 사용자의 액션 수 집계하기

- 액션과 관련된 지표 집계하기
  - 액션 Unique User / 액션 수 / 사용률 / 1명당 액션 수
  - 액션 수와 비율을 계산하는 쿼리

  ```SQL
  WITH
  stats AS (
    -- 로그 전체의 유니크 사용자 수 구하기
    SELECT COUNT(DISTINCT session) AS total_uu
    FROM action_log
  )
  SELECT
    l.action
    -- action uu
    , COUNT(DISTINCT l.session) AS action_uu
    -- action count
    , COUNT(1) AS action_count
    -- total uu
    , s.total_uu
    -- usage_rate
    , 100.0 * COUNT(DISTINCT l.session) / s.total_uu AS usage_rate
    -- count_per_user
    , 1.0 * COUNT(1) / COUNT(DISTINCT l.session) AS count_per_user
  FROM
    action_log AS l
    -- 로그 전체의 유니크 사용자 수를 모든 레코드에 결합
    CROSS JOIN 
          stats AS s
    GROUP BY
      l.action, s.total_uu
    ;
  ```

- 로그인 사용자와 비로그인 사용자를 구분해서 집계하기
  - `user_id`가 없는 경우 비로그인으로 구분
  - 로그인 상태를 판별하는 쿼리

  ```SQL
  WITH
  action_log_with_status AS (
    SELECT
      session
      , user_id
      , action
      -- user_id = NULL OR user_id != '', login
      , CASE WHEN COALESCE(user_id, '') <> '' THEN 'login' ELSE 'guest' END
        AS login_status
      FROM action_log
    )
  SELECT *
  FROM
    action_log_with_status
  ;
  ```

  - 로그인 상태에 따라 액션 수 등을 따로 집계하는 쿼리
    - 로그인은 user_id all은 session 기반으로 집계하므로 비로그인 사용자가 로그인하면 count가 중복되는 문제가 생김

  ```SQL
  WITH
  action_log_with_status AS (
    ...
  )
  SELECT
    COALESCE(action, 'all') AS action
    , COALESCE(login_status, 'all') AS login_status
    , COUNT(DISTINCT session) AS action_uu
    , COUNT(1) AS action_count
  FROM
    action_log_with_status
  GROUP BY
    -- PostgreSQL, SparkSQL
    ROLLUP(action, login_status)
    -- Hive
    action, login_status WITH ROLLUP
  ;
  ```

- 이전에 한 번이라도 로그인 했다면 회원으로 간주
  - 회원 상태를 판별하는 쿼리

  ```SQL
  WITH
  action_log_with_status AS (
    SELECT
      session
      , user_id
      , action
      -- log를 timestamp 순으로 나열, 한번이라도 로그인했을 경우,
      -- 이후의 모든 로그 상태를 member로 설정
      , CASE
        WHEN
            COALESCE(MAX(user_id)
              OVER(PARTITION BY session ORDER BY stamp
                ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
              , '') <> ''
            THEN 'member'
          ELSE 'none'
        END AS member_status
      , stamp
    FROM
      action_log
    )
    SELECT *
    FROM
      action_log_with_status
    ;
  ```

### 11-2 연령별 구분 집계하기

- 사용자 속성을 정의하고 집계하면 다양한 리포트를 만들 수 있음
  - 사용자의 생일을 계산하는 쿼리

  ```SQL
  WITH
  -- 생일과 특정 날짜를 정수로 표현한 결과
  mst_users_with_int_birth_date AS (
    SELECT
      * 
      -- 특정 날짜의 정수 표현(2017.01.01)
      , 20170101 AS int_specific_date
      
      -- 문자열로 구성된 생년월일 -> 정수 표현 변환
      -- PostgreSQL, Redshift
      , CAST(replace(substring(birth_date, 1, 10), '-', '') AS integer) AS int_birth_date
      -- BigQuery
      , CAST(replace(substr(birth_date, 1, 10), '-', '') AS int64) AS int_birth_date
      -- Hive, SparkSQL
      , CAST(regexp_replace(substring(birth_date, 1, 10), '-', '') AS int) AS int_birth_date
    FROM
      mnt_users
  )
  -- 생일 정보를 부여한 사용자 마스터
  , mst_users_with_age AS (
    SELECT
      *
      -- 2017.01.01의 나이
      , floor((int_specific_date - int_birth_date) / 10000) AS age
    FROM mst_users_with_int_birth_date
  )
  SELECT
    user_id, sex, birth_date, age
  FROM
    mst_users_with_age
  ;
  ```

  - 성별과 연령으로 연령별 구분을 계산하는 쿼리

    ```SQL
    WITH
  mst_users_with_int_birth_date AS (
    ...
  )
  , mst_users_with_age AS (
    ...
  )
  , mst_users_with_category AS (
    SELECT
      user_id
      , sex
      , age
      , CONCAT (
        CASE
          WHEN 20 <= age THEN sex
          ELSE ''
        END
      , CASE
          WHEN age BETWEEN 4 AND 12 THEN 'C'
          WHEN age BETWEEN 13 AND 19 THEN 'T'
          WHEN age BETWEEN 20 AND 34 THEN '1'
          WHEN age BETWEEN 35 AND 49 THEN '2'
          WHEN age >= 50 THEN '3'
        END
      ) AS category
    FROM
      mst_users_with_age
  )
  SELECT
    category
    , COUNT(1) AS user_count
  FROM
    mst_users_with_category
  GROUP BY
    category
  ;
  ```

### 11-3 연령별 구분의 특징 추출하기

- 이전 절에서 사용한 연령별 구분을 사용해서 각각 구매한 상품의 카테고리를 집계
  - 연령별 구분과 카테고리를 집계하는 쿼리

  ```SQL
  WITH
  mst_users_with_int_birth_date AS (
    ...
  )
  , mst_users_with_age AS (
    ...
  )
  , mst_users_with_category AS (
    ...
  )
  SELECT
    p.category AS product_category
    , u.category AS user_category
    , COUNT(*) AS purchase count
  FROM
    action_log AS p
  JOIN
    mst_users_with_category AS u ON p.user_id = u.user_id
  WHERE
    -- 구매 로그만 선택
    action = 'purchase'
  GROUP BY
    p.category, u.category
  ORDER BY
    p.category, u.category
  ;
  ```

- 연령과 카테고리를 축별로 바꿔가며 비교해야 좋은 인사이트를 얻을 수 있음.

### 11-4 사용자의 방문 빈도 계산하기

- 서비스를 한 주 동안 며칠 사용하는 사용자가 몇 명인지 집계
- 일주일 동안의 사용자 사용 일수와 구성비
  - 한 주에 며칠 사용되었는지를 집계하는 쿼리

  ```SQL
  WITH
  action_log_with_dt AS (
    SELECT *
    -- 타임 스탬프에서 날짜 추출하기
    -- PostgreSQL, Hive, Redshift, SparkSQL의 경우 substring으로 날짜 추출
    , substring(stamp, 1, 10) AS dt
    -- PostgerSQL, Hive, BigQuery, SparkSQL의 경우 substr 사용
    , substr(stamp, 1, 10) AS dt
    FROM action_log
  )
  , action_day_count_per_user AS (
    SELECT
      user_id
      , COUNT(DISTINCT dt) AS action_day_count
    FROM
      action_log_with_dt
    WHERE
      -- 2016년 11월 1일부터 11월 7일까지의 한 주 동안을 대상으로 지정
      dt BETWEEN '2016-11-01' AND '2016-11-07'
    GROUP BY
      user_id
  )
  SELECT
    action_day_count
    , COUNT(DISTINCT user_id) AS user_count
  FROM
    action_day_count_per_user
  GROUP BY
    action_day_count
  ORDER BY
    action_day_count
  ;
  ```

  - 구성비와 구성비누계를 계산하는 쿼리

  ```SQL
  WITH
  action_day_count_per_user AS (
    ...
  )
  SELECT
    action_day_count
    , COUNT(DISTINCT user_id) AS user_count

    -- 구성비
    , 100.0
      * COUNT(DISTINCT user_id)
      / SUM(COUNT(DISTINCT user_id)) OVER() AS composition_ratio
    
    -- 구성비누계
    , 100.0
      * SUM(COUNT(DISTINCT user_id))
        OVER(ORDER BY action_day_count
          ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
      / SUM(COUNT(DISTINCT user_id)) OVER() AS cumulative_ratio
  FROM
    action_day_count_per_user
  GROUP BY
    action_day_count
  ORDER BY
    action_day_count
  ;
  ```

### 11-5 벤 다이어그램으로 사용자 액션 집계하기

- 여러 기능의 사용 상황을 조사한 뒤 사용자의 액션 파악
- purchase, review, favorite 3개의 액션 로그 파악
  - 사용자들의 액션 플래그를 집계하는 쿼리

  ```SQL
  WITH
  user_action_flag AS (
    -- 사용자가 액션을 했으면 1, 하지 않았을 경우 0 플래그
    SELECT
      user_id
      , SIGN(SUM(CASE WHEN action = 'purchase' THEN 1 ELSE 0 END)) AS has_purchase
      , SIGN(SUM(CASE WHEN action = 'review' THEN 1 ELSE 0 END)) AS has_review
      , SIGN(SUM(CASE WHEN action = 'favorite' THEN 1 ELSE 0 END)) AS has_favorite
    FROM
      action_log
    GROUP BY
      user_id
  )
  SELECT *
  FROM user_action_flag;
  ```

  - 모든 액션 조합에 대한 사용자 수 계산하기
    - `CUBE` 함수는 psql만 가능

  ```SQL
  WITH
  user_action_flag AS (
    ...
  )
  , action_venn_diagram AS (
    -- cube를 사용하여 모든 액션 조합 구하기
    SELECT
      has_purchase
      , has_review
      , has_favorite
      , COUNT(1) AS users
    FROM
      user_action_flag
    GROUP BY
      CUBE(has_purchase, has_revicew, has_favorite)
  )
  SELECT *
  FROM action_venn_diagram
  ORDER BY
    has_purchase, has_review, has_favorite
  ;
  ```

  - `CUBE` 구문을 사용하지 않고 표준 SQL 구문만으로 작성한 쿼리
    - `UNION ALL` 을 많이 사용하는 쿼리로 성능이 좋지 않음

  ```SQL
  WITH
  user_adction_flag AS (
    ...
  )
  , action_venn_diagram AS (
    -- 모든 액션 조합을 개별적으로 구하고 UNION ALL로 결합

    -- 3개의 액션을 모두 한 경우 집계
    SELECT has_purhcase, has_review, has_favorite, COUNT(1) AS users
    FROM user_action_flag
    GROUP BY has_purchase, has_review, has_favorite

    -- 3개의 액션 중에서 2개의 액션을 한 경우 집계
    UNION ALL
      SELECT NULL AS has_purhcase, has_review, has_favorite, COUNT(1) AS users
      FROM user_action_flag
      GROUP BY has_review, has_favorite
    UNION ALL
      SELECT has_purchase, NULL AS has_review, has_favorite, COUNT(1) AS users
      FROM user_action_flag
      GROUP BY has_purchase, has_favorite
    UNION ALL
      SELECT has_purhcase, has_review, NULL AS has_favorite, COUNT(1) AS users
      FROM user_action_flag
      GROUP BY has_purchase, has_review
    
    -- 3개의 액션 중에서 하나의 액션을 한경우 집계
    UNION ALL
      SELECT NULL AS has_purhcase, NULL AS has_review, has_favorite, COUNT(1) AS users
      FROM user_action_flag
      GROUP BY has_favorite
    UNION ALL
      SELECT NULL AS has_purhcase, has_review, NULL AS has_favorite, COUNT(1) AS users
      FROM user_action_flag
      GROUP BY has_review
    UNION ALL
      SELECT has_purhcase, NULL AS has_review, NULL AS has_favorite, COUNT(1) AS users
      FROM user_action_flag
      GROUP BY has_purhcase

    -- 액션과 관계 없이 모든 사용자 집계
    UNION ALL
      SELECT NULL AS has_purhcase, NULL AS has_review, NULL AS has_favorite, COUNT(1) AS users
      FROM user_action_flag
  )

  SELECT *
  FROM action_venn_diagram
  ORDER BY
    has_purchase, has_review, has_favorite
  ;
  ```

  - 유사적으로 NULL을 포함한 레코드를 추가해서 CUBE 구문과 같은 결과를 얻는 쿼리

  ```SQL
  WITH
  user_action_flag AS (
    ...
  )
  , action_venn_diagram AS (
    SELECT
      mod_has_purchase AS has_purchase
      , mod_has_review AS has_review
      , mod_has_favorite AS has_favorite
      , COUNT(1) AS users
    FROM
      user_action_flag
      -- 각 컬럼에 NULL을 포함하는 레코드 유사적으로 추가
      -- BigQuery, CROSS JOIN unnest 함수 사용
      CROSS JOIN unnest(array[has_purchase, NULL]) AS mod_has_purhcase
      , CROSS JOIN unnest(array[has_review, NULL]) AS mod_has_review
      , CROSS JOIN unnest(array[has_favorite, NULL]) AS mod_has_favorite

      -- Hive, SparkSQL의 경우 LATERAL VIEW와 explode 함수 사용
      LATERAL VIEW explode(array(has_purchase, NULL)) e1 AS mod_has_purchase
      , LATERAL VIEW explode(array(has_review, NULL)) e2 AS mod_has_review
      , LATERAL VIEW explode(array(has_favorite, NULL)) e3 AS mod_has_favorite
    GROUP BY
      mod_has_purchase, mod_has_review, mod_has_favorite
  )
  SELECT *
  FROM action_venn_diagram
  ORDER BY
    has_purchase, has_review, has_favorite
  ;
  ```

  - 벤 다이어그램을 만들기 위해 데이터를 가공하는 쿼리
    - any / any / any 는 모든 사용자를 뜻함
    -

  ```SQL
  WITH
  user_action_flag AS (
    ...
  )
  , action_venn_diagram AS (
    ...
  )
  SELECT
    -- 0,1 플래그를 문자열로 가공
    CASE has_purchase
      WHEN 1 THEN 'purchase' WHEN 0 THEN 'not purhcase' ELSE 'any'
    END AS has_purchase
    , CASE has_review
        WHEN 1 THEN 'review' WHEN 0 THEN 'not review' ELSE 'any'
      END AS has_review
    , CASE has_favorite
        WHEN 1 THEN 'favorite' WHEN 0 THEN 'not favorite' ELSE 'any'
      END AS has_favorite
    , users
      -- 전체 사용자 수를 기반으로 비율 구하기
    , 100.0 * users
        / NULLIF(
          -- 모든 액션이 NULL인 사용자 수 = 전체 사용자 수,
          -- 해당 레코드의 사용자 수를 Windows 함수로 구하기
          SUM(CASE WHEN has_purhcase IS NULL
                    AND has_review   IS NULL
                    AND has_favorite IS NULL
              THEN users ELSE 0 END) OVER()
          , 0)
        AS ratio
    FROM
      action_venn_diagram
    ORDER BY
      has_purchase, has_review, has_favorite
  ;
  ```

### 11-6 Decile 분석을 사용해 사용자를 10단계 그룹으로 나누기

- 데이터를 10단계로 분할해서 중요도를 파악
- 사용자의 구매 금액에 따라 순위를 구분하고 중요도를 파악하는 리포트
  1. 사용자를 구매 금액이 많은 순으로 정렬
  2. 정렬된 사용자 상위부터 10%씩 Decile 1 부터 10까지의 그룹을 할당
  3. 각 그룹의 구매 금액 합계를 집계
  4. 전체 구매 금액에 대해 각 Decile의 구매 금액 비율(구성비)를 계산
  5. 상위에서 누적으로 어느 정도의 비율을 차지하는지 구성비뉴계를 집계
  - 구매액이 많은 순서로 사용자 그룹을 10등분하는 쿼리
    - 같은 수로 데이터 그룹을 만들 때는 `NTILE` 윈도 함수 사용

  ```SQL
  WITH
  user_purchase_amount AS (
    SELECT
      user_id
      , SUM(amount) AS purchase_amount
    FROM
      action_log
    WHERE
      action = 'purchase'
    GROUP BY
      user_id
  )
  , users_with_decile AS (
    SELECT
      user_id
      , purchase_amount
      , ntile(10) OVER (ORDER BY purchase_amount DESC) AS decile
    FROM
      user_purchase_amount
  )
  SELECT *
  FROM users_with_decile
  ```

  - 10분할한 Decile들을 집계하는 쿼리

  ```SQL
  WITH
  user_purchase_amount AS (
    ...
  )
  , users_with_decile AS (
    ...
  )
  , decile_with_purhcase_amount AS (
    SELECT
      decile
      , SUM(purchase_amount) AS amount
      , AVG(purchase_amount) AS avg_amount
      , SUM(SUM(purchase_amount)) OVER (ORDER BY decile) AS cumulative_amount
      , SUM(SUM(purchase_amount)) OVER () AS total_amount
    FROM
      users_with_decile
    GROUP BY
      decile
  )
  SELECT *
  FROM
    decile_with_purchase_amount
  ;
  ```

  - 구매액이 많은 Decile 순서로 구성비와 구성비 누계를 계산하기

  ```SQL
  WITH user_purchase_amount AS (
    ...
  )
  , users_with_decile AS (
    ...
  )
  , decile_with_purchase_amount AS (
    ...
  )
  SELECT
    decile
    , amount
    , avg_amount
    , 100.0 * amount / total_amount AS total_ratio
    , 100.0 * cumulative_amount / total_amount AS cumulative_ratio
  FROM
    decile_with_purchase_amount;
  ```

- Decile 그룹에 따른 속성들을 더 수집해서 활용할 수 있음

### 11-7 RFM 분석으로 사용자를 3가지 관점의 그룹으로 나누기

- Decile은 검색 기간에 따른 문제가 있음 ex) 휴면 고객
- RFM은 보다 자세하게 사용자를 그룹으로 나누는 방법
- RFM 분석의 3가지 지표 집계하기
  - Recency 최근 구매일
  - Frequency 구매 횟수
  - Monetary 구매 금액 합계
  - 사용자별로 RFM을 집계하는 쿼리

  ```SQL
  WITH
  purchase_log AS (
    SELECT
      user_id
      , amount
      -- timestamp를 기반으로 날짜 추출
      -- PostgreSQL, Hive, Redshift, SparkSQL, substring 날짜 추출
      , substring(stamp, 1, 10) AS dt
      -- PostgreSQL, Hive, BigQuery, SparkSQL, substr 사용
      , substr(stamp, 1, 10) AS dt
    FROM
      action_log
    WHERE
      action = 'purchase'
  )
  , user_rfm AS (
    SELECT
      user_id
      , MAX(dt) AS recent_date
      -- PostgreSQL, Redshift, 날짜 형식간 빼기 연산 가능
      , CURRENT_DATE - MAX(dt::date) AS recency
      -- BigQuery, date_diff 함수
      , date_diff(CURRENT_DATE, date(timestamp(MAX(dt))), day) AS recency
      -- Hive, SparkSQL, datediff 함수
      , datediff(CURRENT_DATE(), to_date(MAX(dt))) AS recency
      , COUNT(dt) AS frequency
      , SUM(amount) AS mometary
    FROM
      purchase_log
    GROUP BY
      user_id
  )
  SELECT *
  FROM
    user_rfm
  ```

- RFM 랭크 정의하기
  - 3개의 지표를 5개 그룹으로 125(5 x 5 x 5)개의 그룹으로 사용자를 나눠 파악할 수 있음
  - 단계 정의
  >**랭크**|**R: 최근 구매일**|**F: 누적 구매 횟수**|**M: 누계 구매 금액**
  >-----|-----|-----|-----
  >5|14일 이내|20회 이상|300만원 이상
  >4|28일 이내|10회 이상|100만원 이상
  >3|60일 이내|5회 이상|30만원 이상
  >2|90일 이내|2회 이상|5만원 이상
  >1|91일 이상|1회|5만원 미만
  - 사용자들의 RFM 랭크를 계산하는 쿼리

  ```SQL
  WITH
  user_rfm AS (
    ...
  )
  , user_rfm_rank AS (
    SELECT
      user_id
      , recent_date
      , recency
      , frequency
      , monetary
      , CASE
          WHEN recency < 14 THEN 5
          WHEN recency < 28 THEN 4
          WHEN recency < 60 THEN 3
          WHEN recency < 90 THEN 2
        ELSE 1
        END AS r
      , CASE
          WHEN 20 <= frequency THEN 5
          WHEN 10 <= frequency THEN 4
          WHEN 5 <= frequency THEN 3
          WHEN 2 <= frequency THEN 2
          WHEN 1 <= frequency THEN 1
        END AS f
      , CASE
          WHEN 300000 <= monetary THEN 5
          WHEN 100000 <= monetary THEN 4
          WHEN 30000 <= monetary THEN 3
          WHEN 5000 <= monetary THEN 2
        ELSE 1
        END AS m
  )
  SELECT *
  FROM
    user_rfm_rank
  ;
  ```

  - 각 그룹의 사람 수를 확인하는 쿼리

  ```SQL
  WITH
  user_rfm AS (
    ...
  )
  , user_rfm_rank AS (
    ...
  )
  , mst_rfm_index AS (
    -- 1부터 5까지의 숫자를 가지는 테이블
    -- PostgreSQL, generate_series 함수 등으로 대체 가능
    SELECT 1 AS rfm_index
    UNION ALL SELECT 2 AS rfm_index
    UNION ALL SELECT 3 AS rfm_index
    UNION ALL SELECT 4 AS rfm_index
    UNION ALL SELECT 5 AS rfm_index
  )
  , rfm_flag AS (
    SELECT
      m.rfm_index
      , CASE WHEN m.rfm_index = r.r THEN 1 ELSE 0 END AS r_flag
      , CASE WHEN m.rfm_index = r.f THEN 1 ELSE 0 END AS f_flag
      , CASE WHEN m.rfm_index = r.m THEN 1 ELSE 0 END AS m_flag
    FROM
      mst_rfm_index AS m
    CROSS JOIN
      user_rfm_rank AS r
  )
  SELECT
    rfm_index
    , SUM(r_flag) AS r
    , SUM(f_flag) AS f
    , SUM(m_flag) AS m
  FROM
    rfm_flag
  GROUP BY
    rfm_index
  ORDER BY
    rfm_index DESC
  ```

- 1차원으로 사용자 인식하기
  - 125개의 그룹은 관리하기 힘드므로 적게 그룹을 나누어 관리하기
  - 1차원으로 구분하여 13개의 그룹으로 나누어 관리
  - 통합 랭크를 계산하는 쿼리(R+F+M)

  ```SQL
  WITH
  user_rfm AS (
    ...
  )
  , user_rfm_rank AS (
    ...
  )
  SELECT
    r + f + m AS total_rank
    , r, f, m
    , COUNT(user_id)
  FROM
    user_rfm_rank
  GROUP BY
    r, f, m
  ORDER BY
    total_rank DESC, r DESC, F DESC, m DESC;
  ```

  - 종합 랭크별로 사용자 수를 집계하는 쿼리

  ```SQL
  WITH
  user_rfm AS (
    ...
  )
  , user_rfm_rank AS (
    ...
  )
  SELECT
    r + f + m AS total_rank
    , COUNT(user_id)
  FROM
    user_rfm_rank
  GROUP BY
    -- PostgreSQL, Redshift, BigQuery
    -- SELECT 구문에서 정의한 별칭을 GROUP BY 구문에 지정 가능
    total_rank
    -- PostgreSQL, Hive, Redshift, SparkSQL의 경우
    -- SELECT 구문에서 별칭을 지정하기 전의 식을 GROUP BY 구문에 지정할 수 있음
    -- r + f + m
  ORDER BY
    total_rank DESC;
  ```

- 2차원으로 사용자 인식하기
  - R과 F를 사용해 2차원으로 사용해 2차원 사용자 층의 사용자 수를 집계하는 쿼리
  - **R**, **F**를 사용해 집계하는 방법
  >R/F|**20회 이상**|**10회 이상**|**5회 이상**|**2회 이상**|**1회**
  >:-----:|:-----:|:-----:|:-----:|:-----:|:-----:
  >14일 이내|단골| | |신규|신규
  >28일 이내| |안정|안정| |
  >60일 이내|단골 이탈 전조|단골 이탈 전조| |신규 이탈 전조|신규 이탈 전조
  >90일 이내| | | |신규 이탈|신규 이탈
  >91일 이상|단골/안정 이탈|단골/안정 이탈|단골/안정 이탈|신규 이탈|신규 이탈

  ```SQL
  WITH
  user_rfm AS (
    ...
  )
  , user_rfm_rank AS (
    ...
  )
  SELECT
    CONCAT('r_', r) AS r_rank
    -- BigQuery의 경우 CONCAT 함수의 매개 변수를 string으로 변환해야 함
    , CONCAT('r_', CAST(r as string)) AS r_Rank
    , CONCAT(CASE WHEN f = 5 THEN 1 END) AS f_5
    , CONCAT(CASE WHEN f = 4 THEN 1 END) AS f_4
    , CONCAT(CASE WHEN f = 3 THEN 1 END) AS f_3
    , CONCAT(CASE WHEN f = 2 THEN 1 END) AS f_2
    , CONCAT(CASE WHEN f = 1 THEN 1 END) AS f_1
  FROM
    user_rfm_rank
  GROUP BY
    r
  ORDER BY
    r_rank DESC;
  ```

- RFM 분석의 각 지표에 따라 사용자의 속성을 정의하고 1, 2, 3 차원으로 표현하는 방법을 살펴봄
- 서비스 개선, 검토, 사용자에 따른 메일 최적화 등 다영한 용도로 활용할 수 있음

---

## 12강 시계열에 따른 사용자 전체의 상태 변화 찾기

- 사용자는 시간이 지남에 따라 사용자의 상태(단골, 휴면 등)이 변화함

### 12-1 등록 수의 추이와 경향 보기

- 날짜별 등록 수의 추이를 집계하는 쿼리

  ```SQL
  SELECT
    register_Date
    , COUNT(DISTINCT user_id) AS register_count
  FROM
    mst_users
  GROUP BY
    register_date
  ORDER BY
    register_date
  ;
  ```

- 매달 등록 수와 전월비를 계산하는 쿼리

  ```SQL
  WITH
  mst_users_with_year_month AS (
    SELECT
      *
      -- PostgreSQL, Hive, Redshift, SparkSQL, substring으로 연~월 부분 추출
      , substring(register_Date, 1, 7) AS year_month
      -- PostgreSQL, Hive, BigQuery, SparkSQL, substr 사용
      , substr(register_date, 1, ,7) AS year_month
    FROM
      mst_users
  )
  SELECT
    year_month
    , COUNT(DISTINCT user_id) AS register_count

    , LAG(COUNT(DISTINCT user_id)) OVER(ORDER BY year_month)
    -- SparkSQL의 경우 다음과 같이 사용
    , LAG(COUNT(DISTINCT user_id))
      OVER(ORDER BY year_month ROWS BETWEEN 1 PRECEDING AND 1 PRECEDING)
    
    AS last_month_count
    
    , 1.0
      * COUNT(DISTINCT user_id)
      / LAG(COUNT(DISTINCT user_id)) OVER(ORDER BY year_month)
    -- sparkSQL의 경우 다음과 같이 사용
      / LAG(COUNT(DISTINCT user_id))
        OVER(ORDER BY year_month ROWS BETWEEN PRECEDING AND 1 PRECEDING)
    AS month_over_month_ratio
  FROM
    mst_users_with_year_month
  GROUP BY
    year_month
  ;
  ```

- 디바이스들의 등록 수를 집계하는 쿼리

  ```SQL
  WITH
  mst_users_with_year_month AS (
    ...
  )
  SELECT
    year_month
    , COUNT(DISTINCT user_id) AS register_count
    , COUNT(DISTINCT CASE WHEN register_device='pc' THEN user_id END) AS register_pc
    , COUNT(DISTINCT CASE WHEN register_device='sp' THEN user_id END) AS regsiter_sp
    , COUNT(DISTINCT CASE WHEN register_device='app' THEN user_id END) AS register_app
  FROM
    mst_users_with_year_month
  GROUP BY
    year_month
  ;
  ```

### 12-2 지속률과 정착률 산출하기

- 지속률: 등록일 기준으로 이후 지정일 동안 사용자가 서비스를 얼마나 이용했는지 나타내는 지표
  - 매일 이용하지 않더라도 판정 날짜에 사용하면 지속자로 취급
  - 사용자가 매일 사용했으면 하는 서비스 ex) 뉴스, 사이트, SNS, 게임 등
- 정착률: 등록일 기준으로 이후 지정한 7일 동안 사용자가 서비스를 사용했는지 나타내는 지표
  - 7일동안 한 번이라도 서비스를 사용했다면 정착자로 다룸
  - 사용자가 어떤 목적이 생겼을 때 사용했으면 하는 서비스 ex) 리뷰 사이트, QnA 사이트, 투고 사이트 등
- 지속률과 관계있는 리포트
  - 날짜별 n일 지속률 추이
  - '로그 최근 일자'와 '사용자별 등록일의 다음날'을 계산하는 쿼리
    - 최신 일자를 넘으면 NULL로 산정

  ```SQL
  WITH
  action_log_with_mst_users As (
    SELECT
      u.user_id
      , u.register_date
      -- 액션 날짜와 로그 전체의 최신 날짜를 날짜 자료형으로 변환
      , CAST(a.stamp AS date) AS action_date
      , MAX(CAST(a.stamp AS date)) OVER() AS latest_date
      -- BigQuery, 한번 타임 스탬프 자료형으로 변환 후 날짜 자료형으로 변환
      , date(timestamp(a.stamp)) AS action_date
      , MAX(date(timestamp(a.stamp))) OVER() AS latest_date

      -- 등록일의 다음 날짜 계산
      -- PostgreSQL
      , CAST(u.register_date::date + '1 day'::interval AS date)
      -- Redishift
      , dateadd(day, 1, u.register_date::date)
      -- BigQuery
      , date_add(CAST(u.register_date AS date), interval 1 day)
      -- Hive, SparkSQL
      , date_add(CAST(u.register_date AS date), 1)
      AS next_day_1
    FROM
      mst_users AS u
      LEFT OUTER JOIN action_log AS a ON u.user_id = a.user_id
  )
  SELECT *
  FROM
    action_log_with_mst_users
  ORDER BY
    register_date
  ```

  - 사용자의 액션 플래그를 계산하는 쿼리
    - 지정한 날의 다음날에 액션을 했는지 0과 1로 플래그로 표현

  ```SQL
  WITH
  action_log_with_mst_users AS (
    ...
  )
  , user_action_flag AS (
    SELECT
      user_id
      , register_date
      -- 등록일 다음날에 액션 여부를 플래그로 표현
      , SIGN(
        -- 사용자별 등록일 다음날에 한 액션의 합계 구하기
        SUM(
          -- 등록일 다음날이 로그 최신 날짜 이전인지 확인
          CASE WHEN next_day_1 <= latest_date THEN
            -- 등록일 다음날의 날짜에 액션을 했다면 1, 아니면 0 지정
            CASE WHEN next_day_1 = action_date THEN 1 ELSE 0 END
          END
        )
      ) AS next_1_day_action
    FROM
      action_log_with_mst_users
    GROUP BY
      user_id, register_date
  )
  SELECT *
  FROM
    user_action_flag
  ORDER BY
    register_date, user_id;
  ```

  - 다음날 지속률을 계산하는 쿼리
    - n번째 날짜의 지속률을 구할려면 매번 action_log_with_mst_users에서 날짜를 바꿔줘야 함

  ```SQL
  WITH
  action_log_with_mst_users AS (
    ...
  )
  , user_action_flag As (
    ...
  )
  SELECT
    register_Date
    , AVG(100.0 * next_1_day_action) AS repeat_rate_1_day
  FROM
    user_action_flag
  GROUP BY
    register_date
  ORDER BY
    register_date
  ;
  ```

  - 지속률 지표를 관리하는 마스터 테이블을 작성하는 쿼리

  ```SQL
  WITH
  repeat_interval(index_name, interval_date) AS (
    -- PostgreSQL의 경우 VALUES 구문으로 일시 테이블 생성 가능
    -- Hive, Redshift, BigQuery, SparkSQL의 경우 UNION ALL로 대체 가능
    -- 8.5 참조
    VALUES
      ('01 day repeat', 1)
      , ('02 day repeat', 2)
      , ('03 day repeat', 3)
      , ('04 day repeat', 4)
      , ('05 day repeat', 5)
      , ('06 day repeat', 6)
      , ('07 day repeat', 7)
  )
  SELECT *
  FROM repeat_interval
  ORDER BY index_name
  ;
  ```

  - 지속률을 세로기반으로 집계하는 쿼리
    - 판정 기간의 로그가 존재하지 않는 경우 NULL로 출력

  ```SQL
  WITH
  repeat_interval AS (
    ...
  )
  , action_log_with_index_date AS (
    SELECT
      u.user_id
      , u.register_date
      , CAST(a.stamp AS date) AS action_date
      , MAX(CAST(a.stamp AS date)) OVER() AS latest_date
      -- BigQuery
      , date(timestamp(a.stamp)) AS action_date
      , MAX(date(timestamp(a.stamp))) OVER() latest_date

      -- 등록일로부터 n일 후의 날짜
      , r.index_name

      -- PostgreSQL
      . CAST(CAST(u.register_date As date) + interval '1 day' * r.interval_date AS date)
      -- Redshift
      , dateadd(day, r.interval_date, u.register_date::date)
      -- BigQuery
      , date_add(CAST(u.register_date As date), interval r.interval_date day)
      -- Hive, SparkSQL
      , date_add(CAST(u.register_Date AS date), r.interval_date)
      AS index_date
    FROM
      mst_users AS u
      LEFT OUTER JOIN action_log AS a ON u.user_id = a.user_id
      CROSS JOIN
        repeat_interval AS r
  )
  , user_action_flag AS (
    SELECT
      user_id
      , register_date
      , index_name
      -- 등록일로부터 n일후 액션 여부 플래그 표현
      , SIGN(
        SUM(
          CASE WHEN index_date <= latest_date THEN
            CASE WHEN index_date = action_date THEN 1 ELSE 0 END
          END
        )
      ) AS index_date_action
    FROM
      action_log_with_index_date
    GROUP BY
      user_id, register_date, index_name, index_date
  )
  SELECT
    register_date
    , index_name
    , AVG(100.0 * index_date_action) AS repeat_rate
  FROM
    user_action_flag
  GROUP BY
    register_date, index_name
  ORDER BY
    register_date, index_name
  ```

- 정착률 관련 리포트
  - 매일의 n일 정착률 추이
  - 지속률에서 썻던 interval_date를 begin과 end로 확장해야함
  - 정착률 지표를 관리하는 마스터 테이블을 작성하는 쿼리

  ```SQL
  WITH
  repeat_interval(index_name, interval_begin_date, interval_end_date) AS (
    -- PostgreSQL, VALUES 구문으로 일시 테이블 생성
    -- Hive,Redshift, BigQuery, SparkSQL, UNION ALL 등으로 대체 가능
    -- 8-5
    VALUES
      ('07 day retention', 1, 7)
      , ('14 day retention', 8, 14)
      , ('21 day retention', 15, 21)
      , ('28 day retention', 22, 28)
  )
  SELECT *
  FROM repeat_interval
  ORDER BY index_name
  ;
  ```

  - 정착률을 계산하는 쿼리

  ```SQL
  WITH
  repeat_interval AS (
    ...
  )
  , action_log_with_index_date AS (
    SELECT
      u.user_id
      , u.register_date
      -- 액션의 날짜와 로그 전체의 최신 날짜를 날짜 자료형으로 변환
      , CAST(a.stamp AS date) AS action_date
      , MAX(CAST(a.stamp AS date)) OVER() AS latest_date
      -- BigQuery, 한 번 타임 스탬프 자료형으로 변환하고, 날짜 자료형으로 변환
      , date(timestamp(a.stamp)) AS action_date
      , MAX(date(timestamp(a.stamp))) OVER() AS latest_date
      , r.index_name

      -- 지표의 대상 기간 시작일 / 종료일 계산
      -- PostgreSQL
      , CAST(u.register_date::date + '1 day'::interval * r.interval_begin_date AS date) AS index_begin_date
      , CAST(u.register_date::date + '1 day'::interval * r.interval_end_date AS date) AS index_end_date
      -- Redshift
      , dateadd(day, r.interval_begin_date, u.register_date::date) AS index_begin_date
      , dateadd(day, r.interval_end_date, u.register_date::date) AS index_end_date
      -- BigQuery
      , date_add(CAST(u.register_date AS date), interval r.interval_begin_date day) AS index_begin_date
      , date_add(CAST(u.register_date AS date, interval r.index_end_date day)) AS index_end_date
      -- Hive, SparkSQL
      , date_add(CAST(u.register_date AS date), r.interval_begin_date) AS index_begin_date
      , date_add(CAST(u.register_date AS date), r.interval_end_date) AS index_end_date)
    FROM
      mst_users AS u
      LEFT OUTER JOIN action_log AS a 
      ON u.user_id = a.user_id
      CROSS JOIN
        repeat_interval AS r 
  )
  , user_action_flag As (
    SELECT
      user_id
      , register_date
      , index_name
      -- 지표의 대상 기간에 액션을 했는지 플래그 판단
      , SIGN(
        -- 사용자별 대상 기간에 한 액션의 합 구하기
          SUM(
            -- 대상 기간의 종료일이, 로그 최신날짜 이전인지 판단
            CASE WHEN index_end_Date <= latest_date THEN
              -- 지표의 대상 기간안에 액션을 했다면 1 아니면 0
              CASE WHEN action_date BETWEEN index_begin_date AND index_end_date
                THEN 1 ELSE 0
              END
            END
          )
      ) AS index_date_action
    FROM
      action_log_with_index_date
    GROUP BY
      user_id, register_date, index_name, index_begin_date, index_end_date
  )
  SELECT
    register_date
    , index_name
    , AVG(100.0 * index_date_action) AS index_rate
  FROM
    user_action_flag
  GROUP BY
    register_date, index_name
  ORDER BY
    register_date, index_name
  ```

  - 지속률 지표를 관리하는 마스터 테이블을 정착률 형식으로 수정한 쿼리

  ```SQL
  WITH
  repeat_interval(index_name, interval_begin_date, interval_end_date) AS (
    -- PostgreSQL, VALUES로 일시 테이블 생성 가능
    -- Hive, Redshift, BigQuery, SparkSQL의 경우 UNION ALL 대체 가능
    VALUES
      ('01 day repeat'      , 1, 1)
      , ('02 day repeat'    , 2, 2)
      , ('03 day repeat'    , 3, 3)
      , ('04 day repeat'    , 4, 4)
      , ('05 day repeat'    , 5, 5)
      , ('06 day repeat'    , 6, 6)
      , ('07 day repeat'    , 7, 7)
      , ('07 day retention' , 1, 7)
      , ('14 day retention' , 8, 14)
      , ('21 day retention' , 15, 21)
      , ('28 day retention' , 22, 28)
  )
  SELECT *
  FROM repeat_interval
  ORDER BY index_name
  ;
  ```

  - n일 지속률들을 집계하는 쿼리

  ```SQL
  WITH
  repeat_interval AS (
    ...
  )
  , action_log_with_index_date AS (
    ...
  )
  , user_action_flag AS (
    ...
  )
  SELECT
    index_name
    , AVG(100.0 * index_date_action) AS repeat_rate
  FROM
    user_action_flag
  GROUP BY
    index_name
  ORDER BY
    index_name
  ```

- 지속률과 정착률은 모두 등록일 기준으로 N일 후의 행동을 집계하는 것으로서, 단기간에 결과를 보고 대책을 세울 수 있는 지표로 활용

### 12-3 지속과 정착에 영향을 주는 액션 집계하기

- 지속률이나 정착률을 올리기 위한 대책이 필요함
- 액션에 따른 사용자, 비사용자의 1일 지속률을 비교해야 함
  - 모든 사용자와 액션의 조합을 도출하는 쿼리

  ```SQL
  WITH
  repeat_interval(index_name, interval_begin_Date, interval_end_date) AS (
    -- PostgreSQL의 경우 VALUES로 일시 테이블 생성 가능
    -- Hive, Redshift, Bigquery, SparkSQL의 경우 SELECT로 대체 가능
    -- 4.5 참고
    VALUES ('01 day repeat', 1, 1)
  )
  , action_log_with_index_Date AS (
    ...
  )
  , user_action_flag AS (
    ...
  )
  , mst_actions AS (
    SELECT 'view' AS action
    UNION ALL SELECT 'comment' AS action
    UNION ALL SELECT 'follow' AS action
  )
  , mst_user_actions AS (
    SELECT
      u.user_id
      , u.register_date
      , a.action
    FROM
      mst_users AS u
      CROSS JOIN
        mst_actions AS a
  )
  SELECT *
  FROM mst_user_actions
  ORDER BY user_id, action
  ;
  ```

  - 사용자의 액션 로그를 0, 1의 플래그로 표현하는 쿼리
    - register_date에 action을 취했고 1일 지속이 됏다면 index_date_action에 1로 표시됨
    - do_action이 0이면 비사용자

  ```SQL
  WITH
  repeat_interval AS (
    ...
  )
  , action_log_with_index_date AS (
    ...
  )
  , user_action_flag AS (
    ...
  )
  , mst_actions AS (
    ...
  )
  , mst_user_actions AS (
    ...
  )
  , register_action_flag AS (
    SELECT DISTINCT
      m.user_id
      , m.register_date
      , m.action
      , CASE
          WHEN a.action IS NOT NULL THEN 1
          ELSE 0
        END AS do_action
      , index_name
      , index_date_action
    FROM
      mst_user_actions AS m
      LEFT JOIN
        action_log AS a
        ON m.user_id = a.user_id
        AND CAST(m.register_date AS date) = CAST(a.stamp AS date)
        -- BigQuery, Timestamp 자료형으로 변환한 뒤, 날짜 자료형으로 변환
        AND CAST(m.regsiter_date AS date) = date(timestamp(a.stamp))
        AND m.action = a.action
      LEFT JOIN
        user_action_flag AS f
        ON m.user_id = f.user_id
    WHERE
      f.index_date_action IS NOT NULL
  )
  SELECT *
  FROM register_action_flag
  ORDER BY user_id, index_name, action
  ;
  ```

  - 액션에 따른 지속률과 정착률을 집계하는 쿼리

  ```SQL
  WITH
  repeat_interval AS (
    ...
  )
  , action_log_with_index_date AS (
    ...
  )
  , user_action_flag AS (
    ...
  )
  , mst_actions AS (
    ...
  )
  , mst_user_actions AS (
    ...
  )
  , register_action_flag AS (
    SELECT DISTINCT
      m.user_id
      , m.register_date
      , m.action
      , CASE
          WHEN a.action IS NOT NULL THEN 1
          ELSE 0
        END AS do_action
      , index_name
      , index_date_action
    FROM
      mst_user_actions AS m
      LEFT JOIN
        action_log AS a
        ON m.user_id = a.user_id
        AND CAST(m.register_date AS date) = CAST(a.stamp AS date)
        -- BigQuery, Timestamp 자료형으로 변환한 뒤, 날짜 자료형으로 변환
        AND CAST(m.regsiter_date AS date) = date(timestamp(a.stamp))
        AND m.action = a.action
      LEFT JOIN
        user_action_flag AS f
        ON m.user_id = f.user_id
    WHERE
      f.index_date_action IS NOT NULL
  )
  SELECT *
  FROM register_action_flag
  ORDER BY user_id, index_name, action
  ;
  ```

### 12-4 액션 수에 따른 정착률 집계하기

- 7일 동안에 실행한 액션 수에 따라 14일 정착률이 어떻게 변화하는지 보기
  - 액션의 계급 마스터와 사용자 액션 플래그의 조합을 산출하는 쿼리

  ```SQL
  WITH
  repeat_interval(index_name, interval_begin_date, interval_end_date) AS (
    -- PostgreSQL의 경우 VALUES로 일시 테이블 생성 가능
    -- Hive, Redshift, BigQuery, SparkSQLL의 경우 SELECT로 ㄷ ㅐ체 가능
    -- 8강 5절
    VALUES ('14 day retention', 8, 14)
  )
  , action_log_with_index_date AS (
    -- 12-10 코드
  )
  , user_action_flag AS (
    -- 12-10 코드
  )
  , mst_action_bucket(action, min_count, max_count) AS (
    -- 액션 단계 마스터
    VALUES
      ('comment', 0, 0)
      , ('comment', 1, 5)
      , ('comment', 6, 10)
      , ('comment', 11, 9999) -- 최댓값으로 간단하게 9999 입력
      , ('follow', 0, 0)
      , ('follow', 1, 5)
      , ('follow', 6, 10)
      , ('follow', 11, 9999) --최댓값으로 간단하게 9999 입력
  )
  , mst_user_action_bucket AS (
    -- 사용자 마스터와 액션 단계 마스터 조합
    SELECT
      u.user_id
      , u.register_date
      , a.action
      , a.min_count
      , a.max_count
    FROM
      mst_users AS u
      CROSS JOIN
        mst_action_bucket AS a
  )
  SELECT *
  FROM
    mst_user_action_bucket
  ORDER BY
    user_id, action, min_count
  ```

  - 등록 후 7일 동안의 액션 수를 집계하는 쿼리

  ```SQL
  WITH
  repeat_interval AS (
    ...
  )
  , action_log_with_index_date AS (
    ...
  )
  , user_action_flag AS (
    ...
  )
  , mst_action_bucket AS (
    ...
  )
  , mst_user_action_bucket AS (
    ...
  )
  , register_action_flag As (
    -- 등록일에서 7일 후까지 액션수를 세고,
    -- 액션 단계와 14일 정착 달성 플래그 계산
    SELECT
      m.user_id
      , m.action
      , m.min_count
      , m.max_count
      , COUNT(a.action) AS action_count
      , CASE
          WHEN COUNT(a.action) BETWEEN m.min_count AND m.max_count THEN 1
          ELSE 0
        END As achieve
      , index_name
      , index_date_action
    FROM
      mst_user_action_bucket AS m
      LEFT JOIN
        action_log AS a
        ON m.user_id = a.user_id
        -- 등록일 당일부터 7일 후까지의 액션 로그 결합
        -- PostgreSQL, Redshift의 경우
        AND CAST(a.stamp AS date)
          BETWEEN CAST(m.register_date AS date)
          AND CAST(m.register_date AS date) + interval '7 days'
        -- BigQuery의 경우
        AND date(timestamp(a.stamp))
          BETWEEN CAST(m.register_date AS date)
          AND date_add(CAST(m.register_date AS date), interval 7day)
        -- SparkSQL
        AND CAST(a.stamp AS date)
          BETWEEN CAST(m.register_date AS date)
          AND date_add(CAST(m.register_date AS date), 7)
        -- Hive의 경우 JOIN 구문에 BETWEEN을 사용할 수 없으므로, WHERE을 사용하여야 함
        AND m.action = a.action
      LEFT JOIN
        user_action_flag AS f
        ON m.user_id = f.user_id
      WHERE
        f.index_date_action IS NOT NULL
      GROUP BY
        m.user_id
        , m.action
        , m.min_count
        , m.max_count
        , f.index_name
        , f.index_date_action
  )
  SELECT *
  FROM
    register_action_flag
  ORDER BY
    user_id, action, min_count
  ;
  ```

  - 등록 후 7일 동안의 액션 횟수별로 14일 정착률을 집계하는 쿼리

  ```SQL
  WITH
  repeat_interval AS (
    ...
  )
  , action_log_with_index_date AS (
    ...
  )
  , user_action_flag AS (
    ...
  )
  , mst_action_bucket AS (
    ...
  )
  , mst_user_action_bucket AS (
    ...
  )
  , register_action_flag AS (
    ...
  )
  SELECT
    action
    -- PostgreSQL, Redshift, 문자열 연결
    , min_count || '~' || max_count AS count_range
    -- BigQuery
    , CONCAT(CAST(min_count AS string), '~', CAST(max_count AS string)) AS count_range
    , SUM(CASE achieve WHEN 1 THEN 1 ELSE 0 END) AS archieve
    , index_name
    , AVG(CACSE achieve WHEN 1 THEN 100*0 * index_date_action END) AS achieve_index_date
  FROM
    register_action_flag
  GROUP BY
    index_name, action, min_count, max_count
  GROUP BY
    index_name, action, min_count;
  ```

### 12-5 사용 일수에 따른 정착률 집계하기

- 7일 정착기간 동안 사용자가 며칠 사용했는지가 이후 정착률에 어떠한 영향을 주는지
- 리포트 만드는 법
  - 등록일 다음날부터 7일 동안의 사용 일수를 집계
  - 사용 일수별로 집계한 사용자 수의 구성비와 구성비누계를 계산
  - 사용 일수별로 집계한 사용자 수를 분모로 두고, 28일 정착률을 집계한 뒤 그 비율을 계산
- 리포트로 알 수 있는 것
  - 70% 사용자가 7일 판정 기간에 1일~4일 밖에 사용하지 않음
  - 5일 동안 사용자의 28일 정착률은 45%
  - 5일의 이탈자가 많으면 1~6일 보상 6일의 빅이벤트 보상 같은걸로 정착률을 늘릴 수 있음
- 리포트 SQL
  - 등록된 다음날부터 7일 동안의 사용 일수를 집계
  - 사용일수별로 집계한 사용자 수의 구성비와 구성비누계를 계산
  - 사용 일수별로 집계한 사용자 수를 분모로 두고 28일 정착률을 집계한 뒤 비율을 계산
  - 등록일 다음날부터 7일 동안의 사용 일수와 28일 정착 플래그를 생성하는 쿼리
    - 등록일 다음날부터 7일 동안의 사용 일수와 해당 사용자의 28일 정착 플래그 생성

  ```SQL
  WITH
  repeat_interval(index_name, interval_begin_date, interval_end_date) AS (
    -- PostgreSQL은 VALUES로 일시 테이블 생성 가능
    -- Hive, Redshift, BigQuery, SparkSQL의 경우 SELECT로 대체 가능
    VALUES ('28 day retention', 22, 28)
  )
  , action_log_with_index_date AS (
    ...
  )
  , user_action_flag AS (
    ...
  )
  , register_action_flag AS (
    SELECT
      m.user_id
      , COUNT(DISTINCT CAST(a.stamp AS date)) AS dt_count
      -- BigQuery의 경우 다음과 같이 사용
      , COUNT(DISTINCT date(timestamp(a.stamp))) AS dt_count
      , index_name
      , index_date_action
    FROM
      mst_users AS m
      LEFT JOIN
        action_log AS a
        ON m.user_id = a.user_id

        -- 등록 다음날부터 7일 이내의 액션 로그 결합하기
        -- PostgreSQL, Redshift의 경우 다음과 같이 사용
        AND CAST(a.stamp AS date)
          BETWEEN CAST(m.register_date AS date) + interval '1 day'
            AND CAST(m.register_date AS date) + interval '8 days'
        
        -- BigQuery
        AND date(timestamp(a.stamp))
          BETWEEN date_add(CAST(m.register_date AS date), interval 1 day)
            AND date_add(CAST(m.register_date AS date), interval 8 day)

        -- SparkSQL
        AND CAST(a.stamp AS date)
          BETWEEN date_add(CAST(m.register_Date AS date), 1)
            AND date_add(CAST(m.register_Date AS date), 8)
        
        -- Hive, JOIN에 BETWEEN을 사용할 수 없으므로, WHERE 사용
      LEFT JOIN
        user_action_flag AS f
        ON m.user_id = f.user_id
    WHERE
      f.index_date_action IS NOT NULL
    GROUP BY
      m.user_id
      , f.index_name
      , f.index_date_action
  )
  SELECT *
  FROM
    register_action_flag;
  ```

  - 사용 일수에 따른 정착율을 집계하는 쿼리

  ```SQL
  WITH
  repeat_interval AS (
    ...
  )
  , action_log_with_index_date AS (
    ...
  )
  , user_action_flag AS (
    ...
  )
  , register_action_flag AS (
    ...
  )
  SELECT
    dt_count AS dates
    , COUNT(user_id) AS users
    , 100.0 * COUNT(user_id) / SUM(COUNT(user_id)) OVER() AS user_ratio
    , 100.0 *
      SUM(COUNT(user_id))
        OVER(ORDER BY index_name, dt_count)
          ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
      / SUM(COUNT(user_id)) OVER() AS cum_ratio
    , SUM(index_date_action) AS achieve_users
    , AVG(100.0 * index_date_action) AS achieve_ratio
  FROM
    register_action_flag
  GROUP BY
    index_name, dt_count
  ORDER BY
    index_name, dt_count;
  ```

  - 사용 일수 말고도 글의 개수나 게임 레벨 등으로 대상을 변경해서 활용도 가능

### 12-6 사용자의 잔존율 집계하기

---

- 서비스를 지속적으로 사용하는 사용자를 파악함으로써 문제발견과 미래 목표를 세울 수 있음
  - 잔존율이 내려가면? 서비스 장벽이 높은지 확인
  - 잔존율이 갑자기 내려가면? 사용 목적을 달성하는데 시간이 너무 짧지 않은지 확인
  - 오래 사용하던 사용자의 잔존율이 내려가면? 서비스 경쟁등으로 지친것인지 확인
- 잔존율을 월 단위로 집계하려면 12개월짜리의 추가 테이블이 필요함
  - 12개월 후까지의 월을 도출하기 위한 보조 테이블을 만드는 쿼리
  
  ```SQL
  WITH
  mst_intervals(interval_month) AS (
    -- 12개월 동안의 순번 만들기(generate_series 등으로 대체 가능)
    -- PostgreSQL의 경우 VALUES 구문으로 일시 테이블 생성
    -- Hive, Redshift, BigQuery, SparkSQL의 경우 SELECT 구문과 UNION ALL로 대체 가능
    VALUES (1), (2), (3), (4), (5), (6), (7), (8), (9), (10), (11), (12)
  )
  SELECT *
  FROM mst_intervals
  ;
  ```

- 월 단위 잔존율 집계
  - 등록 월에서 12개월 후까지의 잔존율을 집계하는 쿼리

  ```SQL
  WITH
  mst_intervals AS (
    ...
  )
  , mst_users_with_index_month AS (
    -- 사용자 마스터에 등록 월부터 12개월 후까지의 월 추가
    SELECT
      u.user_id
      , u.register_date
      -- n개월 후의 날짜, 등록일, 등록 월 n개월 후의 월 계산
      -- PostgreSQL의 경우 다음과 같이 사용
      , CAST(u.register_date::date + i.interval_month * '1 month'::interval AS date) AS index_date
      , substring(u.register_date, 1, 7) AS register_month
      , substring(CAST(
        u.register_date::date + i.interval_month * '1 month'::interval AS text), 1, 7) AS index_month
      
      -- Redshift
      , dateadd(month, i.interval_month, u.register_date::date) AS index_date
      , substring(u.register_date, 1, 7) AS register_month
      , substring(
        CAST(dateadd(month, i.interval_month, u.regsiter_date::date) AS text), 1, 7) AS index_month
      
      -- BigQuery
      , date_add(date(timestamp(u.register_date)), interval i.interval_month month) AS index_date
      , substr(u.register_date, 1, 7) AS register_month
      , substr(CAST(
        date_add(date(timestamp(u.register_date)), interval i.interval_month month) AS string), 1,7) AS index_month
      
      -- Hive, SparkSQL
      , add_month(u.register_date, i.interval_month) AS index_date
      , substring(u.register_date, 1, 7) AS register_month
      , substring(
        CAST(add_months(u.register_date, i.interval_month) AS string), 1, 7) AS index_month
    FROM
      mst_users AS u
      CROSS JOIN
        mst_intervals AS i
  )
  , action_log_in_month AS (
    -- 액션 로그의 날짜에서 월 부분만 추출
    SELECT DISTINCT
      user_id
      , substring(stamp, 1, 7) AS action_month
      -- BigQuery의 경우 substr 사용
      , substr(stamp, 1, 7) AS action_month
    FROM
      action_log
  )
  SELECT
    -- 사용자 마스터와 액션 로그를 결합 후, 월별로 잔존율 집계
    u.register_month
    , u.index_month
    -- action_month가 NULL이 아니라면(액션을 했다면) 사용자 수 집계
    , SUM(CASE WHEN a.action_month IS NOT NULL THEN 1 ELSE 0 END) AS users
    , AVG(CASE WHEN a.action_month IS NOT NULL THEN 100.0 ELSE 0.0 END) AS retention_rate
  FROM
    mst_users_with_index_month AS u
    LEFT JOIN
      action_log_in_month AS a
      ON u.user_id = a.user_id
      AND u.index_month = a.action_month
  GROUP BY
    u.register_month, u.index_month
  ORDER BY
    u.register_month, u.index_month
  ;
  ```
  
  - 사용자 등록과 지속 사용을 파악하기 용이
  - 마케팅이나 캠페인등을 함께 기록하여 수치 변화의 원인을 파악하면 좋음

### 12-7 방문 빈도를 기반으로 사용자 속성을 정의하고 집계하기

---

- 서비스 사용자의 방문 빈도를 월 단위로 파악하고, 방문 빈도에 따라 사용자를 분류하고 내역을 집계
- MAU
  - Monthly Active Users
  - 월에 서비스를 사용한 사용자 수
  - 신규인지 기존인지 몰라 3개로 나누어 분석
    - 신규 사용자
    - 리피트 사용자: 이전 달에도 사용했던 사용자
    - 컴백 사용자: 한동안(2달 전) 사용하지 않았다가 돌아온 사용자
  - 신규, 리피트, 컴백 사용자 수를 집계하는 쿼리
  
  ```SQL
  WITH
  monthly_user_action AS (
    -- 월별 사용자 액션 집약하기
    SELECT DISTINCT
      u.user_id
      -- PostgreSQL
      , substring(u.register_date, 1, 7) AS register_month
      , substring(l.stamp, 1, 7) AS action_month
      , substring(CAST(
        l.stamp::date - interval '1 month' AS text
      ), 1, 7) AS action_month_priv

      -- Redshift
      , substring(u.regsiter_date, 1, 7) AS register_month
      , substring(l.stamp, 1, 7) As action_month
      , substring(
        CAST(dateadd(month, -1, l.stamp::date) AS text), 1, 7
      ) AS action_month_priv

      -- BigQuery
      , substr(u.register_date, 1, 7) AS register_month
      , substr(l.stamp, 1, 7) AS action_month
      , substr(CAST(
        date_sub(date(timestamp(l.stamp)), interval 1 month) AS string
      ), 1, 7) AS action_month_priv

      -- Hive, SparkSQL
      , substring(u.register_date, 1, 7) AS register_month
      , substring(l.stamp, 1, 7) AS action_month
      , substring(
        CAST(add_month(to_date(l.stamp), -1) AS string), 1, 7
      ) AS action_month_priv
    
    FROM
      mst_users AS u
      JOIN
        action_log AS l
        ON u.user_id = l.user_id
  )
  , monthly_user_with_type AS (
    -- 월별 사용자 분류 테이블
    SELECT
      action_month
      , user_id
      , CASE
          -- 등록 월과 액션월이 일치하면 신규 사용자
          WHEN register_month = action_month THEN 'new_user'
          -- 이전 월에 액션이 있다면, 리피트 사용자
          WHEN action_month_priv = 
            LAG(action_month)

            OVER(PARTITION BY user_id ORDER BY action_month)
            -- SparkSQL의 경우 다음과 같이 사용
            OVER(PARTITION BY user_id ORDER BY action_month ROWS BETWEEN 1 PRECEDING AND 1 PRECEDING)
            THEN 'repeat_user'
          -- 이외의 경우에는 컴백 사용자
          ELSE 'come_back_user'
        END AS c
      , action_month_priv
    FROM
      monthly_user_action
  )
  SELECT
    action_month

    -- 특정 달의 MAU
    , COUNT(user_id) AS mau
    -- new_users / repeat_users / com_back_users
    , COUNT(CASE WHEN c = 'new_user' THEN 1 END) AS new_users
    , COUNT(CASE WHEN c = 'repeat_user' THEN 1 END) AS repeat_users
    , COUNT(CASE WHEN c = 'come_back_user' THEN 1 END) AS come_back_users
  FROM
    monthly_user_with_type
  GROUP BY
    action_month
  ORDER BY
    action_month
  ;
  ```

  - 리피트 사용자를 3가지로 분류하기
    - 신규 리피트 사용자: 이전 달 신규 + 이번 달 사용
    - 기존 리피트 사용자: 이전 달 리피트 + 이번 달 사용
    - 컴백 리피트 사용자: 이전 달 컴백 + 이번 달 사용
  - 리피트 사용자를 세분화해서 집계하는 쿼리

  ```SQL
  WITH
  monthly_user_action AS (
    ...
  )
  , monthly_user_with_type AS (
    ...
  )
  , monthly_users AS (
    SELECT
      m1.action_month
      , COUNT(m1.user_id) AS mau
      , COUNT(CASE WHEN m1.c = 'new_user'   THEN 1 END) AS new_users
      , COUNT(CASE WHEN m1.c = 'repeat_user'  THEN 1 END) AS repeat_users
      , COUNT(CASE WHEN m1.c = 'come_back_user' THEN 1 END) AS come_back_users
      
      , COUNT(
        CASE WHEN m1.c = 'repeat_user' AND m0.c = 'new_user' THEN 1 END
      ) AS new_repeat_users
      , COUNT(
        CASE WHEN m1.c = 'repeat_user' AND m0.c = 'repeat_user' THEN 1 END
      ) AS continuous_repeat_users
      , COUNT(
        CASE WHEN m1.c = 'repeat_user' AND m0.c = 'come_back_user' THEN 1 END
      ) AS come_back_repeat_uesrs
    FROM
      -- m1 : 해당 월의 사용자 분류 테이블
      monthly_user_with_type AS m1
      LEFT OUTER JOIN
      -- m0 : 이전 달의 사용자 분류 테이블
      monthly_user_with_type AS m0
      ON m1.user_id = m0.user_id
      AND m1.action_month_priv = m0.action_month
    GROUP BY
      m1.action_month
  ) 
  SELECT
    *
  FROM
    monthly_users
  ORDER BY
    action_month;
  ```

- MAU 속성별 반복률 계산하기
  - 위의 구성비 만으로는 리피트 전환, 컴백 캠페인 효과를 측정하기는 어려움
  - MAU 내역과 MAU 속성들의 반복률을 계산하는 쿼리

  ```SQL
  WITH
  monthly_user_action AS (
    ...
  )
  , monthly_user_with_type AS (
    ...
  )
  , monthly_users AS (
    ...
  )
  SELECT
    action_month
    , mau
    , new_users
    , repeat_users
    , come_back_users
    , new_repeat_users
    , continuous_repeat_users
    , come_back_repeat_users

    -- PostgreSQL, Redshift, BigQuery
    -- Hive의 경우 NULLIF를 CASE로 변경
    -- SparkSQL, LAG 함수의 프레임에 ROWS BETWEEN 1 PRECEDING AND 1 PRECEDING 지정
    -- 이번 달에 신규 사용자이면서, 해당 월에 신규 리피트 사용자인 사용자의 비율
    , 100.0 * new_repeat_users
      / NULLIF(LAG(new_users) OVER(ORDER BY action_month), 0)
      AS priv_new_repeat_ratio
    
    -- 이전 달에 리피트 사용자이면서, 해당 월에 기존 리피트 사용자인 사용자의 비율
    , 100.0 * continuous_repeat_users
      / NULLIF(LAG(repeat_users) OVER(ORDER BY action_month), 0)
      AS priv_continuous_repeat_ratio
    
    -- 이전 달에 컴백 사용자이면서, 해당 월에 컴백 리피트 사용자인 사용자의 비율
    , 100.0 * come_back_repeat_users
      / NULLIF(LAG(come_back_users) OVER(ORDER BY action_month), 0)
      AS priv_come_back_repeat_ratio
    
  FROM
    monthly_users
  ORDER BY
    action_month
  ;
  ```

  - 월 단위 집계로, 1일에 등록한 사용자가 30일 이상 사용하지 않으면 리피트 사용자가 되지 않음.
  - 판정기간에 문제가 있지만 간단하게 추이를 확인하거나 서비스끼리 비교할 때는 굉장히 유용한 리포트
  - 판정기간에 신경 쓰이면 독자적인 정의로 집계

### 12-8 방문 종류를 기반으로 성장지수 집계하기

---

- 사용자의 사용, 리텐션 등을 높이기 위한 팀이 있음. = 그로스 해킹
- 서비스의 성장을 지표화하거나, 그로스 해킹 팀의 성과를 지표화하는 방법의 하나로 '성장지수'라는 지표를 알아봄

#### 성장지수

- 사용자의 서비스 사용과 관련한 상태 변화를 수치화해서 서비스가 성장하는지 알려주는 지표
- 1이상이면 성장중 0보다 낮으면 퇴보중
- 4가지 상태 변화 패턴을 통해 산출함
  - 성장지수 = Signup + Reactivation - Deactivation - Exit
    - Signup : 신규 등록하고 사용을 시작
    - Deactivation : 액티브 유저 -> 비액티브 유저
    - Reactivation : 비액티브 유저 -> 액티브 유저
    - Exit : 서비스를 탈퇴하거나 사용 중지
- `서비스를 쓰게 된 사용자` `떠난 사용자`를 집계해서 수치화한 것
- 성장지수를 늘리는 2가지 방법
  - Signup과 Reactivation 높이기
  - Deactivation 낮추기
- 성장지수 산출을 위해 사용자 상태를 집계하는 쿼리

``` SQL
WITH
unique_action_log AS (
  -- 같은 날짜 로그를 중복하여 세지 않도록 중복 배제
  SELECT DISTINCT
    user_id
    , substring(stamp, 1, 10) AS action_date
    -- BigQuery
    , substr(stamp, 1, 10) AS action_date
  FROM
    action_log
)
, mst_calendar AS (
  -- 집계하고 싶은 기간을 캘린더 테이블로 생성
  -- generate_series 등으로 동적 생성도 가능
            SELECT '2016-10-01' AS dt
  UNION ALL SELECT '2016-10-02' AS dt
  UNION ALL SELECT '2016-10-03' AS dt
  -- 생략
  UNION ALL SELECT '2016-11-04' AS dt
)
, target_date_with_user AS (
  -- 사용자 마스터에 캘린더 테이블의 날짜를 target_date로 추가
  SELECT
    c.dt AS target_date
    , u.user_id
    , u.register_date
    , u.withdraw_date
  FROM
    mst_users AS u
    CROSS JOIN
      mst_calendar AS c
)
, user_status_log AS (
  SELECT
    u.target_date
    , u.user_id
    , u.regsiter_date
    , u.withdraw_date
    , a.action_date
    , CASE WHEN u.register_date = a.action_date THEN 1 ELSE 0 END AS is_new
    , CASE WHEN u.withdraw_date = a.action_date THEN 1 ELSE 0 END AS is_exit
    , CASE WHEN u.target_date = a.action_date THEN 1 ELSE 0 END AS is_access
    , LAG(CASE WHEN u.target_date = a.action_date TEHN 1 ELSE 0 END)
      OVER(
        PARTITION BY u.user_id
        ORDER BY u.target_date
        -- SparkSQL, 다음과 같이 프레임 지정
        ROWS BETWEEN 1 PRECEDING AND 1 PRECEDING
      ) AS was_access
  FROM
    target_date_With_user AS u
    LEFT JOIN
      unique_action_log AS a
      ON u.user_id = a.user_id
      AND u.target_date = a.action_date
  WHERE
    -- 집계 기간을 등록일 이후로만 필터링
    u.register_date <= u.target_date
    -- 탈퇴 날짜가 포함되어 있으면, 집계 기간을 탈퇴 날짜 이전만으로 필터링
    AND (
      u.withdraw_date IS NULL
      OR u.target_date <= u.withdraw_date
    )
)
SELECT
  target_date
  , user_id
  , is_new
  , is_exit
  , is_access
  , was_access
FROM
  user_status_log
;
```

- 매일의 성장지수를 계산하는 쿼리

```SQL
WITH
unique_action_log AS (
  ...
)
, mst_calendar AS (
  ...
)
, target_date_with_user AS (
  ...
)
, user_status_log AS (
  ...
)
, user_growth_index AS (
  SELECT
    *
    , CASE
      -- 어떤 날짜에 신규 등록 또는 탈퇴한 경우 signup | exit으로 판정
      WHEN is_new + is_exit = 1 THEN
        CASE
          WHEN is_new = 1 THEN 'signup'
          WHEN is_exit = 1 THEN 'exit'
        END
      -- 신규 등록과 탈퇴가 아닌 경우 reactivation 또는 deactivation으로 판정
      -- 이때 reactivation, deactivation의 정의에 맞지 않는 경우는 NULL
      WHEN is_new + is_exit = 0 THEN
        CASE
          WHEN was_access = 0 AND is_access = 1 THEN 'reactivation'
          WHEN was_access = 1 AND is_access = 0 THEN 'deactivation'
        END
      --어떤 날짜에 신규 등록과 탈퇴를 함께 했다면(is_new + is_exit = 2) NULL로 지정
      END AS growth_index
  FROM
    user_status_log
)
SELECT
  target_date
  , SUM(CASE growth_index WHEN 'signup'         THEN 1 ELSE 0 END) AS signup
  , SUM(CASE growth_index WHEN 'reactivation'   THEN 1 ELSE 0 END) AS reactivation
  , SUM(CASE growth_index WHEN 'deactivation'   THEN -1 ELSE 0 END) AS deactivation
  , SUM(CASE growth_index WHEN 'exit'           THEN -1 ELSE 0 END) AS exit
  -- 성장 지수 정의에 맞게 계산
  , SUM(
      CASE growth_index
        WHEN 'signup'       THEN 1
        WHEN 'reactivation' THEN 1
        WHEN 'deactivation' THEN -1
        WHEN 'exit'         THEN -1
        ELSE 0
      END
    ) AS growth_index
FROM
  user_growth_index
GROUP BY
  target_date
ORDER BY
  target_date
;
```

- 성장지수의 성장/하락 포인트마다 데이터를 비교해서 서비스를 성장시켜야 함

### 12-9 지표 개선 방법 익히기

---

- 향상 방법
  1. 높이고 싶은 **지표**를 결정
  2. 지표에 영향을 많이 줄 것으로 보이는 **행동**을 결정
  3. 2번의 행동을 기준으로 1번의 지표차이가 나는 부분들을 비교한다.

- 예시
  - 글의 업로드 댓글수를 늘리고 싶은 경우?
    - 팔로잉/팔로워/프로필 사진 유무 등
  - 신규 사용자의 리피트율 개선?
    - 게시글 개수/팔로잉/7일 이내 사용 일수 등
  - CVR을 개선하고 싶은 경우
    - 상세페이지뷰/관심기능 등

## 13강 시계열에 따른 사용자의 개별적인 행동 분석하기

- 액션의 **타이밍**에 중점을 둠
- 결과에 이르기 까지 어떤 과정과 기간이 걸리는지 보기 위함

### 13-1 사용자의 액션 간격 집계하기

---

- 신청일과 숙박일의 리드 타임을 계산하는 쿼리(같은 레코드)

```sql
WITH
reservations(reservation_id, register_date, visit_date, days) As (
  -- PostgreSQL의 경우 VALUES 구문으로 일시 테이블 생성 가능
  -- Hive, Redshift, BigQuery, SparkSQL의 경우 UNION ALL로 대체 가능
  VALUES
      (1, date '2016-09-01', date '2016-10-01', 3)
    , (2, date '2016-09-20', date '2016-10-01', 2)
    , (3, date '2016-09-30', date '2016-11-20', 2)
    , (4, date '2016-10-01', date '2017-01-03', 2)
    , (5, date '2016-11-01', date '2016-12-20', 3)
)
SELECT
  reservation_id
  , register_date
  , visit_date
  -- PostgreSQL, Redshift, 날짜끼리 뺄셈 가능
  , visit_date::date - register_date::date AS lead_time
  -- BigQuery, date_diff
  , date_diff(date(timestamp(visit_date)), date(timestamp(register_date)), day) AS lead_time
  -- Hive, SparkSQL, datediff 함수 사용
  , datediff(to_date(visit_date), to_date(register_date)) AS lead_time
FROM
  reservations
;
```

- 각 단계에서의 리드 타임과 토탈 리드 타임을 계산하는 쿼리(다른 테이블)

```sql
WITH
requests(user_id, product_id, request_date) AS (
  -- PostgreSQL의 경우 VALUES로 일시 테이블 생성 가능
  -- Hive, Redshift, BigQuery, SparkSQL의 경우 UNION ALL로 대체
  VALUES
      ('U001', '1', date '2016-09-01')
    , ('U001', '2', date '2016-09-20')
    , ('U002', '3', date '2016-09-30')
    , ('U003', '4', date '2016-10-01')
    , ('U004', '5', date '2016-01-01')
)
, estimates(user_id, product_id, estimate_date) AS (
  VALUES
      ('U001', '2', date '2016-09-21')
    , ('U002', '3', date '2016-10-15')
    , ('U003', '4', date '2016-09-30')
    , ('U004', '5', date '2016-12-01')
)
, orders(user_id, product_id, order_date) AS (
  VALUES
      ('U001', '2', date '2016-10-01')
    , ('U004', '5', date '2016-12-05')
)
SELECT
  r.user_id
  , r.product_id
  -- PostgreSQL, Redshift, 날짜끼리 뺄셈 가능
  , e.estimate_date::date - r.request_date::date AS estimate_lead_time
  , o.order_date::date    - e.estimate_date::date AS order_lead_time
  , o.order_date::date    - r.request_date::date AS total_lead_time
  -- BigQuery, date_diff
  , date_diff(date(timestamp(e.estimate_date)), date(timestamp(r.request_date)), day) AS estimate_lead_time
  , date_diff(date(timestamp(o.order_date)), date(timestamp(e.estimate_date)), day) AS order_lead_time
  , date_diff(date(timestamp(o.order_date)), date(timestamp(r.request_date)), day) AS total_lead_time
  -- Hive, SparkSQL, datediff
  , datediff(to_date(e.estimate_date), to_date(r.request_date)) AS estimate_lead_time
  , datediff(to_date(o.order_date), to_date(e.estimate_date)) AS order_lead_time
  , datediff(to_date(o.order_Date), to_date(r.request_date)) AS total_lead_time
FROM
  requests AS r
  LEFT OUTER JOIN
    estimates AS e
      ON r.user_id = e.user_id 
      AND r.product_id = e.product_id
  LEFT OUTER JOIN
    orders AS o
      ON r.user_id = o.user_id
      AND r.product_id = o.product_id
;
```

- 이전 구매일로부터 일수를 계산하는 쿼리(같은 테이블)

```sql
WITH
purchase_log(user_id, product_id, purchase_date) AS (
  -- PostgreSQL, VALUES로 일시 테이블 생성
  -- Hive, Redshift, BigQuery, SparkSQL의 경우 UNION ALL로 대체
  VALUES
      ('U001', '1', '2016-09-01')
    , ('U001', '2', '2016-09-20')
    , ('U002', '3', '2016-09-30')
    , ('U001', '4', '2016-10-01')
    , ('U002', '5', '2016-11-01')
)
SELECT
  user_id
  , purchase_date
  -- PostgreSQL, Redshift, 날짜끼리 뺄셈
  , purchase_date::date
    - LAG(purchase_date::date)
      OVER(
        PARTITION BY user_id
        ORDER BY purchase_date
      ) AS lead_time
  -- BigQuery, date_diff 함수
  , date_diff(to_date(purchase_date), 
      LAG(to_date(purchase_date))
        OVER(PARTITION BY user_id ORDER BY purchase_date)) AS lead_time
  -- SparkSQL의 경우 datediff 함수에 LAG 함수의 프레임 지정이 필요
  , datediff(to_date(purchase_date), 
      LAG(to_date(purchase_date))
        OVER(PARTITION BY user_id ORDER BY purchase_date
          ROWS BETWEEN 1 PRECEDING AND 1 PRECEDING)) AS lead_time
FROM
  purchase_log
;
```

### 13-2 카트 추가 후에 구매했는지 파악하기

- 카트 탈락의 원인
  - 상품 구매 절차상 문제
  - 예상치 못한 비용(배송료, 수수료)
  - 북마크 기능으로만 이용
- 카트에 추가된지 48시간 이내에구매되지 않은 상품 = 카트탈락
- `카트탈락률` = 카트탈락 / 카트에 담긴 상품수
- 상품들이 카트에 추가된 시각과 구매된 시각을 산출하는 쿼리
  - `,` 로 구분된 product id를 구분해야함
  
  ```sql
  WITH
  row_action_log AS (
    SELECT
      dt
      , user_id
      , action
      -- 쉼표로 구분된 product_id 리스트 전개하기
      -- PostgreSQL의 경우 regexp_split_to_table 함수 사용 가능
      , regexp_split_to_table(products, ',') AS product_id
      -- Hive, BigQuery, SparkSQL의 경우, FROM 구문으로 전개하고 product_id 추출
      , product_id
      , stamp
    FROM
      action_log
      -- BigQuery
      CROSS JOIN unnest(split(products, ',')) As product_id
      -- Hive, SparkSQL
      LATERAL VIEW explode(split(products, ',')) e AS product_id

  , action_time_stats AS (
    -- 사용자와 상품 조합의 카트 추가 시간과 구매 시간 추출
    SELECT
      user_id
      , product_id
      , MIN(CASE action WHEN 'add_cart' THEN dt END) AS dt
      , MIN(CASE action WHEN 'add_cart' THEN stamp END) AS add_cart_time
      , MIN(CASE action WHEN 'purchase' THEN stamp END) AS purchase_time
      -- PostgreSQL, timestamp 자료형으로 변환하여 간격을 구한 뒤, EXTRACT(epoc ~)로 초단위 변환
      , EXTRACT(epoch from
        MIN(CASE action WHEN 'purchase' THEN stamp::timestamp END)
        - MIN(CASE action WHEN 'add_cart' THEN stamp::timestamp END))
      -- BigQuery, unix_seconds 함수로 초 단위 UNIX 시간 추출 후 차이 구하기
      , MIN(CASE action WHEN 'purchase' THEN unix_seconds(timestamp(stamp)) END)
        - MIN(CASE action WHEN 'add_cart' THEN unix_seconds(timestamp(stamp)) END)
      -- Hive, Spark, unix_timestamp 함수로 초 단위 UNIX 시간 추출 후 차이 구하기
      , MIN(CASE action WHEN 'purchase' THEN unix_timestamp(stamp) END)
        - MIN(CASE action WHEN 'add_cart' THEN unix_timestamp(stamp) END)
      AS lead_time
    FROM
      row_action_log
    GROUP BY
      user_id, product_id
  )
  SELECT
    user_id
    , product_id
    , add_cart_time
    , purchase_time
    , lead_time
  FROM
    action_time_stats
  ORDER BY
    user_id, product_id
  ;
  ```

- 카트 추가 후 n시간 이내에 구매된 상품 수와 구매율을 집계하는 쿼리

```sql
WITH
row_action_log AS (
  ...
)
, action_time_stats AS (
  ...
)
, purchase_lead_time_flag AS (
  SELECT
    user_id
    , product_id
    , dt
    , CASE WHEN lead_time <= 1 * 60 * 60 THEN 1 ELSE 0 END AS purchase_1_hour
    , CASE WHEN lead_time <= 6 * 60 * 60 THEN 1 ELSE 0 END AS purchase_6_hours
    , CASE WHEN lead_time <= 24 * 60 * 60 THEN 1 ELSE 0 END AS purchase_24_hours
    , CASE WHEN lead_time <= 48 * 60 * 60 THEN 1 ELSE 0 END AS purchase_48_hours
    , CASE
        WHEN lead_time IS NULL OR NOT (lead_time <= 48 * 60 * 60) THEN 1
        ELSE 0
      END AS not_purchase
  FROM action_time_stats
)
SELECT
  dt
  , COUNT(*) AS add_cart
  , SUM(purchase_1_hour) AS purhcase_1_hour
  , AVG(purchase_1_hour) AS purhcase_1_hour_rate
  , SUM(purchase_6_hours) AS purchase_6_hours
  , AVG(purchase_6_hours) AS purhcase_6_hours_rate
  , SUM(purchase_24_hours) AS purchase_24_hours
  , AVG(purchase_24_hours) AS purchase_24_hours_rate
  , SUM(purchase_48_hours) AS purchase_48_hours
  , AVG(purchase_48_hours) AS purchase_48_hours)rate
FROM
  purchase_lead_time_flag
GROUP BY
  dt
;
```

### 13-3 등록으로부터의 매출을 날짜별로 집계하기

- 사용자의 등록으로부터 시간 경과에 따른 매출을 집계해서 광과와 제휴 비용이 적절하게 투자되었는지 판단
- 사용자 등록을 월별로 집계하고, n일 경과 시점의 1인당 매출 금액을 집계
- 사용자들의 등록일부터 경과한 일수별 매출을 계산하는 쿼리

```sql
WITH
index_intervals(index_name, interval_begin_date, interval_end_date) AS (
  -- PostgreSQL, VALUES
  -- Hive, Redshift, BigQuery, SparkSQL, UNION ALL
  VALUES
    ('30 day sales amount', 0, 30)
    , ('45 day sales amount', 0, 45)
    , ('60 day sales amount', 0, 60)
)
, mst_users_with_base_date AS (
  SELECT
    user_id
    -- 기준일로 등록일 사용
    , register_date AS base_date

    -- 처음 구매한 날을 기준으로 삼고 싶다면, 다음과 같이 사용
    , first_purchase_date AS base_date
  FROM
    mst_users
)
, purchase_log_With_index_date AS (
  SELECT
    u.user_id
    , u.base_date
    -- 액션의 날짜와 로그 전체의 최신 날짜를 날짜 자료형으로 변환
    , CASE(p.stamp AS date) AS action_date
    , MAX(CAST(p.stamp AS date)) OVER() AS latest_date
    , substring(p.stamp, 1, 7) AS month
    -- BigQuery, 한 번 타임스탬프 자료형으로 변환하고 날짜 자료형으로 변환
    , date(timestamp(p.stamp)) AS action_date
    , MAX(date(timestamp(p.stamp))) OVER() AS latest_date
    , substr(p.stamp, 1, 7) AS month

    , i.index_name
    -- 지표 대상 기간의 시작일과 종료일 계산
    -- PostgreSQL
    , CASE(u.base_date::date + '1 day'::interval * i.interval_begin_date AS date)
      AS index_begin_date
    , CASE(u.base_date::date + '1 day'::interval * i.interval_end_date AS date)
      AS index_end_date
    -- Redshift
    , dateadd(day, r.interval_begin_date, u.base_date::date) AS index_begin_date
    , dateadd(day, r.interval_end_date, u.base_date::date) AS index_end_date
    -- BigQuery
    , date_add(CAST(u.base_date AS date), interval r.interval_begin_date day)
      AS index_begin_date
    , date_add(CAST(u.base_date AS date), interval r.interval_end_date day)
      AS index_end_date
    -- Hive, SparkSQL
    , date_add(CAST(u.base_date AS date), r.interval_begin_date)
      AS index_begin_date
    , date_add(CAST(u.base_date AS date), r.interval_end_date)
      AS index_end_date
    , p.amount
  FROM
    mst_users_with_base_date AS u
    LEFT OUTER JOIN
      action_log AS p
      ON u.user_id = p.user_id
      AND p.action = 'purchase'
    CROSS JOIN
      index_intervals AS i
)
SELECT *
FROM
  purchase_log_with_index_date
;
```

- 월별 등록자수와 경과일수별 매출을 집계하는 쿼리
  - 로그 날짜가 포함되어 있는지 아닌지로 구분
  - 등록 후의 경과일수로 매출 계산
  - 0과 1의 액션 플래그가 아닌 구매액을 리턴
  - 분모를 서비스 사용자 수로 넣으면 ARPU, 과금 사용자 수를 넣으면 ARPPU

```sql
WITH index_intervals(index_name, interval_begin_date, interval_end_date) AS (
  ...
)
, mst_users_with_base_date AS (
  ...
)
, purchase_log_with_index_date AS (
  ...
)
, user_purchase_amount AS (
  SELECT
    user_id
    , month
    , index_name
    -- 3. 지표 대상 기간에 구매한 금액을 사용자별로 합계
    , SUM (
      -- 1. 지표의 대상 기간의 종료일이 로그의 최신 날짜에 포함되었는지 확인
      CASE WHEN index_end_date <= latest_date THEN
        -- 2. 지표의 대상 기간에 구매한 경우에는, 구매 금액, 이외에는 0 지정
        CASE
          WHEN action_date BETWEEN index_begin_date AND index_end_date THEN amount ELSE 0
        END
      END
    ) AS index_date_amount
  FROM
    purchase_log_with_index_date
  GROUP BY
    user_id, month, index_name, index_begin_date, index_end_date
)
SELECT
  month
  -- 등록자 수 세기
  -- 다만 지표와 대상 기간의 종료일이 로그의 최신 날짜 이전에 포함되지 않게 조건 걸기
  , COUNT(index_date_amount) AS users
  , index_name
  -- 지표의 대상 기간 동안 구매한 사용자 수
  , COUNT(CASE WHEN index_date_amount > 0 THEN user_id END) AS purchase_uu
  -- 지표와 대상 기간 동안의 합계 매출
  , SUM(index_date_amount) AS total_amount
  -- 등록자별 평균 매출
  , AVG(index_date_amount) AS avg_amount
FROM
  user_purchase_amount
GROUP BY
  month, index_name
ORDER BY
  month, index_name
;
```

#### LTV(고객 생애 가치)

- Life Time Value
- CPA(Cost Per acquistion) 고객 획득 가치를 설정/관리할 때 중요한 지표
- LTV = <연간 거래액> x <수익률> x <지속 연수(체류 기간)>

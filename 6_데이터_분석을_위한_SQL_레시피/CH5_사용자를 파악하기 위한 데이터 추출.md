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

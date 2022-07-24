# CH6_웹사이트에서의 행동을 파악하는 데이터 추출하기

---

## 14강 사이트 전체의 특징/경향 찾기

---

- 빅데이터 기반으로 값을 추출하는 것은 기존의 파악한 값들과 다를 수 있음
- 사용자나 상품 데이터와 조합을 한다면 새로운 리포트 작성 가능

### 14-1 날짜별 방문자 수 / 방문 횟수 / 페이지 뷰 집계하기

- 지표 정의
  - 방문자 수: 브라우저를 꺼도 사라지지 않는 쿠키의 유니크 수
  - 방문 횟수: 브라우저를 껐을 때 사라지는 쿠키의 유니크 수
  - 페이지 뷰: 페이지를 출력한 로그의 수
- 날짜별 접근데이터를 집계하는 쿼리
  - 로그인/비로그인 UU를 구분하는 것도 좋은 방법

```sql
SELECT
  -- PostgreSQL, Hive, Redshift, SparkSQL, subString으로 날짜 부분 추출
  substring(stamp, 1, 10) as dt
  -- PostgreSQL, Hive, BigQuery, SparkSQL, substr 사용
  , substr(stamp, 1, 10) AS dt
  -- 쿠키 계산하기
  , COUNT(DISTINCT long_session) AS access_users
  -- 방문 횟수 계산
  , COUNT(DISTINCT short_session) AS access_count
  -- 페이지 뷰 계산
  , COUNT(*) AS page_view

  -- 1인당 페이지 뷰 수
  -- PostgreSQL, Redshift, BigQuery, SparkSQL, NULLIF 함수 사용
  , 1.0 * COUNT(*) / NULLIF(COUNT(DISTINCT long_session), 0) AS pv_per_user
  -- Hive, NULLIF 함수 대신 CASE 사용
  , 1.0 * COUNT(*)
      / CASE
          WHEN COUNT(DISTINCT long_session) <> 0 THEN COUNT(DISTINCT long_session)
        END
    AS pv_per_user
FROM
  access_log
GROUP BY
  -- PostgreSQL, Redshift, BigQuery
  -- SELECT 구문에서 정의나 별칭을 GROUP BY에 지정할 수 있음
  dt
  -- PostgreSQL, Hive, Redshift, SparkSQL
  -- SELECT 구문에서 별칭을 지정하기 이전의 식을 GROUP BY에 지정할 수 있음
  substring(stamp, 1, 10)
ORDER BY
  dt
;
```

### 14-2 페이지별 쿠키 / 방문 횟수 / 페이지 뷰 집계하기

- URL에는 화면 제어를 위한 파라미터가 포함되어 있음
- URL별로 집계하는 쿼리

```SQL
SELECT
  url
  , COUNT(DISTINCT short_session) AS access_count
  , COUNT(DISTINCT long_session) AS access_users
  , COUNT(*) AS page_view
FROM
  access_log
GROUP BY
  url
;
```

- 경로별로 집계하기
  - 전체 집계는 밀도가 너무 작아 상사 페이지를 집계하기

```sql
WITH
access_log_with_path AS (
  -- URL에서 경로 추출
  SELECT
    -- PostgreSQL의 경우, 정규 표현식으로 경로 부분 추출
    , substring(url from '//[^/]+([^?#]+)') AS url_path
    -- Redshift의 경우, regexp_substr 함수와 regexp_replace 함수를 조합하여 사용
    , regexp_replace(regexp_substr(url, '//[^/]+[^?#]+'), '//[^/]+', '') AS url_path
    -- BigQuery의 경우 정규 표현식에 regexp_extract 사용
    , regexp_extract(url, '//[^/]+([^?#]+)') AS url_path
    -- Hive, SparkSQL, parse_url 함수로 url 경로 부분 추출
    , parse_url(url, 'PATH') AS url_path
  FROM access_log
)
SELECT
  url_path
  , COUNT(DISTINCT short_session) AS access_count
  , COUNT(DISTINCT long_session) AS access_users
  , COUNT(*) AS page_view
FROM
  access_log_with_path
GROUP BY
  url_path
;
```

- URL에 의미를 부여해서 집계하기
  - 카테고리별로 묶기 ex) /list/cd, /list/dvd
  - 로그를 설계할 때 페이지 이름을 따로 전송해주면 후의 집계작업이 편해짐

```sql
WITH
access_log_with_path AS (
  ...
)
, access_log_with_split_path AS (
  -- 경로의 첫 번째 요소와 두 번째 요소 추출하기
  SELECT *
    -- PostgreSQL, Redshift, split_part로 n번째 요소 추출하기
    , split_part(url_path, '/', 2) AS path1
    , split_part(url_path, '/', 3) AS path2
    -- BigQuery, split 함수로 배열로 분해하고 추출하기
    , split(url_path, '/')[SAFE_ORDINAL(2)] AS path1
    , split(url_path, '/')[SAFE_ORDINAL(3)] AS path2
    -- Hive, SparkSQL, split 함수로 배열로 분해하고 추출하기
    , split(url_path, '/')[1] AS path1
    , split(url_path, '/')[2] AS path2
  FROM access_log_with_path
)
, access_log_with_page_name AS (
  -- 경로를 /로 분할하고, 조건에 따라 페이지 이름 붙이기
  SELECT *
  , CASE
      WHEN path1 = 'list' THEN
        CASE
          WHEN path2 = 'newly' THEN 'newly_list'
          ELSE 'category_list'
        END
      -- 이외의 경우에는 경로 그대로 사용
      ELSE url_path
    END AS page_name
  FROM access_log_with_split_path
)
SELECT
  page_name
  , COUNT(DISTINCT short_session) AS access_count
  , COUNT(DISTINCT long_session) AS access_users
  , COUNT(*) FROM page_view
FROM access_log_with_page_name
GROUP BY page_name
ORDER BY page_name
;
```

### 14-3 유입원별로 방문 횟수 또는 CVR 집계하기

- 유입 경로
  - 검색 엔진
  - 검색 연동형 광고
  - 트위터, 페이스북 등의 소셜 미디어
  - 제휴 사이트
  - 광고 네트워크
  - 블로그 등 타 사이트 링크
- 마케팅 부문의 효과를 시각적으로 표현하는 목적으로도 사용할 수 있음
- 직전 페이지를 `referer` 라고 부름. 이것을 보고 URL 판정 가능
- URL 생성도구
  - Campaign URL Builder를 사용해 생성된 URL을 살펴보면 아래와 같음
  - 매개변수를 보고 다양한 유입 계측이 가능
    **URL 생성 도구 항목**|**URL 매개변수**|**용도**
    -----|-----|-----
    Campain Source(필수)|utm\_source|The referer; 사이트 이름 또는 매체의 명칭
    Compaign Medium|utm\_medium|Marketing medium; 유입 분류
    Compaign Name|utm\_compaign|Product, promo code, or slogan; 캠페인 페이지의 이름
    Compaign Term|utm\_term|Identity the paid keywords; 유입 키워드
    Compaign Content|utm\_content|Use to differentiate ads; 광고를 분류하는 식별자
- 레퍼러 도메인과 랜딩 페이지
  - 직접 URL 매겨변수를 조작할 수 없으면 referer를 사용해서 파악해야함
- 유입원별로 방문 횟수를 집계하는 쿼리
  - 유입원 구분
    - **유입 경로**|**판정 방법**|**집계 방법**
        -----|-----|-----
        검색 연동 광고|URL 생성 도구로 만들어진 매개 변수|URL에 utm\_source, utm\_medium이 있을 경우, 이러한 두 개의 문자열을 결합하여 집계
        제휴 마케팅 사이트|URL 생성 도구로 만들어진 매개 변수|URL에 utm\_source, utm\_medium이 있을 경우, 이러한 두 개의 문자열을 결합하여 집계
        검색 엔진|도메인|레퍼러의 도메인이 다음과 같은 검색엔진일 때 <br> - search.naver.com <br /> - www.google.co.kr
        소셜 미디어|도메인|레퍼러의 도메인이 다음과 같은 소셜 미디어일 때 <br /> - twitter.com <br /> - www.facebook.com
        기타 사이트|도메인|위에 언급한 도메인이 아닐 경우, other라는 이름으로 집계

```sql
WITH
access_log_with_parse_info AS (
  -- 유입원 정보 추출하기
  SELECT *
    -- PostgreSQL, 정규 표현식으로 유입원 정보 추출하기
    , substring(url from 'https?://([^\]*)') AS url_domain
    , substring(url from 'utm_source=([^&]*)') AS url_utm_source
    , substring(url from 'utm_medium=([^&]*)') AS url_utm_medium
    , substring(referer from 'https?://([^/]*)') AS referer_domain
    -- Redshift, regexp_substr 함수와 regexp_replace 함수를 조합하여 사용
    , regexp_replace(regexp_substr(url, 'https?://[^/]*'), 'https?://', '') AS url_domain
    , regexp_replace(regexp_substr(url, 'utm_source=[^&]*'), 'utm_source=', '') AS url_utm_source
    , regexp_replace(regexp_substr(url, 'utm_medium=[^&]*'), 'utm_medium=', '') AS url_utm_medium
    , regexp_replace(regexp_substr(referer, 'https?://[^/]*'), 'https?://', '') AS referer_domain
    -- BigQuery, 정규표현식 regexp_extract 사용
    , regexp_extract(url, 'https?://([^/]*)') AS url_domain
    , regexp_extract(url, 'utm_source=([^&]*)') AS url_utm_source
    , regexp_extract(url, 'utm_medium([^&]*)') AS url_utm_medium
    , regexp_extract(referer, 'https?://([^/]*)') AS referer_domain
    -- Hive, SparkSQL, parse_url 함수 사용
    , parse_url(url, 'HOST') AS url_domain
    , parse_url(url, 'QUERY', 'utm_source') AS url_utm_source
    , parse_url(url, 'QUERY', 'utm_medium') AS url_utm_medium
    , parse_url(referer, 'HOST') AS referer_domain
  FROM access_log
)
, access_log_with_via_info AS (
  SELECT *
    , ROW_NUMBER() OVER(ORDER BY stamp) AS log_id
    , CASE
        WHEN url_utm_source <> '' AND url_utm_medium <> ''
          -- PostgreSQL, Hive, BigQuery, SpaerkSQL, concat 함수에 여러 매개변수 사용 가능
          THEN concat(url_utm_source, '-', url_utm_medium)
          -- PostgreSQL, Redshift, 문자열 결합에 || 연산자 사용
          THEN url_utm_source || '-' || url_utm_medium
        WHEN referer_domain IN ('search.naver.com', 'www.google.co.kr') THEN 'search'
        WHEN referer_domain IN ('twitter.com', 'www.facebook.com') THEN 'social'
        ELSE 'other'
      -- ELSE referer_domain으로 변경하면, 도메인별 집계 가능
      END AS via
  FROM access_log_with_parse_info
  -- 레퍼러가 없는 경우와 우리 사이트 도메인의 경우 제외
  WHERE COALESCE(referer_domain, '') NOT IN ('', url_domain)
)
SELECT via, COUNT(1) AS access_count
FROM access_log_with_via_info
GROUP BY via
ORDER BY access_count DESC;
```

- 유입원별로 CVR 집계하기
  - 각 방문에서 구매한 비율
  - access_log_with_purcahse_amount 테이블을 수정하면 다양한 액션이 비율을 집계 가능
  - 어떤 유입경로에 신경써야 하는지 판단 가능

```sql
WITH
access_log_with_parse_info AS (
  ...
)
, access_log_with_via_info AS (
  ...
)
, access_log_with_purchase_amount AS (
  SELECT
    a.log_id
    , a.via
    , SUM(
        CASE
          -- PostgreSQL, interval 자료형의 데이터로 날짜와 시간 사칙연산 가능
          WHEN p.stamp::date BETWEEN a.stamp::date AND a.stamp::date + '1 day'::interval
          -- Redshift, dateadd 함수
          WHEN p.stamp::date BETWEEN a.stamp::date AND dateadd(day, 1, a.stamp::date)
          -- BigQuery, date_add 함수
          WHEN date(timestamp(p.stamp))
            BETWEEN date(timestamp(a.stamp))
              AND date_add(date(timestamp(a.stamp)), interval 1 day)
          -- Hive, SparkSQL, date_add
          -- *BigQuery와 서식이 조금 다름
          WHEN to_date(p.stamp)
            BETWEEN to_date(a.stamp) AND date_add(to_date(a.stamp), 1)

            THEN amount
        END
      ) AS amount
  FROM
    access_log_with_via_info AS a
    LEFT OUTER JOIN
      purchase_log AS p
      ON a.long_session = p.long_session
  GROUP BY a.log_id, a.via
)
SELECT
  via
  , COUNT(1) AS via_count
  , count(amount) AS conversions
  , AVG(100.0 * SIGN(COALESCE(amount, 0))) AS cvr
  , SUM(COALESCE(amount, 0)) AS amount
  , AVG(1.0 * COALESCE(amount, 0)) AS avg_amount
FROM
  access_log_with_purchase_amount
GROUP BY via
GROUP BY cvr DESC
;
```

### 14-4 접근 요일, 시간대 파악하기

- 서비스에 따라 사용자가 접근하는 요일 시간대가 달라집
- 공지사항, 캠페인 시작/종료, 푸시메시지 발신 시간등을 검토할 수 있음
- 리포트를 만드는 과정
  - 24시간에서 추출하고자 하는 단위를 결정(10분, 30분 등등)
  - 접근한 시간을 해당 단위로 집계하고, 요일과 함께 방문자 수를 집계
- 요일/시간대별 방문자 수를 집계하는 쿼리

```sql
WITH
access_log_with_dow AS (
  SELECT
    stamp
    -- 일요일(0)부터 토요일(6)까지의 요_일 번호 추출하기
    -- PostgreSQL, Redshift의 경우 date_part 함수 사용하기
    , date_part('dow', stamp::timestamp) AS dow
    -- BigQuery의 경우 EXTRACT(dayofweek from ~) 함수 사용하기
    , EXTRACT(dayofweek from timestamp(stamp)) - 1 AS dow
    -- Hive, Spark의 경우 from_unixtime 함수 사용하기
    , from_unixtime(unix_timestamp(stamp), 'u') % 7 dow

    -- 00:00:00부터의 경과 시간을 초 단위로 계산
    -- PostgreSQL, Hive, Redshift, SparkSQL
    --  substring 함수를 사용해 시, 분, 초를 추출하고 초 단위로 환산하여 더하기
    -- BigQuery의 경우 substring을 substr, int를 int64로 수정하기
    , CAST(substring(stamp, 12, 2) AS int) * 60 * 60
      + CAST(substring(stamp, 15, 2) AS int) * 60
      + CAST(substring(stamp, 18, 2) AS int)
      AS whole_seconds
    
    -- 시간 간격 정하기
    -- 현재 예제에서는 30분(1800초)으로 지정하기
    , 30 * 60 AS interval_seconds
  FROM access_log
)
, access_log_with_floor_seconds AS (
  SELECT
    stamp
    , dow
    -- 00:00:00부터의 경과 시간을 interval_seconds로 나누기
    -- PostgreSQL, Hive, Redshift, SparkSQL의 경우는 다음과 같이 사용하기
    -- BigQuery의 경우 int를 int64로 수정하기
    , CAST((floor(while_seconds / interval_seconds) * interval_seconds) AS int)
      AS floor_seconds
  FROM access_log_with_dow
)
, access_log_with_index AS (
  SELECT
    stamp
    , dow
    -- 초를 다시 타임스탬프 형식으로 변환하기
    -- PostgreSQL, Redshift의 경우는 다음과 같이 하기
    , lpad(floor(floor_seconds / (60 * 60))::text, 2, '0') || ':'
        || lpad(floor(floor_seconds % (60 * 60) / 60)::text, 2, '0') || ':'
        || lpad(floor(floor_seconds % 60)::text, 2, '0')
    -- BigQuery의 경우 다음과 같이 하기
    , concat(
        lpad(CAST(floor(floor_seconds / (60 * 60)) AS string), 2, '0'), ':'
        , lapd(CAST(floor(floor_seconds, 60 * 60)) / 60 AS string), ':'
        , lpad(CAST(floor(floor_seconds % 60)) AS string), 2, '0')
    -- Hive, SparkSQL의 경우
    , concat(
        lpad(CAST(floor(floor_seconds / (60 * 60)) AS string), 2, '0'), ':'
        , lpad(CAST(floor(floor_seconds % (60 * 60) / 60) AS string), 2, '0'), ':'
        , lpad(CAST(floor(floor_seconds % 60) AS string), 2, '0')
      )
      AS index_name
  FROM access_log_with_floor_seconds
)
SELECT
  index_time
  , COUNT(CAST dow WHEN 0 THEN 1 END) AS sun
  , COUNT(CAST dow WHEN 1 THEN 1 END) AS mon
  , COUNT(CAST dow WHEN 2 THEN 1 END) AS tue
  , COUNT(CAST dow WHEN 3 THEN 1 END) AS wed
  , COUNT(CAST dow WHEN 4 THEN 1 END) AS thu
  , COUNT(CAST dow WHEN 5 THEN 1 END) AS fri
  , COUNT(CAST dow WHEN 6 THEN 1 END) AS sat
FROM
  access_log_with_index
GROUP BY
  index_name
ORDER BY
  index_name
;
```

## 15강 사이트 내의 사용자 행동 파악하기

---

- 웹사이트에서의 특정적인 지표(방문자 수, 방문 횟수, 이탈율 등), 리포트(사용자 흐름, 폴아웃 리포트) 작성하는 SQL
- 입구 페이지와 출구 페이지 집계하기
  - 시간차 순으로 정렬하여 `first_value` 와 `last_value`를 사용하여 값을 찾음

```sql
WITH
activity_log_with_landing_exit AS (
  SELECT
    session
    , path
    , stamp
    , FIRST_VALUE(path)
      OVER(
        PARTITION BY session
        ORDER BY stamp ASC
          ROWS BETWEEN UNBOUNDED PRECEDING
                    AND UNBOUNDED FOLLOWING
      ) AS landing
    , LAST_VALUE(path)
      OVER(
        PARTITION BY session
        ORDER BY stamp ASC
          ROWS BETWEEN UNBOUNDED PRECEDING
                    AND UNBOUNDED FOLLOWING
      ) AS exit
  FROM activity_log
)
SELECT *
FROM
  activity_log_with_landing_exit
;
```

- 각 세션의 입구 페이지와 출구 페이지를 기반으로 방문 횟수를 추출하는 쿼리

```sql
WITH
activity_log_with_landing_exit AS (
  ...
)
, landing_count AS (
  -- 입구 페이지의 방문 횟수 집계
  SELECT
    landing AS path
    , COUNT(DISTINCT session) AS count
  FROM
    activity_log_with_landing_exit
  GROUP BY landing
)
, exit_count AS (
  -- 출구 페이지의 방문 횟수 집계
  SELECT
    exit AS path
    , COUNT(DISTINCT session) AS count
  FROM
    activity_log_with_landing_exit
  GROUP BY exit
)
-- 입구 페이지와 출구 페이지 방문 횟수 결과 한번에 출력
SELECT 'landing' AS type, * FROM landing_count
UNION ALL
SELECT 'exit' AS type, * FROM exit_count
;
```

- 세션별 입구 페이지와 출구 페이지의 조합을 집계하는 쿼리

```sql
WITH
activity_log_with_landing_exit AS (
  ...
)
, landing_count AS (
  -- 입구 페이지의 방문 횟수 집계
  SELECT
    landing AS path
    , COUNT(DISTINCT session) AS count
  FROM
    activity_log_with_landing_exit
  GROUP BY landing
)
, exit_count AS (
  -- 출구 페이지의 방문 횟수 집계
  SELECT
    exit AS path
    , COUNT(DISTINCT session) AS count
  FROM
    activity_log_with_landing_exit
  GROUP BY exit
)
-- 입구 페이지와 출구 페이지 방문 횟수 결과 한번에 출력
SELECT 'landing' AS type, * FROM landing_count
UNION ALL
SELECT 'exit' AS type, * FROM exit_count
;
```

### 15-2 이탈률과 직귀율 계산하기

- 출구 페이지를 통해 이탈률 계산하여 문제가 되는 페이지를 찾아내기
- 직귀율 = 특정 페이지만 조회하고 곧바로 이탈한 비율
- 이탈률 집계하기
  - `이탈률 = 출구 수 / 페이지뷰`
  - 단순히 이탈률이 높은 페이지가 나쁜 페이지는 아님. 상세페이지나 결제완료페이지는 이탈률이 높아야함
- 경로별 이탈률을 집계하는 쿼리

```sql
WITH
activity_log_with_exit_flag AS (
  SELECT
    *
    -- 출구 페이지 판정
    , CASE
        WHEN ROW_NUMBER() OVER(PARTITION BY session ORDER BY stamp DESC) = 1 THEN 1
        ELSE 0
      END AS is_exit
    FROM
      activity_log
)
SELECT
  path
  , SUM(is_exit) AS exit_count
  , COUNT(1) AS page_view
  , AVG(100.0 * is_exit) AS exit_ratio
FROM
  activity_log_with_exit_flag
GROUP BY path
;
```

- 직귀율 집계하기
  - `직귀율 = 직귀 수 / 입구 수` or `직귀율 = 직귀 수 / 방문 횟수`
  - 여기서는 전자를 사용.
  - 연관 기사나 사용자를 이동시키는 모듈이 정상적이지 않거나 콘텐츠 불만족, 이동이 복잡 등의 다양한 이유가 있음

```sql
WITH
activity_log_with_landing_bounce_flag AS (
  SELECT
    *
    -- 입구 페이지 판정
    , CASE
        WHEN ROW_NUMBER() OVER(PARTITION BY session ORDER BY stamp ASC) = 1 THEN 1
        ELSE 0
      END AS is_landing
    -- 직귀 판정
    , CASE
        WHEN COUNT(1) OVER(PARTITION BY session) = 1 THEN 1
        ELSE 0
      END AS is_bounce
  FROM
    activity_log
)
SELECT
  path
  , SUM(is_bounce) AS bounce_count
  , SUM(is_landing) AS landing_count
  , AVG(100.0 * CASE WHEN is_landing = 1 THEN is_bounce END) AS bounce_ratio
FROM
  actibity_log_with_landing_bounce_flag
GROUP BY path
;
```

### 15-3 성과로 이어지는 페이지 파악하기

- 방문횟수를 늘려도 성과가 발생해야 함
- CVR을 향상시키기 위해 성과가 좋지 않음 검색 기능을 변경 하는 식으로 서비스 개선
- 다양한 비교 패턴
  - 여기서의 성과는 `구인/구직 신청` 이다.
  - 기능 기반 비교(업종, 직종, 조건)
  - 캠페인 기반 비교(축하금, 기프트 카드 켐페인)
  - 페이지 기반 비교(서비스 소개, 비공개 구인)
  - 콘텐츠 종류 기반 비교(그림 유무)
- 컨버전 페이지보다 이전 접근에 플래그를 추가하는 쿼리

```sql
WITH
, activity_log_with_conversion_flag AS (
  SELECT
    session
    , stamp
    , path
    -- 성과를 발생시키는 컨버전 페이지의 이전 접근에 플래그 추가
    , SIGN(SUM(CASE WHEN path = '/complete' THEN 1 ELSE 0 END)
            OVER(PARTITION BY session ORDER BY stamp DESC
              ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW))
      AS has_conversion
  FROM
    activity_log
)
SELECT *
FROM
  activity_log_with_conversion_flag
ORDER BY
  session, stamp
;
```

- 경로들의 방문 횟수와 구성 수를 집계하는 쿼리

```sql
WITH
activity_log_with_conversion_flag AS (
  -- CODE.15.6.
)
SELECT
  path
  -- 방문 횟수
  , COUNT(DISTINCT session) AS sessions
  -- 성과 수
  , SUM(has_conversion) AS conversions
  -- 성과 수 / 방문 횟수
  , 1.0 * SUM(has_conversion) / COUNT(DISTINCT session) AS cvr
FROM
  activity_log_with_conversion_flag
-- 경로별로 집약
GROUP BY path
;
```

### 15-4 페이지 가치 산출하기

- 성과를 수치화하기
  - 

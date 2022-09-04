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
activity_log_with_conversion_flag AS (
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
  - 성과를 `구매`, `의뢰 신청` 등으로 설정
- 페이지 가치를 할당하는 5가지 방법
  - 마지막 페이지
  - 첫 페이지
  - 방문한 모든 페이지에 균등하게 분산
  - 마지막 페이지에 가까울 수록 가중치 부여
  - 첫 페이지에 가까울 수록 가중치 부여
- 페이지 가치 할당을 계산하는 쿼리

```sql
WITH
activity_log_with_session_conversion_flag AS (
  -- CODE.15.6.
)
, activity_log_with_conversion_assign AS (
  SELECT
    session
    , stamp
    , path
    -- 성과에 이르기까지 접근 로그를 오름차순 정렬
    , ROW_NUMBER() OVER(PARTITION BY session ORDER BY stamp ASC) AS asc_order
    -- 성과에 이르기까지 접근 로그를 내림차순으로 순번
    , ROW_NUMBER() OVER(PARTITION BY session ORDER BY stamp DESC) AS desc_order
    -- 성과에 이르기까지 접근 수 세기
    , COUNT(1) OVER(PARTITION BY session) AS page_count

    -- 1. 성과에 이르기까지 접근 로그에 균등한 가치 부여
    , 1000.0 / COUNT(1) OVER(PARTITION BY session) AS fair_assign

    -- 2. 성과에 이르기까지 접근 로그의 첫 페이지 가치 부여
    , CASE
        WITH ROW_NUMBER() OVER(PARTITION BY session ORDER BY stamp ASC) = 1
          THEN 1000.0
        ELSE 0.0
      END AS first_assign
    
    -- 3. 성과에 이르기까지 접근 로그의 마지막 페이지 가치 부여
    , CASE
        WHEN ROW_NUMBER() OVER(PARTITION BY session ORDER BY stamp DESC) = 1
          THEN 1000.0
        ELSE 0.0
      END AS last_assign
    
    -- 4. 성과에 이르기까지 접근 로그의 성과 지점에서 가까운 페이지 높은 가치 부여
    , 1000.0
        * ROW_NUMBER() OVER(PARTITION BY session ORDER BY stamp ASC)
        -- 순번 합계로 나누기( N*(N+1)/2 )
        / ( COUNT(1) OVER(PARTITION BY session)
            * (COUNT(1) OVER(PARTITION BY session) + 1) / 2)
      AS decrease_assign

    -- 5. 성과에 이르기까지의 접근 로그의 성과 지점에서 먼 페이지에 높은 가치 부여
    , 1000.0
        * ROW_NUMBER() OVER(PARTITION BY session ORDER BY stamp DESC)
        -- 순번 합계로 나누기( N*(N+1)/2 )
        / ( COUNT(1) OVER(PARTITION BY session) 
            * (COUNT(1) OVER(PARTITION BY session) + 1) / 2)
      AS increase_assign
  FROM activity_log_with_conversion_flag
  WHERE
    -- conversion으로 이어지는 세션 로그만 추출
    has_conversion = 1
    -- 입력, 확인, 완료 페이지 제외하기
    AND path NOT IN ('/input', '/confirm', '/complete')
)
SELECT
  session
  , asc_order
  , path
  , fair_assign AS fair_a
  , first_assign AS first_a
  , last_assign AS last_a
  , decrease_assign AS dec_a
  , increase_assign AS inc_a
FROM
  activity_log_with_conversion_assign
ORDER BY
  session, asc_order;
``` 

- 경로별 페이지 가치 합계를 구하는 쿼리
```sql
WITH
activity_log_with_session_conversion_flag AS (
  -- CODE.15.6.
)
, activity_log_with_conversion_assign AS (
  -- CODE.15.8.
)
, page_total_values AS (
  -- 페이지 가치 합계 계산
  SELECT
    path
    , SUM(fair_assign) AS fair_assign
    , SUM(first_assign) AS first_assign
    , SUM(last_assign) AS last_assign
  FROM
    activity_log_with_conversion_assign
  GROUP BY
    path
)
SELECT * FROM page_total_values;
```

- 경로들의 평균 페이지 가치를 구하는 쿼리

```sql
WITH
activity_log_with_session_conversion_flag AS (
  -- CODE.15.6.
)
, activity_log_with_conversion_assign AS (
  -- CODE.15.8
), page_total_values AS (
  -- CODE.15.9
)
, page_total_cnt AS (
  -- 페이지 뷰 계산
  SELECT
    path
    , COUNT(1) AS access_cnt -- 페이지 뷰
    -- 방문 횟수로 나누고 싶은 경우, 다음과 같은 코드 작성
    , COUNT(DISTINCT session) AS access_cnt
  FROM
    activity_log
  GROUP BY
    path
)
SELECT
  -- 한 번의 방문에 따른 페이지 가치 계산
  s.path
  , s.access_cnt
  , v.sum_fair / s.access_cnt AS avg_fair
  , v.sum_first / s.access_cnt AS avg_first
  , v.sum_last / s.access_cnt AS avg_last
  , v.sum_dec / s.access_cnt AS avg_dec
  , v.sum_inc / s.access_cnt AS avg_inc
FROM
  page_total_cnt AS s
JOIN
  page_total_values As v
  ON s.path = v.path
ORDER BY
  s.access_cnt DESC;
```

### 15-5 검색 조건들의 사용자 행동 가시화 하기

- 검색 조건을 상사헤가 할수록 성과로 이어지는 비율이 높음
  - CTR: 검색 조건을 활용해서 **상세 페이지**로 이동한 비율
  - CVR: 상세 페이지 조회 후에 **성과**로 이어지는 비율
- URL 매개변수를 통해서 방문 횟수, CTR, CVR 등을 집계
- 클릭 플래그와 컨버전 플래그를 계산하는 쿼리

```sql
WITH
activity_log_with_session_click_conversion_flag AS (
  SELECT
    session
    , stamp
    , path
    , search_type
    -- 상세 페이지 이전 접근에 플래그 추가
    , SIGN(SUM(CASE WHEN path = '/detail' THEN 1 ELSE 0 END)
                OVER(PARTITION BY session ORDER BY stamp DESC
                  ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW))
      AS has_session_click
    -- 성과를 발생시키는 페이지의 이전 접근에 플래그 추가
    , SIGN(SUM(CASE WHEN path='/complete' THEN 1 ELSE 0 END)
                OVER(PARTITION BY session ORDER BY stamp DESC
                  ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW))
      AS has_session_conversion
  FROM activity_log
)
SELECT
  session
  , stamp
  , path
  , search_type
  , has_session_click AS click
  , has_session_conversion AS cnv
FROM
  activity_log_with_session_click_conversion_flag
ORDER BY
  session, stamp
;
```

- 검색 타입별 CTR,CVR을 집계하는 쿼리

```sql
WITH
activity_log_with_session_click_conversion_flag AS (
  -- CODE.15.11
)
SELECT
  search_type
  , COUNT(1) AS count
  , SUM(has_session_click) AS detail
  , AVG(has_session_click) AS ctr
  , SUM(CASE WHEN has_session_click = 1 THEN has_session_conversion END) AS conversion
  , AVG(CASE WHEN has_session_click = 1 THEN has_session_conversion END) AS cvr
FROM
  activity_log_with_session_click_conversion_flag
WHERE
  -- 검색 로그만 추출
  path = '/search_list'
-- 검색 조건으로 집약
GROUP BY
  search_type
ORDER BY
  count DESC
;
```

- 위의 쿼리로는 여러 개의 검색이 모두 카운트 됨
- `LAG` 함수를 이용하여 클릭 플래그를 직전 페이지에 한정하는 쿼리

```sql
WITH
activity_log_with_session_click_conversion_flag AS (
  SELECT
    session
    , stamp
    , path
    , search_type
    -- 상세 페이지의 직전 접근에 플래그 추가
    , CASE
        WHEN LAG(path) OVER(PARTITION BY session ORDER BY stamp DESC) = '/detail'
          THEN 1
        ELSE 0
      END AS has_session_click
    -- 성과가 발생하는 페이지의 이전 접근에 플래그 추가
    , SIGN(
      SUM(CASE WHEN path ='/complete' THEN 1 ELSE 0 END)
        OVER(PARTITION BY session ORDER BY stamp DESC
          ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
      ) AS has_session_conversion
  FROM
    activity_log
)
SELECT
  session
  , stamp
  , path
  , search_type
  , has_session_click AS click
  , has_session_conversion AS cnv
FROM
  activity_log_with_session_click_conversion_flag
ORDER BY
  session, stamp
;
```

### 15-6 폴아웃 리포트를 사용해 사용자 회유를 가시화하기

- 사용자 흐름 중 어디에서 이탈이 많은지, 이동이 이루어지는지를 파악해서 CVR을 향상시킬 수 있음
  - 폴스루(Fall Through): 어떤 지점으로 옮겨감
  - 폴아웃(Fall Out): 어떤 지점에서 이탈
- 폴아웃 단계 순서를 접근 로그와 결합하는 쿼리

```sql
WITH
mst_fallout_step AS (
  -- 폴아웃 단계와 경로의 마스터 테이블
            SELECT 1 AS step, '/' AS path
  UNION ALL SELECT 2 AS step, '/search_list' AS path
  UNION ALL SELECT 3 AS step, '/detail' AS path
  UNION ALL SELECT 4 AS step, '/input' AS path
  UNION ALL SELECT 5 AS step, '/complete' AS path
)
, activity_log_with_fallout_step AS (
  SELECT
    l.session
    , m.step
    , m.path
    -- 첫 접근과 마지막 접근 시간 구하기
    , MAX(l.stamp) AS max_stamp
    , MIN(l.stamp) AS min_stamp
  FROM
    mst_fallout_step AS m
    JOIN
      activity_log As l
      ON m.path = l.path
  GROUP BY
    -- 세션별로 단계 순서와 경로를 사용해 집약
    l.session, m.step, m.path
)
, activity_log_with_mod_fallout_step AS (
  SELECT
    session
    , step
    , path
    , max_stamp
    -- 직전 단계에서의 첫 접근 시간 구하기
    , LAG(min_stamp)
        OVER(PARTITION BY session ORDER BY step)
        -- sparkSQL의 경우 LAG 함수에 프레임 지정 필요
        OVER(PARTITION BY session ORDER BY step
          ROWS BETWEEN 1 PRECEDING AND 1 PRECEDING)
      AS lag_min_stamp
    -- 세션에서의 단계 순서 최소값 구하기
    , MIN(step) OVER(PARTITION BY session) AS min_step
    -- 해당 단계 도달할때까지 걸린 단계 수 누계
    , COUNT(1)
      OVER(PARTITION BY session ORDER BY step
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
      AS cum_count
  FROM
    activity_log_with_fallout_step
)
SELECT
  *
FROM
  activity_log_with_mod_fallout_step
ORDER BY
  session, step
;
```

- 폴아웃 리포트에 필요한 로그를 압축하는 쿼리
  - session에 step=1인 URL(최상위 페이지)이 있으면 폴아웃 리포트 대상에서 제외
  - step=2 페이지(/search_list) 스킵 + step=3 페이지(/detail)에 직접 접근한 로그 등을 제외
  - step = 3 페이지에 접근한 후에 step=2로 돌아갈경우 제외

```sql
WITH
mst_fallout_step AS (
  -- CODE.15.14
)
, activity_log_with_fallout_step AS (
  -- CODE.15.14
)
, activity_log_with_mod_fallout_step AS (
  -- CODE.15.14
)
, fallout_log AS (
  -- 폴아웃 리포트에 사용할 로그만 추출
  SELECT
    session
    , step
    , path
  FROM
    activity_log_with_mod_fallout_step
  WHERE
    -- 세션에서 단계 순서가 1인지 확인하기
    min_step = 1
    -- 현재 단계 순서가 해당 단계의 도달할 떄까지 누계 단계 수와 같은지 확인
    AND step = cum_count
    -- 직전 단계의 첫 접근 시간이 NULL 또는 현재 시간의 최종 접근 시간보다 이전인지 확인
    AND (lag_min_stamp IS NULL OR max_stamp >= lag_min_stamp)
)
SELECT
  *
FROM
  fallout_log
ORDER BY
  session, step
;
```

- 폴아웃 리포트를 출력하는 쿼리
  - 스텝 순서와 URL로 집약하고 접근수와 페이지 이동률을 집계

```sql
WITH
mst_fallout_step AS (
  -- CODE.15.4
)
, activity_log_with_fallout_step AS (
  -- CODE.15.4
)
, fallout_log AS(
  -- CODE.15.5
)
SELECT
  step
  , path
  , COUNT(1) AS count
  -- 단계 순서 = 1 URL부터의 이동률
  , 100.0 * COUNT(1)
    / FIRST_VALUE(COUNT(1))
      OVER(ORDER BY step ASC ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING)
    AS first_trans_rate
  -- 직전 단계까지의 이동률
  , 100.0 * COUNT(1)
    / LAG(COUNT(1)) OVER(ORDER BY step ASC)
    -- sparkSQL의 경우 LAG함수에 프레임 지정 필요
    / LAG(COUNT(1)) OVER(ORDER BY step ASC ROWS BETWEEN 1 PRECEDING AND 1 PRECEDING)
    AS step_trans_rate
FROM
  fallout_log
GROUP BY
  step, path
ORDER BY
  step
;
```


---

- 사용자 흐름 그래프 추출
  - 사용자가 제공자가 의도한 대로 서비스를 사용하는지 파악 가능
- 시작지점으로 삼을 페이지를 결정해야 함
  - 최상위 페이지에서 어떤 식으로 유도
    - 검색 사용 여부
    - 서비스 소개 페이지 보는지
  - 상세 화면 전후에서 어떤 행동을 하는가 
    - 추천 사용 여부
    - 검색 결과에서 상세페이지 가는지
    - 상세 화면 후 장바구니를 담는지
    - 상세 화면 후 다른 상품 상세 화면으로 이동하는지
- /detail 페이지 이후의 사용자 흐름을 집계하는 쿼리

```sql
WITH
activity_log_with_lead_path AS (
  SELECT
    session
    , stamp
    , path AS path0
    -- 곧바로 접근한 경로 추출하기
    , LEAD(path, 1) OVER(PARTITION BY session ORDER BY stamp ASC) AS path1
    -- 이어서 접근한 경로 추출하기
    , LEAD(path, 2) OVER(PARTITION BY session ORDER BY stamp ASC) AS path2
  FROM
    activity_log
)
, raw_user_flow AS (
  SELECT
    path0
    -- 시작 지점 경로로의 접근 수
    , SUM(COUNT(1)) OVER() AS count0
    -- 곧바로 접근한 경로 (존재하지 않는 경우 문자열 NULL)
    , COALESCE(path1, 'NULL') AS path1
    -- 곧바로 접근한 경로로의 접근 수
    , SUM(COUNT(1)) OVER(PARTITION BY path0, path1) AS count1
    -- 이어서 접근한 경로(존재하지 않는 경우 문자열로 `NULL` 지정)
    , COALESCE(path2, 'NULL') AS path2
    -- 이어서 접근한 경로로의 접근 수
    , COUNT(1) AS count2
  FROM
    activity_log_with_lead_path
  WHERE
    -- 상세 페이지를 시작 지점으로 두기
    path0 = '/detail'
  GROUP BY
    path0, path1, path2
)
SELECT
  path0
  , count0
  , path1
  , count1
  , 100.0 * count1 / count0 AS rate1
  , path2
  , count2
  , 100.0 * count2 / count1 AS rate2
FROM
  raw_user_flow
ORDER BY
  count1 DESC, count2 DESC
;
```

- 중복된 데이터가 많이나와서 `LAG`함수를 사용해서 바로 위의 레코드와 같은 값을 가졌을 때 출력하지 않도록 쿼리

```sql
WITH
activity_log_with_lead_path AS (
  -- CODE.15.17
)
, raw_user_flow AS (
  -- CODE.15.17
)
SELECT
  CASE
    WHEN
      COALESCE(
        -- 바로 위의 레코드가 가진 path0 추출(존재 하지 않는 경우 NOT FOUND)
        LAG(path0)
          OVER(ORDER BY count1 DESC, count2 DESC)
          , 'NOT FOUND'
      ) <> path0
    THEN path0
  END AS path0
  , CASE
      WHEN
        COALESCE(
          LAG(path0)
          OVER(ORDER BY count1 DESC, count2 DESC)
          , 'NOT FOUND'
        ) AS path0
      THEN count0
    END AS count0
  , CASE
      WHEN
        COALESCE(
          -- 바로 위의 레코드가 가진 여러 값을 추출할 수 있게, 문자열 결합 후 추출
          -- PostgreSQL, Redshift의 경우 || 연산자 사용
          -- Hive, BigQuery, SparkSQL의 경우 concat 함수 사용
          LAG(path0 || path1)
            OVER(ORDER BY count1 DESC, count2 DESC)
          , 'NOT FOUND'
        ) <> (path0 || path1)
      THEN path1
    END AS page1
  , CASE
      WHEN
        COALESCE(
          LAG(path0 || path1)
            OVER(ORDER BY count1 DESC, count2 DESC)
          , 'NOT FOUND'
        ) <> (path0 || path1)
      THEN count1
    END AS count1
  , CASE
      WHEN
        COALESCE(
          LAG(path0 || path1)
            OVER(ORDER BY count1 DESC, count2 DESC)
          , 'NOT FOUND') <> (path0 || path1)
      THEN 100.0 * count1 / count0
    END AS rate1
  , CASE
      WHEN
        COALESCE(
          LAG(path0 || path1 || path2)
            OVER(ORDER BY count1 DESC, count2 DESC)
          , 'NOT FOUND') <> (path0 || path1 || path2)
      THEN path2
    END AS page2
  , CASE
      WHEN
        COALESCE(
          LAG(path0 || path1 || path2)
            OVER(ORDER BY count1 DESC, count2 DESC)
          , 'NOT FOUND') <> (path0 || path1 || path2)
      THEN count2
    END AS count2
  , CASE
      WHEN
        COALESCE(
          LAG(path0 || path1 || path2)
            OVER(ORDER BY count1 DESC, count2 DESC)
          , 'NOT FOUND') <> (path0 || path1 || path2)
      THEN 100.0 * count2 / count1
    END AS rate2
  FROM
    raw_user_flow
  ORDER BY
    count1 DESC
    , count2 DESC
;
```

- 이전 페이지 집계하기
  - 다음페이지 집계했던거에서 `LEAD` 함수를 `LAG` 함수로 변경하면 됨

```sql
WITH
activity_log_with_lag_path AS (
  SELECT
    session
    , stamp
    , path AS path0
    -- 바로 전에 접근한 경로 추출하기(존재하지 않는 경우 문자열 'NULL'로 지정)
    , COALESCE(LAG(path, 1) OVER(PARTITION BY session ORDER BY stamp ASC), 'NULL') AS path1
    -- 그 전에 접근한 페이지 추출하기(존재하지 않는 경우 문자열 'NULL'로 지정)
    , COALESCE(LAG(path, 2) OVER(PARTITION BY session ORDER BY stamp ASC), 'NULL') AS path2
  FROM
    activity_log
)
, raw_user_flow AS (
  SELECT
    path0
    -- 시작 지점 경로로의 접근 수
    , SUM(COUNT(1)) OVER() AS count0
    , path1
    -- 바로 전의 경로로의 접근 수
    , SUM(COUNT(1)) OVER(PARTITION BY path0, path1) AS count1
    , path2
    -- 그 전에 접근한 경로로의 접근 수
    , COUNT(1) AS count2
  FROM
    activity_log_with_lag_path
  WHERE
    -- 상세 페이지를 시작 지점으로 두기
    path0 = '/detail'
  GROUP BY
    path0, path1, path2
)
SELECT
  path2
  , count2
  , 100.0 * count2 / count1 AS rate2
  , path1
  , count1s
  , 100.0 * count1 / count0 AS rate1
  , path0
  , count0
FROM
  raw_user_flow
ORDER BY
  count1 DESC
  , count2 DESC
;
```

### 15-8 페이지 완독률 집계하기

- 사용자가 페이지를 끝까지 읽었는지 파악 필요
- 완독률을 집계하는 쿼리

```sql
SELECT
  url
  , action
  , COUNT(1)
    / SUM(CASE WHEN action='view' THEN COUNT(1) ELSE 0 END)
        OVER(PARTITION BY url)
    AS action_per_view
FROM read_log
GROUP BY
  url, action
ORDER BY
  url, count DESC
;
```

### 15-9 사용자 행동 전체를 시각화하기

- 한눈에 볼 수 있도록 조감도를 작성하여 서비스 개선 포인트를 잡아보기

## 16강 입력 양식 최적화하기

---

- 자료 청구, 구매 양식 등 폼을 통해 사용자가 
- EFO = Entry Form Optimization을 진행해야 함
  - 필수 입력과 선택 입력 명확히 구분 후 필수 항목 상단에 위치시킴
  - 오류 발생 빈도 줄이기
  - 쉽게 입력할 수 있도록 항목 줄이거나 자동완성기능 추가
  - 이탈할 만한 요소 제거

### 16-1 오류율 집계하기

- 확인 화면에서의 오류율을 집계하는 쿼리
  - 오류율이 높으면 오류 통지 방법에 문제가 있을 가능성이 높음

```sql
SELECT
  COUNT(*) AS confirm_count
  , SUM(CASE WHEN status='error' THEN 1 ELSE 0 END) AS error_count
  , AVG(CASE WHEN status='error' THEN 1.0 ELSE 0.0 END) AS error_rate
  , SUM(CASE WHEN status='error' THEN 1.0 ELSE 0.0 END) / COUNT(DISTINCT session) AS error_per_user
FROM form_log
WHERE
  -- 확인 화면 페이지 판정
  path = '/regist/confirm'
;
```

### 16-2 입력-확인-완료까지의 이동률 집계하기

- 입력 양식의 폴아웃 리포트

```sql
WITH
mst_fallout_step AS (
  -- /regist 입력 양식의 폴아웃 단계와 경로 마스터
            SELECT 1 AS step, '/regist/input'     AS path
  UNION ALL SELECT 2 AS step, '/regist/confirm'   AS path
  UNION ALL SELECT 3 AS step, '/regist/complete'  AS path
)
, form_log_with_fallout_step AS (
  SELECT
    l.session
    , m.step
    , m.path
    -- 특정 단계 경로의 처음/마지막 접근 시간 구하기
    , MAX(l.stamp) AS max_stamp
    , MIN(l.stamp) AS min_stamp
  FROM
    mst_fallout_step AS m
    JOIN
      form_log AS l
      ON m.path = l.path
  -- 확인 화면의 상태가 오류인 것만 추출하기
  WHERE status = ''
  -- 세션별로 단계 순서와 경로 집약하기
  GROUP BY l.session, m.step, m.path
)
, form_log_with_mod_fallout_step AS (
  SELECT
    session
    , step
    , path
    , max_stamp
    -- 직전 단계 경로의 첫 접근 시간
    , LAG(min_stamp)
        OVER(PARTITION BY session ORDER BY step)
        -- sparkSQL의 경우 LAG 함수에 프레임 지정 필요
        OVER(PARTITION BY session ORDER BY step
          ROWS BETWEEN 1 PRECEDING AND 1 PRECEDING)
      AS lag_min_stamp
    -- 세션 내부에서 단계 순서 최솟값
    , MIN(step) OVER(PARTITION BY session) AS min_Step
    -- 해당 단계에 도달할 때까지의 누계 단계 수
    , COUNT(1)
      OVER(PARTITION BY session ORDER BY step
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT_ROW)
      AS cum_count
  FROM form_log_with_fallout_step
)
, fallout_log AS (
  -- 폴아웃 리포트에 필요한 정보 추출하기
  SELECT
    session
    , step
    , path
  FROM
    form_log_with_mod_fallout_step
  WHERE
    -- 세션 내부에서 단계 순서가 1인 URL에 접근하는 경우
    min_step = 1
    -- 현재 단계 순서가 해당 단계에 도착할 때까지의 누계 단계 수와 같은 경우
    AND step = cum_count
    -- 직전 단계의 첫 접근 시간이 NULL 또는 현재 단계의 최종 접근 시간보다 앞인 경우
    AND (lag_min_stemp IS NULL OR max_stamp >= lag_min_stamp)
)
SELECT
  step
  , path
  , COUNT(1) AS count
  -- '단계 순서 = 1'인 URL로부터의 이동률
  , 100.0 * COUNT(1)
    / FIRST_VALUE(COUNT(1))
      OVER(ORDER BY step ASC ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING)
    AS first_trans_rate
  -- 직전 단계로부터의 이동률
  , 100.0 * COUNT(1)
    / LAG(COUNT(1)) OVER(ORDER BY step ASC)
    -- sparkSQL의 경우 LAG 함수에 프레임 지정 필요
    / LAG(COUNT(1)) OVER(ORDER BY step ASC ROWS BETWEEN 1 PRECEDING AND 1 PRECEDING)
    AS step_trans_rate
FROM
  fallout_log
GROUP BY
  step, path
ORDER BY
  step
;  
```

### 16-3 입력 양식 직귀율 계산하기

- 직귀율이 높다면, 입력 폼의 문제가 있거나 입력을 남발하는 인상을 줄 가능성이 높음
- 입력 양식 직귀율을 계산하는 쿼리
  - 세션 별로 

```sql
WITH
form_with_progress_flag AS (
  SELECT
    -- PostgreSQL, Hive, Redshift, SparkSQL의 경우 substring으로 날짜 부분 추출
    substring(stamp, 1, 10) AS dt
    -- PostgreSQL, Hive, BigQuery, SparkSQL의 경우 substr 사용
    substr(stamp, 1, 10) AS dt
    
    , session
    -- 입력 화면으로부터의 방문 플래그 계산
    , SIGN(
        SUM(CASE WHEN path IN ('/regist/input') THEN 1 ELSE 0 END)
    ) AS has_input
    -- 입력 확인 화면 또는 완료 화면으로의 방문 플래그 계산하기
    , SIGN(
      SUM(CASE WHEN path IN('/regsit/confirm', '/regist/complete') THEN 1 ELSE 0 END)
    ) AS has_progress
  FROM form_log
  GROUP BY
    -- PostgreSQL, Redshift, BigQuery의 경우
    -- SELECT 구문에서 정의한 별칭을 GROUP BY에 지정할 수 있음
    dt, session
    -- PostgreSQL, Hive, Redshift, SparkSQL의 경우
    -- SELECT 구문에서 별칭을 지정하기 이전의 식을 GROUP BY에 지정할 수 있음
    substring(stamp, 1, 10), session
)
SELECT
  dt
  , COUNT(1) AS input_count
  , SUM(CASE WHEN has_progress = 0 THEN 1 ELSE 0 END) AS bounce_count
  , 100.0 * AVG(CASE WHEN has_progrss = 0 THEN 1 ELSE 0 END) AS bounce_rate
FROM
  form_with_progress_flag
WHERE
  -- 입력 화면에 방문했던 세션만 추출하기
  has_input = 1
GROUP BY
  dt
;
```

### 16-4 오류가 발생하는 항목과 내용 집계하기

- 각 입력 양식의 오류 발생 장소와 원인을 집계하는 쿼리

```sql
SELECT
  form
  , field
  , error_type
  , COUNT(1) AS count
  , 100.0 * COUNT(1) / SUM(COUNT(1)) OVER(PARTITION BY form) AS share
FROM
  form_error_log
GROUP BY
  form, field, error_type
ORDER BY
  form, count DESC
```
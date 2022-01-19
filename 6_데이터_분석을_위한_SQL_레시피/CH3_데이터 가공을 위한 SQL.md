# CH3_데이터 가공을 위한 SQL

---

## 5강 하나의 값 조작하기
- 데이터를 가공해야 하는 이유
  - 로그 데이터 같은 경우는 한 줄의 row로 모든 내용이 기록되어 있음
  - 데이터 형식의 불일치 ex) NULL 오류

#### 5-1 코드 값을 레이블로 변경하기
- 코드를 레이블로 변경하는 쿼리
```SQL
SELECT
  user_id,
  CASE WHEN register_device = 1 THEN '데스크톱'
  CASE WHEN register_device = 2 THEN '스마트폰'
  ELSE '' -- default 값
  END AS device_name
FROM mst_users;
```

#### 5-2 URL에서 요소 추출하기
- bigquery와 hive는 URL관련 함수들을 제공하지만, 다른 DW들은 제공하지 않아 정규표현식으로 사용해야함
- HOST 추출 쿼리
```SQL
SELECT
  substring(referrer from 'https?://([^/]*))' -- psql
  regex_replace(regexp_substr(referrer, 'https?://([^/]*'), 'https?://', '') -- redshift
  parse_url(referrer, 'HOST' AS referrer_host) -- Hive sparkSQL
  host(referrer) -- Bigquery
FROM access_log;
```
- URL 경로와 GET 매개변수에 있는 특정 키 값을 추출하는 쿼리
```SQL
SELECT
  stamp
  , url

  -- PostgreSQL의 경우 substring 함수와 정규 표현식 사용
  , substring(url from '//[^/]+([^?#]+)') AS path
  , substring(url from 'id=([^&]*)') AS id

  -- Redshift의 경우 regexp_substr 함수와 regexp_replace 함수를 조합하여 사용
  , regexp_replace(regexp_substr(url, '//[^/]+[^?#]+'), '//[^/]+', '') AS path
  , regexp_replace(regexp_substr(url, 'id=[^&]*'), 'id=', '') AS id

  -- BigQuery의 경우 정규 표현식과 regexp_extract 함수 사용
  , regexp_extract(url, '//[^/]+([^&#]+)') AS path
  , regexp_extract(url, 'id=([^&]*)') AS id

  -- Hive, SparkSQL의 경우 parse_url 함수로 url 경로 / 쿼리 매개변수 추출
  , parse_url(url, 'PATH') AS path
  , parse_url(url, 'QUERY', 'id') As id

FROM access_log;
```

#### 5-3 문자열을 배열로 분해하기
- URL 경로를 슬래시로 분할해서 계층을 추출하는 쿼리
```SQL
SELECT
  stamp
  , url

  -- PostgreSQL의 경우, split_part로 n번째 요소 추출
  , split_part(substring(url from '//[^/]+([^?#]+)'), '/', 2) AS path1
  , split_part(substring(url from '//[^/]+([^?#]+)'), '/', 3) AS path2

  -- Redshift도 split_part로 n번째 요소 추출
  , split_part(regexp_replace(regexp_substr(url, '//[^/]+[^?#]+'), '//[^/]+', ''), '/', 2) AS path1
  , split_part(regexp_replace(regexp_substr(url, '//[^/]+[^?#]+'), '//[^/]+', ''), '/', 3) AS path2

  -- BigQuery의 경우 split 함수를 사용하여 배열로 자름(별도 인덱스 지정 필요)
  , split(regexp_extract(url, '//[^/]+([^&#]+)'), '/')[SAFE_ORDINAL(2)] AS path1
  , split(regexp_extract(url, '//[^/]+([^&#]+)'), '/')[SAFE_ORDINAL(3)] AS path2

  -- Hive, SparkSQL도 split 함수를 사용하여 배열로 자름
  , split(parse_url(url, 'PATH') , '/')[1] AS path1
  , split(parse_url(url, 'PATH') , '/')[2] AS path2

FROM access_log;
```

#### 5-4 날짜와 타임스탬프 다루기
- 현재 날짜와 타임스탬프 출력하는 쿼리
```SQL
SELECT
  -- PostgreSQL, Hive, BigQuery의 경우
  CURRENT_DATE AS dt
  , CURRENT_TIMESTAMP AS stamp

  -- Hive, BigQuery, SparkSQL
  CURRENT_DATE() AS dt
  , CURRENT_TIMESTAMP() AS stamp

  -- Redshift, 현재 날짜는 CURRENT_DATE, 현재 타임 스탬프는 GETDATE() 사용
  CURRENT_DATE AS dt
  , GETDATE() AS stamp

  -- PostgreSQL, CURRENT_TIMESTAMP, timezone이 적용된 타임스탬프
  -- 타임존을 적용하고 싶지 않을 때, LOCALTIMESTAMP 사용
  , LOCALTIMESTAMP AS stamp
  ;
```
- 문자열을 날짜 자료형, 타임스탬프 자료형으로 변환하는 쿼리
```SQL
-- 문자열을 날짜/타임스탬프로 변환

SELECT
-- PostgreSQL, Hive, Redshift, Bigquery, SparkSQL 모두
-- `CAST(value AS type)` 사용
CAST('2016-01-30' AS date) AS dt
, CAST('2016-01-30 12:00:00' AS timestamp) AS stamp

-- Hive, Bigquery, `type(value)` 사용
date('2016-01-30') AS dt
, timestamp('2016-01-30 12:00:00') AS stamp

-- PostgreSQL, Hive, Redshift, BigQuery, SparkSQL, `type value` 사용
-- 단, value는 상수이므로, 컬럼 이름 지정 불가능
date '2016-01-30' AS dt
, timestamp '2016-01-30 12:00:00' AS stamp

-- PostgreSQL, Redshift, `value::type` 사용
'2016-01-30'::date AS dt
, '2016-01-30 12:00:00'::timestamp AS stamp
```
- **타임스탬프 자료형의 데이터**에서 연, 월, 일 등을 추출하는 쿼리
```SQL
SELECT
  stamp
  -- PostgreSQL, Redshift, BigQuery, EXTRACT 함수 사용
  , EXTRACT(YEAR  FROM stamp) AS year
  , EXTRACT(MONTH FROM stamp) AS month
  , EXTRACT(DAY   FROM stamp) AS day
  , EXTRACT(HOUR  FROM stamp) AS hour

  -- Hive, SparkSQL
  , YEAR(stamp) AS year
  , MONTH(stamp) AS month
  , DAY(stamp) AS day
  , HOUR(stamp) AS hour
FROM
  (SELECT CAST('2020-01-16 22:22:00' AS timestamp) AS stamp) AS t
```
- **타임스탬프를 나타내는 문자열**에서 연, 월, 일 등을 추출하는 쿼리
```SQL
SELECT
  stamp

  -- PostgreSQL, Hive, Redshift, SparkSQL, substring 함수 사용
  , substring(stamp, 1, 4) AS year
  , substring(stamp, 6, 2) AS month
  , substring(stamp, 9, 2) AS day
  , substring(stamp, 12, 2) AS hour
  -- 연, 월을 함께 추출
  , substring(stamp, 1, 7) AS year_month

  --- PostgreSQL, Hive, BigQuery, SparkSQL, substr 함수 사용
  , substr(stamp, 1, 4) AS year
  , substr(stamp, 6, 2) AS month
  , substr(stamp, 9, 2) AS day
  , substr(stamp, 12, 2) AS hour
  , substr(stamp, 1, 7) AS year_month
FROM
  -- PostgreSQL, Redshift의 경우 문자열 자료형(text)
  (SELECT CAST('2020-01-16 22:26:00' AS text) AS stamp) AS t

  -- Hive, BigQuery, SparkSQL의 경우 문자열 자료형(string)
  (SELECT CAST('2020-01-16 22:26:00' AS string) AS stamp) AS t
```

#### 5-5 결손 값을 디폴트값으로 대치하기
- NULL과 함께하는 연산은 무조건 NULL로 됨
-구매액에서 할인 쿠폰 값을 제외한 매출 금액을 구하는 쿼리
```SQL
SELECT
  purchase_id
  , amount
  , coupon
  , amount - coupon AS discount_amount1
  , amount - COALESCE(coupon, 0) AS discount_amount2
FROM
  purchase_log_with_coupon
```

---

## 6강 여러 개의 값에 대한 조작
- 여러 값을 집약 및 비교하여 다양한 관점에서의 데이터를 바라봄
#### 6-1 문자열 연결하기
-문자열을 연결하는 쿼리
```SQL
SELECT
  user_id
  
  -- PostgreSQL, Hive, Redshift, BigQuery, SparkSQL 모두 CONCAT 함수 사용 가능
  -- 다만 redshift의 경우는 매개변수를 2개밖에 못받는다
  , CONCAT(pref_name, city_name) AS pref_city
  
  -- PostgreSQL, Redshift의 경우 || 연산자 사용 가능
  , pref_name || city_name AS pref_City
FROM
  mst_user_location
```

#### 6-2 여러 개의 값 비교하기
- 쿼터별 매출액을 비교하기
- q1, q2 컬럼을 비교하는 쿼리
```SQL
SELECT
  year
  ,q1
  ,q2
  
  -- q1과 q2의 매출변화 평가
  , CASE
    WHEN q1 < q2 THEN '+'
    WHEN q1 = q2 THEN ' '
    ELSE '-'
  END AS judge_q1_q2
  
  -- q1, q2의 매출액 차이 계산
  , q2 - q1 AS diff_q2_q1
  
  -- q1과 q2의 매출 변화를 1, 0, -1로 표현
  , SIGN(q2 - q1) AS sign_q2_q1
FROM
  quarterly_Sales
ORDER BY
  year
```
- 연간 최대/최소 4분기 매출을 찾는 쿼리
```SQL
SELECT
  year
  
  -- q1 ~ q4의 최대 매출 구하기
  , greatest(q1, q2, q3, q4) AS greatest_sales
  
  -- q1 ~ q4의 최소 매출 구하기
  , least(q1, q2, q3, q4) AS least_sales
FROM
  quarterly_sales
ORDER BY
  year
```
- COALESCE를 사용해 NULL을 0으로 변환하고 4분기 평균 매출을 구하는 쿼리
```SQL
SELECT
  year
  , (COALESCE(q1, 0) + COALESCE(q2, 0) + COALESCE(q3, 0) + COALESCE(q4, 0)) / 4 AS average
FROM
  quarterly_sales
ORDER BY
  year
```
- NULL이 아닌 컬럼만을 사용해서 평균값을 구하는 쿼리
```SQL
SELECT
  year
, (COALESCE(q1, 0) + COALESCE(q2, 0) + COALESCE(q3, 0) + COALESCE(q4, 0))
/ (SIGN(COALESCE(q1, 0)) + SIGN(COALESCE(q2, 0)) + SIGN(COALESCE(q3, 0)) + SIGN(COALESCE(q4, 0))) AS average
FROM
  quarterly_sales
ORDER BY
  year
```

---

#### 6-3 2개의 값 비율 계산하기
- 광고 통계 정보를 통한 CTR(Click Through Rate) 클릭 노출수 계산
- 정수 자료형의 데이터를 나누는 쿼리
```SQL
SELECT
  dt
  , ad_id

  -- Hive, Redshift, Bigquery, SparkSQL
  -- 정수를 나눌때, 자동으로 실수형 변환
  , clicks / impressions AS ctr
  
  -- PostgreSQL, 정수 나눌경우, 소수점이 잘리므로, 명시적으로 자료형 변환
  , CAST(clicks AS double precision) / impressions AS ctr
  
  -- 실수를 상수로 앞에 두고 계산하면, 암묵적으로 자료형 변환
  , 100.0 * clicks / impressions AS ctr_as_percent
FROM
  advertising_stats
WHERE
  dt='2017-04-01'
ORDER BY
  dt, ad_id
```
- 0으로 나누는 것을 피해 CTR을 계산하는 쿼리
```SQL
SELECT
  dt
  , ad_id
  
  -- CASE 식으로 분모가 0일 경우를 분기, 0으로 나누지 않도록 함
  , CASE
    WHEN impressions > 0 THEN 100.0 * clicks / impressions
  END AS ctr_as_percent_by_case
  
  -- 분모가 0이라면 NULL로 변환하여, 0으로 나누지 않도록 함
  -- PostgreSQL, Redshift, BigQuery, SparkSQL의 경우 NULLIF 함수 사용
  , 100.0 * clicks / NULLIF(impressions, 0) AS ctr_as_percent_by_null
  
  -- Hive의 경우 NULLIF 대신 CASE식 사용하기
  , 100*0 * clicks / 
  CASE WHEN impressions = 0 THEN NULL ELSE impressions END
FROM
  advertising_stats
ORDER_BY
  dt, ad_id
```

#### 6-4 두 값의 거리 계산하기
- 물리적 거리뿐만 아니라 평균점수와 떨어져 있는정도, 매출의 차이 등을 `거리`라는 개념을 사용
- 일차원 데이터의 절댓값과 제곱 평균 제곱근을 계산하는 쿼리
```SQL
SELECT
  ABS(x1 -x2) AS abs
  , sqrt(power(x1 - x2, 2)) AS rms
FROM location_1d
```
- 2차원 테이블에 대해 평균 제곱근(유클리드 거리)을 구하는 쿼리
```SQL
SELECT
  sqrt(power(x1 - x2, 2) + power(y1 - y2)) AS dist
  
  -- PostgreSQL, point 자료형과 거리 연산자 (<->) 사용
  , point(x1, y1) <-> point(x2, y2) AS dist
FROM
  location_2d
;
```

#### 6-5 날짜/시간 계산하기
- 미래 또는 과거의 날짜/시간을 계산하는 쿼리
```SQL
SELECT
  user_id

  -- PostgreSQL, interval 자료형의 데이터에 사칙 연산 적용
  , register_stamp::timestamp AS register_stamp
  , register_stamp::timestamp + '1 hour'::interval AS after_1_hour
  , register_stamp::timestamp - '30 minutes'::interval AS berfore_30_minutes

  , register_stamp::date AS register_date
  , (register_stamp::date + '1 day'::interval)::date AS after_1_day
  , (register_stamp::date - '1 month'::interval)::date AS before_1_month

  -- Redshift, dateadd 함수 사용
  , register_stamp::timestamp AS register_stamp
  , dateadd(hour, 1 ,register_stamp::timestamp) AS after_1_hour
  , dateadd(monute, -30, register_stamp::timestamp) AS before_30_minutes

  , register_stamp::date register_date
  , dateadd(day, 1, register_stamp::date) AS after_1_day
  , dateadd(month, -1, register_stamp:date) AS before_1_month

  -- BigQuery, timestamp_add/sub, date_add/sub 함수 사용
  , timestamp(register_stamp) AS register_stamp
  , timestamp_add(timestamp(register_stamp), interval 1 hour) AS after_1_hour
  , timestamp_add(timestamp(register_stamp), interval 30 minute) AS before_30_minutes

  -- 타임스탬프 문자열 기반으로 직접 날짜 계산을 할 수 없으므로
  -- 타임 스탬프 자료형 -> 날짜/시간 자료형 변환 뒤 계산
  , date(timestamp(register_stamp)) AS register_date
  , date_add(date(timestamp(register_stamp)), interval 1 day) AS after_1_day
  , date_sub(date(timestamp(register_stamp)), interval 1 month) AS before_1_month

  -- Hive, SparkSQL, 날짜/시각 계산 함수 제공 x
  -- unixtime으로 변환 후, 초단위로 계산 적용뒤 다시 타임스탬프로 변환
  , CAST(register_stamp AS timestamp) AS register_stamp
  , from_unixtime(unix_timestamp(register_stamp) + 60 * 60) AS after_1_hour
  , from_unixtime(unix_timestamp(register_stamp) - 30 * 60) AS before_30_minutes

  --- 타임스탬프 문자열을 날짜 변환시, to_date 함수 사용
  -- 단, hive 2.1.0 이전 버전의 경우, 문자열 자료형 리턴
  , to_date(register_stamp) AS register_date

  -- day/month 계산 시, date_add / date_months 함수 사용
  -- 단, year 계산 함수는 제공되지 않음
  , date_add(to_date(regsiter_stamp), 1) AS after_1_day
  , add_months(to_date(register_stamp), -1) AS before_1_month
FROM mst_users_with_dates
```

- 두 날짜의 차이를 계산하는 쿼리
```SQL
SELECT
  user_id

  -- PostgreSQL, Redshift, 날짜 자료형 끼리 연산 가능
  , CURRENT_DATE as today
  , register_stamp::date ADS register_date
  , CURRENT_DATE - register_stamp::date AS diff_days

  -- BigQuery의 경우 date_diff 함수 사용
  , CURRENT_DATE as today
  , date(timestamp(register_stamp)) AS register_date
  , date_diff(CURRENT_DATE, date(timestamp(register_stamp)), day) AS diff_Days

  -- Hive, SparkSQL의 경우 datediff 함수 사용
  , CURRENT_DATE() as today
  , to_date(register_stamp) AS register_date
  , datediff(CURRENT_DATE(), to_date(register_stamp)) AS diff_days
FROM mst_users_with_dates
```

- 사용자의 생년월일로부터 age 함수를 사용해 나이를 계산하는 쿼리
```SQL
SELECT
  user_id

  -- PostgreSQL, age 함수와 EXTRACT 함수를 이용하여 나이 집계
  , CURRENT_DATE AS today
  , regsiter_stamp::date AS register_date
  , birth_date::date AS birth_date
  , EXTRACT(YEAR FROM age(birth_date::date)) AS current_age
  , EXTRACT(YEAR FROM age(register_stamp::date, birth_date::date)) AS reguster)age
FROM mst_users_with_dates
```

- 연 부분 차이를 통해 나이를 계산하는 쿼리
  - 단순 연도 계산으로 해당 연이 생년월일을 넘었는지는 파악 못함
```SQL
SELECT
  user_id

  -- Redshift, datediff 함수로 year을 지정하더라도, 연 부분 차이는 계산 불가
  , CURRENT_DATE AS today
  , register_stamp::date AS register_date
  , birth_date::date AS birth_date
  , datediff(year, birth_date::date, CURRENT_DATE)
  , datediff(year, birth_date::date, register_stamp::date)

  -- BigQuery, date_diff 함수로 year 지정시에도, 연 부분 차이 계산 불가
  , CURRENT_DATE AS today
  , date(timestamp(register_stamp)) AS register_Date
  , date(timestamp(birth_date)) AS birth_date
  , date_diff(CURRENT_DATE, date(timestamp(birth_date)), year) AS current_age
  , date_diff(date(timestamp(register_stamp)), date(timestamp(birth_date)), year) AS register_age
FROM mst_users_with_dates
;
```
- 등록 시점과 현재 시점의 나이를 문자열로 계산하는 쿼리
```SQL
SELECT
  user_id
  , substring(register_stamp, 1, 10) AS register_date
  , birth_date

  -- 등록 시점의 나이 계산
  , floor(
    ( CAST(replace(substring(register_stamp, 1, 10), '-'. '') AS integer)
      - CAST(replace(birth_date, '-', '') AS integer)
      ) / 10000
  ) AS register_age

  -- 현재 시점의 나이 계산
  , floor (
    ( CAST(replace(CAST(CURRENT_DATE as text), '-', '') AS integer)
      - CAST(replace(birth_datey, '-', '') AS integer)
    ) / 10000
  ) AS current_age

  -- BigQuery, text -> string, integer -> int64
  ( CAST(replace(CAST(CURRENT_DATE AS string), '-', '') AS int64)
    - CAST(replace(birth_date, '-', '') AS int64)
  ) / 10000

  -- Hive, SparkSQL, replace -> regexp_replace, text -> string
  -- integer -> int
  -- SparkSQL, CURRENT_DATE -> CURRENT_DATE()
  ( CAST(regexp_replace(CAST(CURRENT_DATE() AS string), '-', '') AS int)
    - CAST(regexp_replace(birth_date, '-', '') AS int)
  ) / 10000
FROM mst_users_with_dates
;
```

#### 6-6 IP 주소 다루기
- IP주소를 비교하거나 동일한 네트워크인지 비교등에 활용
- psql inet 데이터 타입을 통한 비교
```SQL
SELECT
  CAST('127.0.0.1' AS inet) << CAST('127.0.0.0/8' AS inet) AS is_contained
```
- IP 주소에서 4개의 10진수 부분을 추출하는 쿼리
```SQL
SELECT
  ip

  -- PostgreSQL, Redshift의 경우 splift_part로 문자열 분해
  , CAST(split_part(ip, '.', 1) AS integer) AS ip_part_1
  , CAST(split_part(ip, '.', 2) AS integer) AS ip_part_2
  , CAST(split_part(ip, '.', 3) AS integer) AS ip_part_3
  , CAST(split_part(ip, '.', 4) AS integer) AS ip_part_4

  -- BigQuer, split 함수로 배열 분해, n번째 요소 추출
  , CAST(split(ip, '.')[SAFE_ORDINAL(1)] AS int64) AS ip_part_1
  , CAST(split(ip, '.')[SAFE_ORDINAL(2)] AS int64) AS ip_part_2
  , CAST(split(ip, '.')[SAFE_ORDINAL(3)] AS int64) AS ip_part_3
  , CAST(split(ip, '.')[SAFE_ORDINAL(4)] AS int64) AS ip_part_4

  -- Hive, SparkSQL, split 함수로 배열 분해, n번째 요소 추출
  -- 이때 '.'가 특수문자이므로, \로 escaping
  , CAST(split(ip, '\\.')[0] AS int) AS ip_part_1
  , CAST(split(ip, '\\.')[1] AS int) AS ip_part_2
  , CAST(split(ip, '\\.')[2] AS int) AS ip_part_3
  , CAST(split(ip, '\\.')[3] AS int) AS ip_part_4
FROM
  (SELECT '192.168.0.1' AS ip) AS t
  
  -- PostgreSQL의 경우 명시적 자료형 변환
  (SELECT CAST('192.168.0.1' AS text) AS ip) AS t
```
- IP 주소를 정수 자료형 표기로 변환하는 쿼리
```SQL
SELECT
  ip
  -- PostgreSQL, Redshift의 경우 splift_part로 문자열 분해
  , CAST(split_part(ip, '.', 1) AS integer) * 2^24
    + CAST(split_part(ip, '.', 2) AS integer) * 2^16
    + CAST(split_part(ip, '.', 3) AS integer) * 2^8
    + CAST(split_part(ip, '.', 4) AS integer) * 2^0
  AS ip_integer

  -- BigQuer, split 함수로 배열 분해, n번째 요소 추출
  , CAST(split(ip, '.')[SAFE_ORDINAL(1)] AS int64) * pow(2, 24)
    + CAST(split(ip, '.')[SAFE_ORDINAL(2)] AS int64) * pow(2, 16)
    + CAST(split(ip, '.')[SAFE_ORDINAL(3)] AS int64) * pow(2, 8)
    + CAST(split(ip, '.')[SAFE_ORDINAL(4)] AS int64) * pow(2, 0)
  AS ip_integer

  -- Hive, SparkSQL, split 함수로 배열 분해, nq번째 요소 추출
  -- 이때 '.'가 특수문자이므로, \로 escaping
  , CAST(split(ip, '\\.')[0] AS int) * pow(2, 24)
    + CAST(split(ip, '\\.')[1] AS int) * pow(2, 16)
    + CAST(split(ip, '\\.')[2] AS int) * pow(2, 8)
    + CAST(split(ip, '\\.')[3] AS int) * pow(2, 0)
  AS ip_integer
FROM
  (SELECT '192.168.0.1' AS ip) AS t
  
  -- PostgreSQL의 경우 명시적 자료형 변환
  (SELECT CAST('192.168.0.1' AS text) AS ip) AS t
```
- IP주소를 0으로 메우기
```SQL
SELECT
  ip

  -- PostgreSQL, Redshift, lpad 함수로 0 메우기
  , lpad(split_part(ip, '.', 1), 3, '0')
    || lpad(split_part(ip, '.', 2), 3, '0')
    || lpad(split_part(ip, '.', 3), 3, '0')
    || lpad(split_part(ip, '.', 4), 3, '0')
  AS ip_padding

  -- BigQuery, split 함수로 배열 분해, n번째 요소 추출
  , CONCAT(
    lpad(split(ip, '.')[SAFE_ORDINAL(1)], 3, '0')
    , lpad(split(ip, '.')[SAFE_ORDINAL(2)], 3, '0')
    , lpad(split(ip, '.')[SAFE_ORDINAL(3)], 3, '0')
    , lpad(split(ip, '.')[SAFE_ORDINAL(4)], 3, '0')
  ) AS ip_padding

  -- Hive, SparkSQL, split 함수로 배열 분해, n번째 요소 추출
  -- .이 특수문자 이므로 \로 escaping
  , CONCAT(
    lpad(split(ip, '\\.')[0], 3, '0')
    , lpad(split(ip, '\\.')[1], 3, '0')
    , lpad(split(ip, '\\.')[2], 3, '0')
    , lpad(split(ip, '\\.')[3], 3, '0')
  ) AS ip_padding
FROM
  (SELECT '192.168.0.1' AS ip) AS t

  -- PostgreSQL의 경우 명시적 자료형 변환
  (SELECT CAST('192.168.0.1' AS text) AS ip) AS t
```

## 7강 하나의 테이블에 대한 조작
- 레코드 하나하나가 아닌 대량의 데이터를 집계하여 전체의 특징을 파악(집약)

#### 7-1 그룹의 특징 잡기
- 상품 평가 테이블
- 집약 함수를 사용해서 테이블 전체의 특징량을 계산하는 쿼리
```SQL
SELECT
  COUNT(*) AS total_count
  , SUM(score) as sum
  , AVG(score) as avg
  , MAX(score) as max
  , MIN(score) as min
FROM
  review
;
```
- 그루핑한 데이터 특징량 계산
```SQL
SELECT
  user_id
  , COUNT(DISTINCT user_id) AS user_count
  , COUNT(DISTINCT product_id) AS product_count
  , SUM(score) as sum
  , AVG(score) as avg
  , MAX(score) as max
  , MIN(score) as min
FROM
  review
GROUP BY
  user_id
;
```
- 윈도 함수를 사용해 집약 함수의 결과와 원래 값을 동시에 다루는 쿼리
  - `OVER` 구문에 윈도 함수 지정
  - 매개변수 지정하지 않으면 테이블 전체, `PARTITION BY <컬럼>` 을 지정하면 컬럼 기반으로 그룹화하고 집약함수 지정
```SQL
SELECT
  user_id
  , product_id
  , score
  , AVG(score) OVER() AS avg_score -- 전체 평균 리뷰 점수
  , AVG(score) OVER(PARTITION BY user_id) AS user_avg_score -- 사용자 평균 리뷰 점수
  , score - AVG(score) OVER(PARTITION BY user_id) AS user_avg_score_diff -- 개별 리뷰 점수 - 사용자 평균 리뷰 점수
FROM
  review
;
```

#### 7-2 그룹 내부의 순서
- SQL은 기본적으로 순서가 없어 순위 작성이나 시간 순서를 다루는 것이 어려웠음
- 윈도함수가 등장하면서 이 과정이 쉬워짐
  - `RANK`는 같은 순위 허용(1, 2, 2, 4) `DENSE_RANK`는 (1, 2, 2, 3) 처럼 순위를 매김
  - `LAG`는 앞에 있는 행의 값 `LEAD`는 뒤의 있는 행의 값 추출
  - `OVER` 구문 내부에 `ORDER BY` 구문을 사용하여 데이터의 순서를 정의
- 윈도 함수의 ORDER BY 구문을 사용해 테이블 내부의 순서를 다루는 쿼리
```SQL
SELECT
  product_id
  , score
  -- 점수 순서로 유일한 순위
  , ROW_NUMBER()  OVER(ORDER BY score DESC) AS row
  -- 같은 순위 허용, 순위
  , RANK()        OVER(ORDER BY score DESC) AS rank
  -- 같은 순위 허용, 순위 숫자는 건너뜀
  , DENSE_RANK()  OVER(ORDER BY score DESC) AS dense_rank

  -- 현재 행보다 앞에 있는 행 추출
  , LAG(product_id)       OVER(ORDER BY score DESC) AS lag1
  , LAG(product_id, 2)    OVER(ORDER BY score DESC) AS lag2

  -- 현재 행보다 뒤에 있는 행 추출
  , LEAD(product_id)      OVER(ORDER BY score DESC) AS lead1
  , LEAD(product_id, 2)   OVER(ORDER BY score DESC) AS lead2
FROM popular_products
ORDER BY row
;
```
- `ROWS` 구문은 윈도 프레임 지정
- `FIRST_VALUE`, `LAST_VALUE`
- ORDER BY 구문과 집약 함수를 조합해서 계산하는 쿼리
```SQL
SELECT
  produt_id
  , score

  , ROW_NUMBER()  OVER(ORDER BY score DESC) AS row

  -- 순위 상위부터의 누계 구하기
  , SUM(score)
      OVER(ORDER BY score DESC
          ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
      AS cum_score
  
  -- 현재 행 기준 전/후 총 3개행의 평균
  , AVG(score)
      OVER(ORDER BY order DESC
          ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING)
      AS local_avg
  
  -- 순위가 높은 상품 ID(윈도 내부의 첫 레코드)
  , FIRST_VALUE(product_id)
      OVER(ORDER BY score DESC
          ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING)
      AS first_value
  
  -- 순위가 낮은 상품 ID(윈도 내부의 마지막 레코드)
  , LAST_VALUE(product_id)
      OVER(ORDER BY score DESC
          ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING)
      AS last_value

FROM popular_products
ORDER BY row
;
```
- 윈도 프레임 지정
  - 기본은 `ROWS BETWEEN start AND end`
    - `start`와 `end`는 `CURRENT_ROW`, `n PRECENDING`(n행 앞), `n FOLLOWING`(n행 뒤), `UNBOUBDED PRECENDING`(이전 행 전부), `UNBOUBDED FOLLOWING`(이후 행 전부) 등의 키워드를 지정
  - redshift에서는 `listagg`함수가 유사하게 있으나 프레임 지정과 동시에 사용할 수 없음
  - `ORDER BY` 절이 없으면 모든 행
  - `ORDER BY` 절의 default는 첫 행에서 현재 행까지
- 윈도 프레임 지정별 상품 ID를 집약하는 쿼리
```SQL
SELECT
  product_id
  , ROW_NUMBER()  OVER(ORDER BY score DESC) AS row

  -- 가장 앞 순위부터, 뒷 순위까지의 범위를 대상으로 상품 ID 집약
  -- PostgreSQL, array_agg
  , array_agg(product_id)
  -- Hive/SparkSQL, collect_list 사용
  , collect_list(product_id)
      OVER(ORDER BY score DESC
      ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING)
  AS whole_agg

  -- 가장 앞 순위부터 현재 순위까지의 범위를 대상으로 상품 ID 집약
  -- PostgreSQL, array_agg
  , array_agg(product_id)
  -- Hive/SparkSQL, collect_list 사용
  , collect_list(product_id)
      OVER(ORDER BY score DESC
          ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
  AS cum_agg

  -- 순위 하나 앞/뒤까지의 범위를 대상으로 상품 ID 집약
  , array_agg(product_id)
  , collect_list(product_id)
      OVER(ORDER BY score DESC
          ROWS BETWEEEN 1 PRECEDING AND 1 FOLLOWING)
  AS local_agg
FROM popular_products
WHERE category='action'
ORDER BY row
;
```
- PARTITION BY와 ORDER BY를 조합하여 카테고리들의 순위를 계산하는 쿼리
```SQL
SELECT
  category
  , product_id
  , score

  -- 카테고리별 점수 순서로 정렬, 유일 순위
  , ROW_NUMBER()
      OVER(PARTITION BY category ORDER BY score DESC)
  AS row

  -- 카테고리별 같은 순위 허가, 순차 순위
  , RANK()
      OVER(PARTITION BY category ORDER BY score DESC)
  AS rank

  -- 카테고리별 같은 순위 허가, 점프 순위
  , DENSE_RANK()
      OVER(PARTITION BY category ORDER BY score DESC)
  AS dense_rank
FROM popular_products
ORDER BY category, row
;
```
- 카테고리들의 순위 상위 2개까지의 상품을 추출하는 쿼리
```SQL
SELECT *
FROM

-- 서브 쿼리 내부에서 순위 계산
  ( SELECT
      category
      , product_id
      , score
      , ROW_NUMBER()
          OVER(PARTITION BY category ORDER BY score DESC)
      AS rank
   FROM popular_products
  ) AS popular_products_with_rank
WHERE rank <=2
ORDER BY category, rank
;
```
- 카테고리별 순위 최상위 상품을 추출하는 쿼리
```SQL
SELECT DISTINCT
  category
  , FIRST_VALUE(product_id)
      OVER(PARTITION BY category ORDER BY score DESC
          ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING)
  AS product_id
FROM popular_products
;
```

#### 7-3 세로 기반 데이터를 가로 기반으로 변환하기
- 행으로 지정된 지표 값을 열로 변환하는 쿼리
```SQL
SELECT
  dt
  , MAX(CASE WHEN indicator = 'impressions' THEN val END) AS impressions
  , MAX(CASE WHEN indicator = 'sessions' THEN val END) AS implressions
  , MAX(CASE WHEN indicator = 'users' THEN val END) AS users
FROM daily_kpi
GROUP BY dt
ORDER BY dt
;
```
- 행을 집약해서 쉼표로 구분된 문자열로 변환하기
```SQL
SELECT
  purchase_id

  -- 상품 ID 배열에 집약하고, 쉼표로 구분된 문자열로 변환
  -- PostgreSQL, BigQuery의 경우는 string_agg 사용하기
  , string_agg(product_id, ',') AS product_ids

  -- Redshift, listagg 사용
  , listagg(product_id, ',') AS product_ids

  -- Hive, SparkSQL, collect_list, concat_ws 사용
  , concat_ws(',' collect_list(product_id)) AS product_ids
  , SUM(price) AS amount
FROM purchase_detail_log
GROUP BY purchase_id
ORDER BY purchase_id
```

#### 7-4 가로 기반 데이터를 세로 기반으로 변환하기
- 컬럼으로 표현된 가로 기반 데이터는 데이터의 수가 고정되었다는 특징을 가짐
- 데이터 수와 같은 수의 일련 번호를 가진 `피벗 테이블`를 만들고 `CROSS JOIN`진행
```SQL
SELECT
  q.year

  -- q1부터 q4까지 레이블 이름 출력하기
  , CASE
    WHEN p.idx = 1 THEN 'q1'
    WHEN p.idx = 2 THEN 'q2'
    WHEN p.idx = 3 THEN 'q3'
    WHEN p.idx = 4 THEN 'q4'
  END AS quarter
  
  -- q1에서 q4까지의 매출 출력
  , CASE
    WHEN p.idx = 1 THEN q.q1
    WHEN p.idx = 2 THEN q.q2
    WHEN p.idx = 3 THEN q.q3
    WHEN p.idx = 4 THEN q.q4
  END AS sales
FROM
  quarterly_sales AS q
CROSS JOIN
-- 행으로 전개하고 싶은 열의 수만큼, 순번 테이블 만들기
  (SELECT 1 AS idx
  UNION ALL SELECT AS 2 AS idx
  UNION ALL SELECT AS 3 AS idx
  UNION ALL SELECT 4 AS idx
  ) AS p
```
- 테이블 함수를 사용해 배열을 행으로 전개하는 쿼리
```SQL
-- PostgreSQL의 경우 unnest 함수 사용하기
SELECT unnest(ARRAY['A001', 'A002' 'A003']) AS product_id;

-- BigQuery의 경우도 unnest 함수를 사용
-- 테이블 함수는 FROM에서만 사용 가능
SELECT * FROM unnest(ARRAY['A001', 'A002', 'A003']) AS product_id;

-- Hive, SparkSQL의 경우 explode 함수 사용
 SELECT explode(ARRAY('A001', 'A002', 'A003')) AS product_id;
```
- 테이블 함수를 사용해 쉼표로 구분된 문자열 데이터를 행으로 전개하는 쿼리
```SQL
SELECT
  purshase_id
  , product_id
FROM
  purchase_log AS p
-- string_to_array 함수, 문자열 -> 배열 변환, unnest 함수로 테이블 변환
CROSS_JOIN unnest(string_to_array(product_ids, ',')) AS product_id

-- BigQuery의 경우 문자열 분해에 split 함수 사용
CROSS_JOIN unnest(split(product_ids, ',')) AS product_id

-- Hive, SparkSQL, LATERAL VIEW explode 사용
LATERAL VIEW explode(split(product_ids, ',')) e AS product_id
```
- PostgreSQL에서 쉼표로 구분된 데이터를 행으로 전개하는 쿼리
```SQL
SELECT
  purcahse_id
  -- 쉼표로 구분된 문자열을 한번에 행으로 전개
  , regexp_split_to_table(product_ids, ',') AS prodcut_id
FROM purchase_log;
```
- Redshift는 배열 자료형을 제공하지않아 전처리가 더 필요
  - 일련 변호(데이터 최대 수)를 가진 피벗 테이블을 만드는 쿼리
  ```SQL
    SELECT *
    FROM (
      SELECT 1 AS idx
      UNION ALL SELECT 2 AS idx
      UNION ALL SELECT 3 AS idx
    ) AS pivot
    ;
  ```
  - split_part 함수의 사용 예 (n번째 요소 추출)
  ```SQL
  SELECT
  split_part('A001,A002,A003', ',', 1) AS part_1
  , split_part('A001,A002,A003', ',', 2) AS part_2 
  , split_part('A001,A002,A003', ',', 3) AS part_3
  ;
  ```
  - 문자 수의 차이를 사용해 상품 수를 계산하는 쿼리
  ```SQL
  SELECT
  purchase_id
  , product_ids
  -- 상품 ID의 문자열을 기반으로 쉼표를 제거
  -- 문자 수의 차이를 계산하여 상품수 계산
  , 1 + char_length(product_ids)
   - char_length(replace(product_ids, ',', ''))
  AS product_num
  FROM
    purchase_log
  ;
  ```
  - 피벗 테이블을 사용해 문자열을 행으로 전개하는 쿼리
  ```SQL
  SELECT
  l.purchase_id
  , l.product_ids
  -- 상품 수만큼 순번 붙이기
  , p.idx
  -- 문자열 쉼표 구분하여 분할, idx 요소 추출
  , split_part(l.product_ids, ',', p.idx) AS product_id
  FROM
    purchase_log AS l
  JOIN
    ( SELECT 1 AS idx
      UNION ALL SELECT 2 AS idx
      UNION ALL SELECT 3 AS idx
    ) AS p
  -- 피벗 테이블의 id가 상품 수 이하인 경우 결합
  ON p.idx <=
    (1 + char_length(l.product_ids)
      - char_length(replace(l.product_ids, ',', '')))
  ;
  ```
- SQL을 레코드 단위로 분할해두는 것이 기본이지만 1개의 레코드에 집약시키지 못하는 경우가 있어 데이터를 행으로 변환하는 테크닉을 이용해야 함.

---

## 8강 여러 개의 테이블 조작하기

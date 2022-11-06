# CH7 데이터 활용의 정밀도를 높이는 분석 기술

---

## 17강 데이터를 조합해서 새로운 데이터 만들기

- 데이터가 없을 때를 대비하여 오픈 데이터들을 조합해서 활용할 줄 알아야 한다

---

### 17-1 IP 주소를 기반으로 국가와 지역 보완하기

- IP 주소를 가지고 있으면 국가 지역 보완 가능
- [GeoLite2](https://dev.maxmind.com/geoip/geolite2-free-geolocation-data?lang=en) 내려받기
  - CSV 데이터 로드하는 쿼리
    - country: 국가 정보
    - city: 도시 정보 까지

    ```sql
    DROP TABLE IF EXISTS mst_city_ip;
    CREATE TABLE mst_city_ip (
    network                           inet PRIMARY KEY
    , geoname_id                      integer
    , registered_country_geoname_id   integer
    , represented_country_geoname_id  integer
    , is_anonymous_proxy              boolean
    , is_satellite_provider           boolean
    , postal_code                     varchar(255)
    , latitude                        numeric
    , longitude                       numeric
    , accuracy_radius                 integer
    );

    DROP TABLE IF EXISTS mst_locations;
    CREATE TABLE mst_locations (
    , geoname_id                      integer PRIMARY KEY
    , locale_code                     varchar(255)
    , contient_code                   varchar(10)
    , contient_name                   varchar(255)
    , country_iso_code                varchar(10)
    , country_name                    varchar(255)
    , subdivision_1_iso_code          varchar(10)
    , subdivision_1_name              varchar(255)
    , subdivision_2_iso_code          varchar(10)
    , subdivision_2_name              varchar(255)
    , city_name                       varchar(255)
    , metro_code                      integer
    , time_zone                       varchar(255)
    );

    COPY mst_city_ip FROM '/path/to/GeoLite2-City-Blocks-IPv4.csv' WITH CSV HEADER;
    COPY mst_locations FROM '/path/to/GeoLite2-City-Locations-en.csv' WITH CSV HEADER;
    ```

- IP 주소로 국가와 지역 정보 보완하기
  - 액션 로그의 IP 주소로 국가와 지역 정보를 추출하는 쿼리

    ```sql
    SELECT
    a.ip
    , l.contient_name
    , l.country_name
    , l.city_name
    , l.time_zone
    FROM
    action_log AS a
    LEFT JOIN
        mst_city_ip AS i
        ON a.ip::inet << i.network
    LEFT JOIN
        mst_locations AS l
        ON i.geoname_id = l.geoname_id
    ```

### 17-2 주말과 공휴일 판단하기

- 연도와 월에 주말과 공휴일이 얼마나 있는지 판단하여 목표 세울 때 사용
- 공휴일 정보 테이블을 따로 만들어 사용
  
    ```sql
    CREATE TABLE mst_calendar(
    year            integer
    , month         integer
    , day           integer
    , dow           varhcar(10)
    , dow_num       integer
    , holiday_name  varchar(255)
    );
    ```

- 주말과 공휴일 판정하기

    ```sql
    SELECT
    a.action
    , a.stamp
    , c.dow
    , c.holiday_name
    -- 주말과 공휴일 판정
    , c.dow_num IN (0, 6) -- 토요일과 일요일 판정하기
    OR c.holiday_name IS NOT NULL -- 공휴일 판정하기
    AS is_day_off
    FROM
    access_log AS a
    JOIN
        mst_calendar AS c
        -- 액션 로그의 타임스탬프에서 연, 월, 일을 추출하고 결합하기
        -- PostgrdSQL, Hive, Redshift, SparkSQL의 경우 다음과 같이 사용
        -- BigQuery의 경우 substring을 substr, int를 int64로 수정하기
        ON  CAST(substring(a.stamp, 1, 4) AS int) = c.year
        AND CAST(substring(a.stamp, 6, 2) AS int) = c.month
        AND CAST(substring(a.stamp, 9, 2) AS int) = c.day
    ;
    ```

### 17-3 하루 집계 범위 변경하기

- 0시 기준이 아닌 하루 24시간을 다른 범위로 보기 ex) 오전 4시 ~ 다음날 오전 4시
  - 자정 전후 사용 비율이 높아 지속률 계산의 차이가 생김
- 하루 집계 범위 변경하기

    ```sql
    WITH
    action_log_with_mod_stamp AS (
    SELECT *
    -- 4시간 전의 시간 계산하기
    -- PostgreSQL의 경우 interval 자료형의 데이터를 사용해 날짜를 사칙연산 할 수 있음
    , CAST(stamp::timestamp - '4 hours'::interval AS text) AS mod_stamp
    -- Redshift의 경우 dateadd 함수 사용하기
    , CAST(dateadd(hour, -4, stamp::timestamp) AS text) AS mod_stamp
    -- BigQuery의 경우 timestamp_sub 함수 사용하기
    , CAST(timestamp_sub(timestamp(stamp), interval 4 hour) AS string) AS mod_stamp
    -- Hive, SparkSQL의 경우 한 번 unixtime으로 변환한 뒤 초 단위로 계산하고, 다시 타임스탬프로 변환하기
    , from_unixtime(unix_timestamp(stamp) -4 * 60 * 60) AS mod_stamp
    FROM action_log
    )
    SELECT
    session
    , user_id
    , action
    , stamp
    -- raw_date와 4시간 후를 나타내는 mod_date 추출하기
    -- PostgreSQL, Hive, Redshift, SparkSQL의 경우 다음과 같이 사용
    , substring(stamp, 1, 10) As raw_date
    , subsing(mod_stamp, 1, 10) As mod_date
    -- BigQuery의 경우 substring을 substr로 수정하기
    , substr(stamp, 1, 10) AS raw_date
    , substr(mod_stamp, 1, 10) AS mod_date
    FROM action_log_with_mod_stamp;
    ```

## 18강 이상값 검출하기

- 데이터 분석은 데이터 정합성 보장을 전제하지만 실 데이터는 누락이나 노이즈가 많다.
- `데이터 클렌징`을 소개

---

### 18-1 데이터 분산 계산하기

- 분산에서 많이 벗어난 값을 찾기
- 세션별로 페이지 열람 수 랭킹 비율을 구하는 쿼리
  - `PERCENT_RANK` 함수로 (rank - 1)/(전체 수 - 1) 계산
  - 0.05 이하인 것만 필터링 하면 상위 5%

    ```sql
    WITH
    session_count AS (
    SELECT
        session
        , COUNT(1) AS count
    FROM
        action_log_with_noise
    GROUP BY
        session
    )
    SELECT
    session
    , count
    , RANK() OVER(ORDER BY count DESC) AS rank
    , PERCENT_RANK() OVER(ORDER BY count DESC) AS percent_rank
    FROM
    session_count
    ;
    ```

- URL 접근 수 워스트 랭킹 비율 구하기
  - window함수 내 count 오름차순 정렬으로 구함

    ```sql
    WITH
    url_count AS (
    SELECT
        url
        , COUNT(*) AS count
    FROM
        action_log_with_noise
    GROUP BY
        url
    )
    SELECT
    url
    , count
    , RANK() OVER(ORDER BY count ASC) AS rank
    , PERCENT_RANK() OVER(ORDER BY count ASC)
    FROM
    url_count
    ;
    ```

### 18-2 크롤러 제외하기

- 규칙을 기반으로 크롤러를 제외하는 쿼리

    ```sql
    SELECT
    *
    FROM
    action_log_with_noise
    WHERE
    NOT
    -- 크롤러 판정 조건
    ( user_agent LIKE '%bot%'
    OR user_agent LIKE '%crawler%'
    OR user_agent LIKE '%spider%'
    OR user_agent LIKE '%archiver%'
    -- 생략
    )
    ;
    ```

- 마스터 데이터를 사용해 제외하는 쿼리
  - 별도의 크롤러 마스터 데이터를 만들어 번거로움을 줄이기

    ```sql
    WITH
    mst_bot_user_agent AS (
    SELECT '%bot%' AS rule
    UNION ALL SELECT '%crawler%' AS rule
    UNION ALL SELECT '%spider%' AS rule
    UNION ALL SELECT '%archiver%' AS rule
    )
    , filtered_action_log AS (
    SELECT
        l.stamp, l.session, l.action, l.products, l.url, l.ip, l.user_agent
        -- UserAgent의 규칙에 해당하지 않는 로그만 남기기
        -- PostgreSQL, Redshift, BigQuery의 경우 WHERE 구문에 상관 서브쿼리 사용 가능
    FROM
        action_log_with_noise AS l
    WHERE
        NOT EXISTS (
        SELECT 1
        FROM mst_bot_user_agent AS m
        WHERE
            l.user_agent LIKE m.rule
        )
    -- 상관 서브 쿼리를 사용할 수 없는 경우
    -- CROSS JOIN으로 마스터 테이블을 결합하고
    -- HAVING 구문으로 일치하는 규칙이 0(없는) 레코드만 남기기
    -- PostgreSQL, Hive, Redshift, BigQuery, SparkSQL의 경우
    FROM
        action_log_with_noise AS l
        CROSS JOIN
        mst_bot_user_agent AS m
    GROUP BY
        l.stamp, l.session, l.action, l.products, l.url, l.ip, l.user_agent
    HAVING SUM(CASE WHEN l.user_agent LIKE m.rule THEN 1 ELSE 0 END) = 0
    )
    SELECT
    *
    FROM
    filtered_action_log
    ;
    ```

- 크롤러 감시하기
  - 접근이 많은 `user-agent`를 순위대로 추출하여 마스터 테이블 리스트 관리

    ```sql
    WITH
    mst_bot_user_agent AS (
    -- CODE.18.4.
    )
    , filtered_action_log AS (
    -- CODE.18.4.
    )
    SELECT
    user_agent
    , COUNT(1) AS count
    , 100.0
        * SUM(COUNT(1)) OVER(ORDER BY COUNT(1) DESC
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
        / SUM(COUNT(1)) OVER() AS cumulative_ratio
    FROM
    filtered_action_log
    GROUP BY
    user_agent
    ORDER BY
    count DESC
    ;
    ```

### 18-3 데이터 타당성 확인하기

- 로그 데이터의 요건을 만족하는지 확인하는 쿼리
  - 액션들을 `GROUP BY`로 집약하고 만족해야하는 컬럼을 `CASE`식으로 0, 1로 출력 후 AVG로 만족하는 비율을 계산
  - 모든 컬럼이 요건을 만족하면 1.0이고

    ```sql
    SELECT
    action
    -- session은 반드시 NULL이 아니어야 함
    , AVG(CASE WHEN session IS NOT NULL THEN 1.0 ELSE 0.0 END) AS session
    -- user_id는 반드시 NULL이 아니어야 함
    , AVG(CASE WHEN user_id IS NOT NULL THEN 1.0 ELSE 0.0 END) AS user_id
    -- category는 action=view의 경우 NULL, 이외의 경우 NULL이 아니어야 함
    , AVG(
        CASE action
        WHEN 'view' THEN
            CASE WHEN category IS NULL THEN 1.0 ELSE 0.0 END
        ELSE
            CASE WHEN category IS NOT NULL THEN 1.0 ELSE 0.0 END
        END
    ) as category
    -- products는 action=view의 경우 NULL, 이외에 NULL이면 X
    , AVG(
        CASE action
        WHEN 'view' THEN
            CASE WHEN products IS NULL THEN 1.0 ELSE 0.0 END
        ELSE
            CASE WHEN products IS NOT NULL THEN 1.0 ELSE 0.0 END
        END
    ) AS products
    -- amount는 action=purchase의 경우 NULL이 아니어야 하며, 이외의 경우는 NULL
    , AVG(
        CASE action
        WHEN 'purchase' THEN
            CASE WHEN amount IS NOT NULL THEN 1.0 ELSE 0.0 END
        ELSE
            CASE WHEN amount IS NULL THEN 1.0 ELSE 0.0 END
        END
    ) AS amount
    -- stamp는 반드시 NULL이 아니어야 함
    , AVG(CASE WHEN stamp is NOT NULL 1.0 ELSE 0.0 END) AS stamp
    FROM
    invalid_action_log
    GROUP BY
    action
    ;
    ```

### 18-4 특정 IP 주소에서의 접근 제외하기

- PostgreSQL에서 특정 IP 주소(사내 IP, 테스트, 크롤러 등) 제외하기
  - 마스터 테이블 만들기

    ```sql
    WITH
    mst_reserved_ip AS (
                SELECT '127.0.0.0/8'    AS network, 'localhost'       AS description
    UNION ALL SELECT '10.0.0.0/8'     AS network, 'Private network' AS description
    UNION ALL SELECT '172.16.0.0/12'  AS network, 'Private network' AS description
    UNION ALL SELECT '192.0.0.0/24'   AS netowrk, 'Private network' AS description
    UNION ALL SELECT '192.169.0.0/16' AS network, 'Private netowrk' AS description
    )
    SELECT *
    FROM mst_reserved_ip
    ;
    ```

  - inet 자료형을 사용해 IP 주소를 판정하는 쿼리
    - IP주소가 마스터 테이블 네트워크에 포함되어있는지 결합 조건으로 확인 가능
    - `network IS NULL` 로 마스터 테이블 주소 로그 제외 가능

    ```sql
    WITH
    mst_reserved_ip AS (
    -- CODE.18.7.
    )
    , action_log_with_reserved_ip AS (
    SELECT
        l.user_id
        , l.ip
        , l.stamp
        , m.network
        , m.description
    FROM
        action_log_with_ip AS l
        LEFT JOIN mst_reserved_ip AS m
        ON l.ip::inet << m.network::inet
    )
    SELECT *
    FROM action_log_with_reserved_ip;
    ```

- PostgreSQL 이외의 경우
  - 과정
    - 네트워크 범위를 처음 IP 주소와 끝 IP 주소로 표현
    - IP 주소를 대소 비교 가능한 형식으로 변환(정수 자료형, 0으로 매워 문자열)
  - 네트워크 범위를 나태는 처음과 끝 IP 주소를 부여하는 쿼리

    ```sql
    WITH
    mst_reserved_ip_with_range AS (
    -- 마스터 테이블에 네트워크 범위에 해당하는 IP 주소의 최솟값과 최댓값 추가하기
    SELECT '127.0.0.0/8'      AS network
            ,'127.0.0.0'        AS network_start_ip
            ,'127.255.255.255'  AS network_last_ip
            ,'localhost'        AS description
    )
    UNION ALL
    SELECT '10.0.0.0/8'      AS network
            ,'10.0.0.0'        AS network_start_ip
            ,'10.255.255.255'  AS network_last_ip
            ,'Private netowrk' AS description
    UNION ALL
    SELECT '172.16.0.0/12'   AS network
            ,'172.16.0.0'      AS network_start_ip
            ,'172.31.255.255'  AS network_last_ip
            ,'Private netowrk' AS description
    UNION ALL
    SELECT '192.0.0.0/24'     AS network
            ,'192.0.0.0'        AS network_start_ip
            ,'192.0.0.255'      AS network_last_ip
            ,'Private netowrk'  AS description
    UNION ALL
    SELECT '192.168.0.0/16'     AS network
            ,'192.168.0.0'        AS network_start_ip
            ,'192.168.255.255'      AS network_last_ip
            ,'Private netowrk'  AS description
    )
    SELECT *
    FROM mst_reserved_ip_with_range;
    ```

  - IP주소를 0으로 메운 문자열로 변환하고, 특정 IP 로그를 배제하는 쿼리
    - IP 로그를 매번 변환하는 것은 비용이 많이 들어 저장해서 사용하는 것을 추천

    ```sql
    WITH
    mst_reserved_ip_with_range AS (
    -- CODE.18.10.
    )
    , action_log_with_ip_varchar AS (
    -- 액션 로그의 IP 주소를 0으로 메운 문자열로 표현하기
    SELECT
        *
        -- PostgreSQL, Redshift의 경우 lpad 함수로 0 메우기
        , lpad(split_part(ip, '.', 1), 3, '0')
        || lpad(split_part(ip, '.', 2), 3, '0')
        || lpad(split_part(ip, '.', 3), 3, '0')
        || lpad(split_part(ip, '.', 4), 3, '0')
        AS ip_varchar

        -- BigQuery의 경우 split 함수를 사용해 배열로 분해하고 n번째 요소 추출하기
        , CONCAT(
        lpad(split(ip, '.')[SAFE_ORDINAL(1)], 3, '0')
        , lpad(split(ip, '.')[SAFE_ORDINAL(2)], 3, '0')
        , lpad(split(ip, '.')[SAFE_ORDINAL(3)], 3, '0')
        , lpad(split(ip, '.')[SAFE_ORDINAL(4)], 3, '0')
        ) AS ip_varchar

        -- Hive, SparkSQL의 경우 split 함수로 배열로 분해하고, n번째 요소 추출하기
        -- 다만 콜론(마침표)는 특수 문자이므로 역슬래시로 이스케이프 처리해야 함
        , CONCAT(
        lpad(split(ip, '\\.')[0], 3, '0')
        , lpad(split(ip, '\\.')[1], 3, '0')
        , lpad(split(ip, '\\.')[2], 3, '0')
        , lpad(split(ip, '\\.')[3], 3, '0')
        ) AS ip_varchar
    FROM
        action_log_with_ip
    )
    , mst_reserved_ip_with_varchar_range AS (
    -- 마스터 테이블의 IP 주소를 0으로 메운 문자열로 표현하기
    SELECT
        *
        -- PostgreSQL, Redshift의 경우 lpad 함수로 0 메우기
        , lpad(split_part(network_start_ip, '.', 1), 3, '0')
        || lpad(split_part(network_start_ip, '.', 2), 3, '0')
        || lpad(split_part(network_start_ip, '.', 3), 3, '0')
        || lpad(split_part(network_start_ip, '.', 4), 3, '0')
        AS network_start_ip_varchar
        , lpad(split_part(network_last_ip, '.', 1), 3, '0')
        || lpad(split_part(network_last_ip, '.', 2), 3, '0')
        || lpad(split_part(network_last_ip, '.', 3), 3, '0')
        || lpad(split_part(network_last_ip, '.', 4), 3, '0')
        AS network_last_ip_varchar

        -- BigQuery의 경우 split 함수를 사용해 배열로 분해하고 n번째 요소 추출하기
        , CONCAT(
        lpad(split(network_start_ip, '.')[SAFE_ORDINAL(1)], 3, '0')
        , lpad(split(network_start_ip, '.')[SAFE_ORDINAL(2)], 3, '0')
        , lpad(split(network_start_ip, '.')[SAFE_ORDINAL(3)], 3, '0')
        , lpad(split(network_start_ip, '.')[SAFE_ORDINAL(4)], 3, '0')
        ) AS network_start_ip_varchar
        , CONCAT(
        lpad(split(network_last_ip, '.')[SAFE_ORDINAL(1)], 3, '0')
        , lpad(split(network_last_ip, '.')[SAFE_ORDINAL(2)], 3, '0')
        , lpad(split(network_last_ip, '.')[SAFE_ORDINAL(3)], 3, '0')
        , lpad(split(network_last_ip, '.')[SAFE_ORDINAL(4)], 3, '0')
        ) AS network_last_ip_varchar

        -- Hive, SparkSQL의 경우 split 함수를 사용해 배열로 분해하고 n번째 요소 추출하기
        -- 다만 콜론(마침표)은 특수 문자이므로 역슬래시로 이스케이프 처리해야 함
        , CONCAT(
        lpad(split(network_start_ip, '\\.')[0], 3, '0')
        , lpad(split(network_start_ip, '\\.')[1], 3, '0')
        , lpad(split(network_start_ip, '\\.')[2], 3, '0')
        , lpad(split(network_start_ip, '\\.')[3], 3, '0')
        ) AS network_start_ip_varchar
        , CONCAT(
        lpad(split(network_last_ip, '\\.')[0], 3, '0')
        , lpad(split(network_last_ip, '\\.')[1], 3, '0')
        , lpad(split(network_last_ip, '\\.')[2], 3, '0')
        , lpad(split(network_last_ip, '\\.')[3], 3, '0')
        ) AS network_last_ip_varchar
    FROM
        mst_reserved_ip_with_range
    )

    -- 0으로 메운 문자열을 표현한 IP 주소로, 네트워크 범위에 포함되는지 판정하기
    SELECT
    l.user_id
    , l.ip
    , l.ip_varchar
    , l.stamp
    FROM
    action_log_with_ip_varchar AS l
    CROSS JOIN
        mst_reserved_ip_with_varchar_range AS m
    GROUP BY
    l.user_id, l.ip, l.ip_varchar, l.stamp
    -- 예약된 IP 주소 마스터에 포함되지 않은 IP 로그만 HAVING 구문으로 추출하기
    HAVING
    SUM(CASE WHEN l.ip_varchar
        BETWEEN m.network_start_ip_varchar AND m.network_last_ip_varchar
        THEN 1 ELSE 0 END) = 0
    ;
    ```

## 19강 데이터 중복 검출하기

---

- 무결성이 있지 않는 이상 데이터 중복이 발생함

- 마스터 데이터의 중복 검출하기
  - 키의 중복을 확인하는 쿼리

    ```sql
    SELECT
    COUNT(1) AS total_num
    , COUNT(DISTINCT id) AS key_num
    FROM mst_categories
    ;
    ```

  - 키가 중복되는 레코드의 값 확인하기
    - `HAVING`절로 확인 가능
  
    ```sql
    SELECT
    id
    , COUNT(*) AS record_num

    -- 데이터를 배열로 집약하고, 쉼표로 구분된 문자열로 변환하기
    -- PostgreSQL, BigQuery; string_agg
    , string_agg(name, ',') AS name_list
    , string_agg(stamp, ',') AS stamp_list
    -- Redshift; listagg
    , listagg(name, ',') AS name_list
    , listagg(stamp, ',') AS stamp_list
    -- Hive, SparkSQL; collect_list, concat_ws
    , concat_ws(',', collect_list(name)) AS name_list
    , concat_ws(',', collect_list(stamp)) AS stamp_list
    FROM
    mst_categories
    GROUP BY id
    -- 중복된 ID 확인하기
    HAVING COUNT(*) > 1
    ;  
    ```

  - 윈도 함수를 사용해서 중복된 레코드를 압축하는 쿼리

    ```sql
    WITH
    mst_categories_with_key_num AS (
    SELECT
        *
        -- id 중복 세기
        , COUNT(1) OVER(PARTITION BY id) AS key_num
    FROM
        mst_categories
    )
    SELECT
    *
    FROM
    mst_categories_with_key_num
    WHERE
    key_num > 1 -- ID가 중복되는 경우 확인
    ;
    ```

- 여러번 데이트 로드가 원인이라면 여러번 로드해도 같은 실행 결과가 보증되게 만들기
- 새로운 데이터와 오래된 데이터가 중복된 경우에는 새로운 데이터 남기거나, 타임스탬프를 포함해서 유니크 키를 구성하는 방법

### 19-2 로그 중복 검출하기

- 2회 연속 클릭이나 여러번 새로고침으로 중복된 로그가 발생할 수 있음
- 사용자와 상품의 조합에 대한 중복을 확인하는 쿼리

    ```sql
    SELECT
    user_id
    , products
    -- 데이터를 배열로 집약하고, 쉼표로 구분된 문자열로 변환
    -- PostgreSQL, BigQuery의 경우는 string_agg 사용하기
    , string_agg(session, ',') AS session_list
    , string_agg(stamp, ',') AS stamp_list
    -- Redshift의 경우 listagg 사용하기
    , listagg(session, ',') AS session_list
    , listagg(stamp, ',') AS stamp_list
    -- Hive, SparkSQL의 경우 collect_list, concat_ws 사용
    , concat_ws(',', collect_list(session)) AS session_list
    , concat_ws(',', collect_list(stamp)) AS stamp_list
    FROM
    dup_action_log
    GROUP BY
    user_id, products
    HAVING
    COUNT(*) > 1
    ;
    ```

- 중복 데이터 배제하기
  - 같은 세션, 상품일 때 가장 오래된 데이터만을 남기기
  - `GROUP BY`와 `MIN`을 사용해 중복을 배제하는 쿼리
    - 타임스탬프 이외의 컬럼도 활용하여 중복을 제거해야 돼서 단순 집약함수를 적용할 수 없으면 사용하지 못함

    ```sql
    SELECT
    session
    , user_id
    , action
    , products
    , MIN(stamp) AS stamp
    FROM
    dup_action_log
    GROUP BY
    session, user_id, action, products
    ;
    ```

  - `ROW_NUMBER`를 사용해 중복을 배제하는 쿼리

    ```sql
    WITH
    dup_action_log_with_order_num AS (
    SELECT
        *
        -- 중복된 데이터에 순번 붙이기
        , ROW_NUMBER()
            OVER(
            PARTITION BY session, user_id, action, products
            ORDER BY stamp
            ) AS order_num
    FROM
        dup_action_log
    )
    SELECT
    session
    , user_id
    , action
    , products
    , stamp
    FROM
    dup_action_log_with_order_num
    WHERE
    order_num = 1 -- 순번이 1인 데이터(중복된 것 중에서 가장 앞의 것)만 남기기
    ;
    ```

  - 이전 액션으로부터  경과 시간을 계산하는 쿼리

    ```sql
    WITH
    dup_action_log_with_lag_seconds AS (
    SELECT
        user_id
        , action
        , products
        , stamp
        -- 같은 사용자와 상품 조합에 대한 이전 액션으로부터의 경과 시간 계산하기
        -- PostgreSQL의 경우 timestamp 자료형으로 변환하고 차이를 구한 뒤
        -- EXTRACT(epoc ~)를 사용해 초 단위로 변경하기
        , EXTRACT(epoch from stamp::timestamp - LAG(stamp::timestamp)
            OVER(
            PARTITION BY user_id, action, products
            ORDER BY stamp
            )) AS lag_seconds
        -- Redshift의 경우 datediff 함수를 초 단위로 지정
        , datediff(second, LAG(stamp::timestamp)
            OVER(
            PARTITION BY user_id, action, products
            ORDER BY stamp
            ), stamp::timestamp) AS lag_seconds
        -- BigQuery의 경우 unix_seconds 함수로 초 단위 UNIX 타임으로 변환하고 차이 구하기
        , unix_seconds(timestamp(stamp)) - LAG(unix_seconds(timestamp(stamp)))
            OVER(
            PARTITION BY user_id, action, products
            ORDER BY stamp
        -- SparkSQL의 경우 다음과 같이 프레임 지정 추가
            ROWS BETWEEN 1 PRECEDING AND 1 PRECEDING
            ) AS lag_seconds
    FROM
        dup_action_log
    )
    SELECT
    *
    FROM
    dup_action_log_with_lag_seconds
    ORDER BY
    stamp;
    ```

  - 30분 이내의 같은 액션을 중복으로 보고 배제하는 쿼리

    ```sql
    WITH
    dup_action_log_with_lag_seconds AS (
    -- CODE.19.7
    )
    SELECT
    user_id
    , action
    , products
    , stamp
    FROM
    dup_action_log_with_lag_seconds
    WHERE
    (lag_seconds IS NULL OR lag_seconds >= 30 * 60)
    ORDER BY
    stamp
    ;
    ```

- 중복된 데이터의 리포트를 만들면 잘못된 판단할 수 있음. **데이터 중복 제거**를 해야함

## 20강 여러 개의 데이터셋 비교하기

- 데이터 변화한 부분 확인
- 데이터의 차이를 추출하는 경우
  - 우편 번호가 대량으로 변경될 때 어떤 형태로 변경되었는지 알 수 있으면 데이터를 적절히 변경 가능
- 데이터의 순위를 비교하는 경우
  - 인기순위 같은 경우 새로운 로직과 과거의 로직 변화를 정량적으로 설명할 수 있어야 함

### 20-1 데이터의 차이 추출하기

- 추가된 마스터 데이터 추출하는 쿼리
  - 새로운 테이블 기준으로 `LEFT OUTER JOIN` 하여 오래된 테이블의 컬럼이 NULL인 레코드 추출하면 됨

    ```sql
    SELECT
    new_mst.*
    FROM
    mst_products_20170101 AS new mst
    LEFT OUTER JOIN
        mst_products_20161201 AS old_mst
        ON
        new_mst.product_id = old_mst.product_id
    WHERE
    old_mst.product_id IS NULL
    ```

- 제거된 마스터 데이터 추출하는 쿼리
  - 테이블 순서는 그대로 유지하되, `RIGHT OUTER JOIN` 사용

    ```sql
    SELECT
    old_mst.*
    FROM
    mst_products_20170101 AS new_mst
    RIGHT OUTER JOIN
        mst_products_20161201 AS old_mst
        ON
        new_mst.product_id = old_mst.product_id
    WHERE
    new_mst.product_id IS NULL
    ```

- 갱신된 마스터 데이터 추출하는 쿼리
  - 타임스탬프 갱신이 일어난다는 것을 전제로 갱신시점이 다른 레코드만 추출

```sql
SELECT
  new_mst.product_id
  , old_mst.name AS old_name
  , old_mst.price AS old_price
  , new_mst.name AS new_name
  , new_mst.price AS new_price
  , new_mst.updated_at
FROM
  mst_products_20170101 AS new_mst
  JOIN
    mst_products_20161201 AS old_mst
    ON
      new_mst.product_id = old_mst.product_id
WHERE
  -- 갱신 시점이 다른 레코드만 추출하기
  new_mst.updated_at <> old_mst.updated_at
```

- 변경된 마스터 데이터 모두 추출하는 쿼리
  - `FULL OUTER JOIN` 사용
  - `IS DISNINCT FROM` 연산자를 통해 한쪽에만 NULL 있는 레코드를 확인

    ```sql
    SELECT
    COALESCE(new_mst.product_id, old_mst.product_id) AS product_id
    , COALESCE(new_mst.name, old_mst.name) AS name
    , COALESCE(new_mst.price, old_mst.price) AS price
    , COALESCE(new_mst.updated_at, old_mst.updated_at) AS updated_at
    , CASE
        WHEN old_mst.updated_at IS NULL THEN 'added'
        WHEN new_mst.updated_at IS NULL THEN 'deleted'
        WHEN new_mst.updated_at <> old_mst.updated_at THEN 'updated'
        END AS status
    FROM
    mst_products_20170101 AS new_mst
    FULL OUTER JOIN
        mst_products_20161201 AS old_mst
        ON
        new_mst.product_id = old_mst.product_id
    WHERE
    -- PostgreSQL의 경우 IS DISTINCT FROM 연산자를 사용해 NULL을 포함한 비교 가능
    new_mst.updated_at IS DISTINCT FROM old_mst.updated_at
    -- Redshift, BigQuery의 경우
    -- IS DISTINCT FROM 구문 대신 COALESCE 함수로 NULL을 default 값으로 변환하고 비교하기
    COALESCE(new_mst.updated_at, 'infinity')
    <> COALESCE(old_mst.updated_at, 'infinity')
    ;
    ```

### 20-2 두 순위의 유사도 계산하기

- `방문 횟수`, `방문자 수`, `페이지 뷰`를 기반으로 순위를 작성하는 쿼리

    ```sql
    WITH
    path_stat AS (
    -- 경로별 방문 횟수, 방문자 수, 페이지 뷰 계산
    SELECT
        path
        , COUNT(DISTINCT long_session) AS access_users
        , COUNT(DISTINCT short_session) AS access_count
        , COUNT(*) AS page_view
    FROM
        access_log
    GROUP BY
        path
    )
    , path_ranking AS (
    -- 방문 횟수, 방문자 수, 페이지 뷰별로 순위 붙이기
    SELECT 'access_user' AS type, path, RANK() OVER(ORDER BY access_users DESC) AS rank
        FROM path_stat
    UNION ALL
        SELECT 'access_count' AS type, path, RANK() OVER(ORDER BY access_count DESC) AS rank
        FROM path_stat
    UNION ALL
        SELECT 'page_view' AS type, path, RANK() OVER(ORDER BY page_view DESC) AS rank
        FROM path_stat
    )
    SELECT *
    FROM
    path_ranking
    ORDER BY
    type, rank
    ;
    ```

- 경로들의 순위 차이를 계산하는 쿼리
  - `방문 횟수` `방문자 수` 의 비교를 하여 제곱하여 결과 리턴

    ```sql
    WITH
    path_stat AS (
    -- CODE.20.5.
    )
    , path_ranking AS (
    -- CODE.20.5.
    )
    , pair_ranking AS (
    SELECT
        r1.path
        , r1.type AS type1
        , r1.rank AS rank1
        , r2.type AS type2
        , r2.rank AS rank2
        -- 순위 차이 계산하기
        , POWER(r1.rank - r2.rank, 2) AS diff
    FROM
        path_ranking AS r1
        JOIN
        path_ranking AS r2
        ON r1.path = r2.path
    )
    SELECT
    *
    FROM
    pair_ranking
    ORDER BY
    type1, type2, rank1
    ;
    ```

- 스피어만 상관계수 계산하기
  - 두 개의 순위가 완전 일치할 경우 1.0, 완전히 일치하지 않을 경우 -1.0
  - 양수면 관련성이 있고, 음수면 관련성이 없음

    ```sql
    WITH
    path_stat AS (
    -- CODE.20.5.
    )
    , path_ranking AS (
    -- CODE.20.5.
    )
    , pair_ranking AS (
    -- CODE.20.6.
    )
    SELECT
    type1
    , type2
    -- 스피어만 상관계수 계산
    , 1 - (6.0 * SUM(diff) / (POWER(COUNT(1), 3) - COUNT(1))) AS spearman
    FROM
    ir_ranking
    GROUP BY
    type1, type2
    ORDEr BY
    type1, spearman DESC
    ;
    ```

- 국어성적이 높으면 영어성적이 높을까? 처럼 순위간의 유사성을 갖는지 확인하여 이상적인 순위를 자동으로 생성할 수 있음.

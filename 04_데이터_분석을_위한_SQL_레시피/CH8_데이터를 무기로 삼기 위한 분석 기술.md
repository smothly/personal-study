# CH8 데이터 무기로 삼기 위한 분석 기술

- 어떤 검색 쿼리를 입력하고 어떤 결과를 얻는지 분석하는 작업이 중요
---

## 21강 검색 기능 평가하기

- 검색이 원하는 결과면 상세 화면으로 감. 원하지 않는 결과면 이탈이나 다시 검색을 함
- 개선 방법
  - 검색 키워드의 흔들림을 흡수할 수 있게 `동의어 사전` 추가. ex) 알파벳이나 줄임말 검색
  - 검색 키워드를 검색 엔진이 이해할 수 있게 `사용자 사전` 추가. ex) 위스키 -> 위 / 스키 로 분해되어 인덱싱
  - 검색 결과가 사용자가 원하는 순서로 나오게 `정렬 순서 조정` 하기. ex) 문장 출현 위치 빈도, 아이템 갱신 일자, 접속 수등을 고려하여 최적의 정렬

### 21-1 NoMatch 비율과 키워드 집계하기
  - NoMatch 비율 = 검색 결과가 0 인 수 / 총 검색 수
  - NoMatch 비율을 집계하는 쿼리

    ```sql
    SELECT
    -- PostgreSQL, Hive, Redshift, SparkSQL의 경우 substring으로 날짜 추출
    substring(stamp, 1, 10) AS dt
    -- PostgreSQL, Hive, BigQuery, SparkSQL의 경우 substr 사용
    substr(stamp, 1, 10) AS dt
    , COUNT(1) AS search_count
    , SUM(CASE WHEN result_num = 0 THEN 1 ELSE 0 END) AS no_match_count
    , AVG(CASE WHEN result_num = 0 THEN 1.0 ELSE 0.0 END) AS no_match_rate
    FROM
    access_log
    WHERE
    action = 'search'
    GROUP BY
    -- PostgreSQL, Redshift, BigQuery
    -- SELECT 구문에서 정의한 별칭을 GROUP BY에서 지정 가능
    dt
    -- PostgreSQL, Hive, Redshift, SparkSQL
    -- SELECT 구문에서 별칭을 지정하기 이전의 식을 GROUP BY로 지정할 수 있음
    substring(stamp, 1, 10)
    ;
    ```

  - NoMatch 키워드를 집계하는 쿼리

    ```sql
    WITH
    search_keyword_stat AS (
    -- 검색 키워드 전체 집계 결과
    SELECT
        keyword
        , result_num
        , COUNT(1) AS search_count
        , 100.0 * COUNT(1) / COUNT(1) OVER() AS search_share
    FROM
        access_log
    WHERE
        action = 'search'
    GROUP BY
        keyword, result_num
    )
    -- NoMatch 키워드 집계 결과
    SELECT
    keyword
    , search_count
    , search_share
    , 100.0 * search_count / SUM(search_count) OVER() AS no_match_share
    FROM
    search_keyword_stat
    WHERE
    -- 검색 결과가 0개인 키워드만 추출
    result_num = 0
    ```

- 키워드 기반 검색을 기본으로 했지만 카테고리성 검색에서도 `NoMatch` 비율은 중요한 지표가 될 수 있음


### 21-2 재검색 비율과 키워드 집계하기

- 재검색 = 어떤 결과도 클릭하지 않고 새로 검색한 실행

- 검색 화면과 상세 화면의 접근 로그에 다음 줄의 액션을 기록하는 쿼리

    ```sql
    WITH
    access_log_With_next_action AS (
    SELECT
        stamp
        , session
        , action
        , LEAD(action)
        -- PostgreSQL, Hive, Redshift, BigQuery의 경우
        OVER(PARTITION BY session ORDER BY stamp ASC)
        -- SparkSQL, Frame 지정 필요
        OVER(PARTITION BY session ORDER BY stamp ASC
        ROWS BETWEEN 1 FOLLOWING AND 1 FOLLOWING)
            AS next_action
    FROM
        access_log
    )
    SELECT *
    FROM access_log_with_next_Action
    ORDER BY
    session, stamp
    ;
    ```

- 재검색 비율을 집계하는 쿼리

    ```sql
    WITH
    access_log_with_next_action AS (
    -- CODE.21.3.
    )
    SELECT
    -- PostgreSQL, Hive, Redshift, SparkSQL, substring으로 날짜 부분 추출
    substring(stamp, 1, 10) AS dt
    -- PostgreSQL, Hive, BigQuery, SparkSQL, substr 사용
    , substr(stamp, 1, 10) AS dt
    , COUNT(1) AS search_count
    , SUM(CASE WHEN next_action = 'search' THEN 1 ELSE 0 END) AS retry_count
    , AVG(CASE WHEN next_action = 'search' THEN 1.0 ELSE 0.0 END) AS retry_rate
    FROM
    access_log_with_next_action
    WHERE
    action = 'search'
    GROUP BY
    -- PostgreSQL, Redshift, BigQuery
    -- SELECT 구문에서 정의한 별칭을 GROUP BY 지정 가능
    dt
    -- PostgreSQL, Hive, Redshift, SparkSQL의 경우
    -- SELECT 구문에서 별칭을 지정하기 이전의 식을 GROUP BY 지정 가능
    substring(stamp, 1, 10)
    ORDER BY
    dt
    ;
    ```

- 재검색 키워드 집계하기
  - `어벤저스3` <-> `어벤저스: 인피니트 워` 같은 동의어 사전 키워드를 찾을 수 있음

    ```sql
    WITH
    access_log_with_next_search AS (
    SELECT
        stamp
        , session
        , action
        , keyword
        , result_num
        , LEAD(action)
        -- PostgreSQL, Hive, Redshift, BigQuery
        OVER(PARTITION BY session ORDER BY stamp ASC)
        -- SparkSQL, 프레임 지정
        OVER(PARTITION BY session ORDER BY stamp ASC
            ROWS BETWEEN 1 FOLLOWING AND 1 FOLLOWING)
        AS next_action
        , LEAD(keyword)
        -- PostgreSQL, Hive, Redshift, BigQuery
        OVER(PARTITION BY session ORDER BY stamp ASC)
        -- SparkSQL, 프레임 지정
        OVER(PARTITION BY session ORDER BY stamp ASC
            ROWS BETWEEN 1 FOLLOWING AND 1 FOLLOWING)
        AS next_keyword
        , LEAD(result_num)
        -- PostgreSQL, Hive, Redshift, BigQuery
        OVER(PARTITION BY session ORDER BY stamp ASC)
        -- SparkSQL, 프레임 지정
        OVER(PARTITION BY session ORDER BY stamp ASC
            ROWS BETWEEN 1 FOLLOWING AND 1 FOLLOWING)
        AS next_result_num
    FROM
        access_log
    )
    SELECT
    keyword
    , result_num
    , COUNT(1) AS retry_count
    , next_keyword
    , next_result_num
    FROM
    access_log_with_next_search
    WHERE
    action = 'search'
    AND next_action = 'search'
    GROUP BY
    keyword, result_num, next_keyword, next_result_num
    ```

- 재검색 키워드를 집계하고, 검색 시스템이 자동으로 이러한 흔들림을 제거하게 개선하는 방법도 고려

### 21-3 재검색 키워드를 분류해서 집계하기

- 재검색을 했을 경우 어떤 상태와 동기로 재검색을 했는지 사용자의 패턴 분석해보기
  - Nomatch에서의 조건 변경: 결과가 나오지 않아 다른 검색어로 검색
  - 검색 필터링: 단어를 필터링
  - 검색 키워드 변경: 다른 검색어로 다시 검샘
- Nomatch에서 재검색 키워드를 집계하는 쿼리
  - 동의어 사전과 사용자 사전에 추가할 키워드 후보들

    ```sql
    WITH
    access_log_with_next_search AS (
    -- CODE.21.5
    )
    SELECT
    keyword
    , result_num
    , COUNT(1) AS retry_count
    , next_keyword
    , next_result_num
    FROM
    access_log_with_next_search
    WHERE
    action = 'search'
    AND next_Action = 'search'
    -- NoMatch 로그만 필터링하기
    AND result_num = 0
    GROUP BY
    keyword, result_num, next_keyword, next_result_num
    ```

- 검색 결과 필터링 시의 재검색 키워드를 집계하는 쿼리
  - 연관 검색어 등으로 출력

    ```sql
    WITH
    access_log_with_next_search AS (
    -- CODE.21.5.
    )
    SELECT
    keyword
    , result_num
    , COUNT(1) AS retry_count
    , next_keyword
    , next_result_num
    FROM
    access_log_with_next_search
    WHERE
    action = 'search'
    AND next_action = 'search'
    -- 원래 키워드를 포함하는 경우만 추출하기
    -- PostgreSQL, Hive, BigQuery, SparkSQL, concat 함수 사용
    AND next_keyword LIKE concat('%', keyword, '%')
    -- PostgreSQL, Redshift, || 연산자 사용
    AND next_keyword LIKE '%' || keyword || '%'
    GROUP BY
    keyword, result_num, next_keywrod, next_result_num
    ;
    ```

- 검색 키워드 변경
  - 동의어 사전의 기능을 제대로 못함

    ```sql
    WITH
    access_log_with_next_search AS (
    -- CODE.21.5.
    )
    SELECT
    keyword
    , result_num
    , COUNT(1) AS retry_count
    , next_keyword
    , next_result_num
    FROM
    access_log_with_next_search
    WHERE
    action = 'search'
    AND next_action = 'search'
    -- 원래 키워드를 포함하지 않는 검색 결과만 추출
    -- PostgreSQL, Hive, BigQuery, SparkSQL, concat 함수 사용
    AND next_keyword NOT LIKE concat('%', keyword, '%')
    -- PostgreSQL, Redshift, || 연산자 사용
    AND next_keyword NOT LIKE '%' || keyword || '%'
    GROUP BY
    keyword, result_num, next_keyword, next_result_num
    ;
    ```

### 21-4 검색 이탈 비율과 키워드 집계하기

- 검색 이탈 비율을 집계하는 쿼리

    ```sql
    WITH
    access_log_With_next_action AS (
    -- CODE.21.9
    )
    SELECT
    -- PostgreSQL, Hive, Redshift, SparkSQL, substring으로 날짜 추출
    substring(stamp, 1, 10) AS dt
    -- PostgreSQL, Hive, BigQuery, SparkSQL, substr 사용
    substr(stamp, 1, 10) AS dt
    , COUNT(1) AS search_count
    , SUM(CASE WHEN next_action IS NULL THEN 1 ELSE 0 END) AS exit_count
    , AVG(CASE WHEN next_action IS NULL THEN 1.0 ELSE 0.0 END) AS exit_rate
    FROM
    access_log_with_next_action
    WHERE
    action = 'search'
    GROUP BY
    -- PostgreSQL, Redshift, BigQuery
    -- SELECT 구문에서 정의한 별칭을 GROUP BY에 지정 가능
    dt
    -- PostgreSQL, Hive, Redshift, SparkSQL
    -- SELECT 구문에서 별칭을 지정하기 이전의 식을 GROUP BY에 지정 가능
    substring(stamp, 1, 10)
    ORDER BY
    dt
    ;
    ```

- 검색 이탈 키워드 집계하기

    ```sql
    WITH
    access_log_with_next_search AS (
    -- CODE.21.5
    )
    SELECT
    keyword
    , COUNT(1) AS search_count
    , SUM(CASE WHEN next_action IS NULL THEN 1 ELSE 0 END) AS exit_count
    , AVG(CASE WHEN next_action IS NULL THEN 1.0 ELSE 0.0 END) AS exit_rate
    , result_num
    FROM
    access_log_with_next_search
    WHERE
    action='search'
    GROUP BY
    keyword, result_num
    -- 키워드 전체의 이탈률을 계산한 후, 이탈률이 0보다 큰 키워드만 추출하기
    HAVING
    SUM(CASE WHEN next_action IS NULL THEN 1 ELSE 0 END) > 0
    ```

### 21-5 검색 키워드 관련 지표의 집계 효율화하기

- 검색과 관련된 지표를 집계하기 쉽게 중간 데이터를 생성하는 쿼리

    ```sql
    WITH
    access_log_with_next_search AS (
    -- CODE.21.5
    )
    , search_log_with_next_action (
    SELECT *
    FROM
        access_log_with_next_search
    WHERE
        action = 'search'
    )
    SELECT *
    FROM search_log_with_next_action
    ORDER BY
    session, stamp
    ;
    ```

### 21-6 검색 결과의 포괄성을 지표화하기

- 검색 엔진 자체의 정밀도를 평가
- 정답 아이템 테이블을 미리 둠
- 검색 결과와 정답 아이템을 결합하는 쿼리
  - 재현율(Recall) = 키워드의 검색 결과에서 정답이 얼마나 나왔는가

    ```sql
    WITH
    search_result_with_correct_items AS (
    SELECT
        COALESCE(r.keyword, c.keyword) AS keyword
        , r.rank
        , COALESCE(r.item, c.item) AS item
        , CASE WHEN c.item IS NOT NULL THEN 1 ELSE 0 END AS correct
    FROM
        search_result AS r
        FULL OUTER JOIN
        correct_result AS c
        ON r.keyword = c.keyword
        AND r.item = c.item
    )
    SELECT *
    FROM
    search_Result_with_correct_items
    ORDER BY
    keyword, rank
    ;
    ```

- 검색 결과 상위 n개의 재현율을 계산하는 쿼리
  - `correct` `SUM`

    ```sql
    WITH
    search_result_with_correct_items AS (
    -- CODE.21.13.
    )
    , search_result_with_recall AS (
    SELECT
        *
        -- 검색 결과 상위에서, 정답 데이터에 포함되는 아이템 수의 누계 계산
        , SUM(corret)
        -- rank=NULL, 아이템의 정렬 순서에 마지막에 위치
        -- 편의상 가장 큰 값으로 변환
        OVER(PARTITION BY keyword ORDER BY COALESCE(rank, 100000) ASC
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS cum_correct
        , CASE
        -- 검색 결과에 포함되지 않은 아이템은 편의상 적합률을 0으로 다루기
        WHEN rank IS NULL THEN 0.0
        ELSE
            100.0
            * SUM(correct)
                OVER(PARTITION BY keyword ORDER BY COALESCE(rank, 100000) ASC
                ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
                / SUM(correct) OVER(PARTITION BY keyword)
            END AS recall
        FROM
        search_result_with_correct_items
    )
    SELECT *
    FROM
    search_result_with_recall
    ORDER BY
    keyword, rank
    ;
    ```

- 재현율의 값을 집약해서 비교하기 쉽게 만들기
  - 위의 값으로는 파악하기 힘들어 재현되는 아이템이 몇 개인지 구하는 것이 좋음
  - 검색 결과 상위 5개의 재현율을 키워드별로 추출하는 쿼리

    ```sql
    WITH
    search_result_with_correct_items AS (
    -- CODE.21.13
    )
    , search_result_with_recall AS (
    -- CODE.21.14
    )
    , recall_over_rank_5 AS (
    SELECT
        keyword
        , rank
        , recall
        -- 검색 결과 순위가 높은 순서로 번호 붙이기
        -- 검색 결과에 나오지 않는 아이템은 편의상 0으로 다루기
        , ROW_NUMBER()
            OVER(PARTITION BY keyword ORDER BY COALESCE(rank, 0) DESC)
        AS desc_number
    FROM
        search_result_with_recall
    WHERE
        -- 검색 결과 상위 5개 이하 또는 검색 결과에 포함되지 않은 아이템만 출력
        COALESCE(rank, 0) <= 5
    )
    SELECT
    keyword
    , recall AS recall_at_5
    FROM recall_over_rank_5
    -- 검색 결과 상위 5개 중에서 가장 순위가 높은 레코드 추출하기
    WHERE desc_number = 1
    ;
    ```

  - 검색 엔진 전체의 평균 재현율을 계산하는 쿼리

    ```sql
    WITH
    search_result_with_correct_items AS (
    -- CODE.21.13
    )
    , search_result_with_recall AS (
    -- CODE.21.14
    )
    , recall_over_rank_5 AS (
    -- CODE.21.15
    )
    SELECT
    avg(recall) AS average_recall_at_5
    FROM recall_over_rank_5
    -- 검색 결과 상위 5개 중에서 가장 순위가 높은 레코드 추출하기
    WHERE desc_number = 1
    ;
    ```

### 21-7 검색 결과의 타당성을 지표화하기

- 정확률(Precision): 검색 결과에 포함되는 아이템 중 정답 아이템의 비율
- 정확률을 사용해 검색의 타당성 평가하기
- 기본적으로는 재현율과 쿼리가 같으나, 분모 부분만 검색 결과 순위 까지의 누계 아이템 수로 바뀜

    ```sql
    WITH
    search_result_with_Correct_items AS (
    -- CODE.21.13
    )
    , search_result_with_precision AS (
    SELECT
        *
        -- 검색 결과의 상위에서 정답 데이터에 포함되는 아이템 수의 누계 구하기
        , SUM(correct)
        -- rank가 NULL이라면 정렬 순서의 마지막에 위치하므로
        -- 편의상 굉장히 큰 값으로 변환하기
        OVER(PARTITION BY keyword ORDER BY COALESCE(rank, 100000) ASC
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS cum_correct
        , CASE
        -- 검색 결과에 포함되지 않은 아이템은 편의상 적합률을 0으로 다루기
            WHEN rank IS NULL THEN 0.0
            ELSE
            100.0
            * SUM(correct)
                OVER(PARTITION BY keyword ORDER BY COALESCE(rank, 100000) ASC
                    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
            -- 재현률과 다르게, 분모에 검색 결과 순위까지의 누계 아이템 수 지정하기
            / COUNT(1)
                OVER(PARTITION BY keyword ORDER BY COALESCE(rank, 100000) ASC
                    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
            END AS precision
    FROM
        search_result_with_correct_items
    )
    SELECT *
    FROM
    search_result_with_precision
    ORDER BY
    keyword, rank
    ;
    ```

- 정확률 값을 집약해서 비교하기 쉽게 만들기

    ```sql
    WITH
    search_result_with_correct_items AS (
    -- CODE.21.13
    )
    , search_result_with_precision AS (
    -- CODE.21.17
    )
    , precision_over_rank_5 AS (
    SELECT
        keyword
        , rank
        , precision
        -- 검색 결과 순위가 높은 순서로 번호 붙이기
        -- 검색 결과에 나오지 않는 아이템은 편의상 0으로 다루기
        , ROW_NUMBER()
            OVER(PARTITION BY keyword ORDER BY COALESCE(rank, 0) DESC) AS desc_number
    FROM
        search_result_with_precision
    WHERE
        -- 검색 결과의 상위 5개 이하 또는 검색 결과에 포함되지 않는 아이템만 출력하기
        COALESCE(rank, 0) <= 5
    )
    SELECT
    keyword
    , precision AS precision_at_5
    FROM precision_over_rank_5
    -- 검색 결과의 상위 5개 중에서 가장 순위가 높은 레코드만 추출하기
    WHERE desc_number = 1;
    ```

- 검색 엔진 전체의 평균 정확률을 계산하는 쿼리

    ```sql
    WITH
    search_Result_With_correct_items AS (
    -- CODE.21.13
    )
    , search_Result_with_precision AS (
    -- CODE.21.17
    )
    , preceision_over_rank_5 AS (
    -- CODE.21.18
    )
    SELECT
    AVG(precision) AS average_precision_at_5
    FROM precision_over_rank_5
    -- 검색 결과 상위 5개 중에서 가장 순위가 높은 레코드만 추출하기
    WHERE desc_number=1
    ;
    ```

### 검색 결과 순위와 관련된 지표 계산하기
- 재현율과 정확률의 부족한 부분
  - 검색 결과의 순위는 고려하지 않음
    - => 검색 순위를 고려한 지표로는 MAP(Mean Average Precision)과 MRR(Mea Reciprocal Rank) 등이 있음
  - 0과 1만으로 정답 구분
    - => 단계적인 점수를 고려해서 정답 아이템을 다루는 지표로는 DCG(Discounted Cumulated Gain)와 NDCG(Normalized DCG)
  - 모든 아이템의 정답을 미리 준비하는 것은 사실 불가능에 가까움
- MAP(Mean Average Precision)로 검색 결과의 순위를 고려해 평가하기
  - 검색 결과 상위 N개의 적합률 평균
  - 예시
    - 정답 아이템 수가 4개라고 할때, P@10 = 40%
    - 상위 1~4번째가 모두 정답 아이템 => MAP = 100 * ((1/1) + (2/2) + (3/3) + (4/4))/4 = 100으로 계산
    - 상위 7~10번째가 정답 아이템 => MAP = 100 * ((1/7) + (2/8) + (3/9) + (4/10))/4 = 28.15
- 이전 쿼리에서 correct 컬럼의 플래그가 1인 레코드만 추출하면 됨
- 정답 아이템 별로 적합률을 추출하는 쿼리

    ```sql
    WITH
    search_result_with_correct_items AS (
    -- CODE.21.13.
    )
    , search_result_with_precision AS (
    -- CODE.21.17
    )
    SELECT
    keyword
    , rank
    , precision
    FROM
    search_result_with_precision
    WHERE
    correct = 1
    ;
    ```

- 검색 키워드별로 정확률의 평균을 계산하는 쿼리

    ```sql
    WITH
    search_result_with_correct_items AS (
    -- CODE.21.13
    )
    , search_result_with_precision AS (
    -- CODE.21.17
    )
    , average_precision_for_keywords AS (
    SELECT
    keyword
    , AVG(precision) AS average_precision
    FROM
    search_result_with_precision
    WHERE
    correct = 1
    GROUP BY
    keyword
    )
    SELECT *
    FROM
    average_precision_for_keywords
    ;
    ```

- 검색 엔진의 MAP를 계산하는 쿼리

```sql
WITH
search_result_with_correct_itmes AS (
-- CODE.21.13
)
, search_result_with_precision AS (
-- CODE.21.17
)
, average_precision_for_keywords AS (
-- CODE.21.21
)
SELECT
  AVG(average_precision) AS mean_average_precision
FROM
  average_precision_for_keywords
;
```

- 검색 평과와 관련한 다양한 지표들
- ![검색 평가에 사용되는 대표적인 순위 지표](https://user-images.githubusercontent.com/37397737/201526398-3a3ffaa8-2db6-452b-b7f7-1b755673b014.png)


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
    ;
    ```
- 어소시에이션 분석은 두 상품의 상관 규칙만을 주목한 한정적인 분석 방법

## 23강 추천

- 사용자에게 가치 있는 정보를 추천하는 것
- item to item, user to item

### 23-1 추천 시스템의 넓은 의미

- 추천 시스템의 종류
  - Item to Item: 아이템과 관련한 개별적인 아이템 제안. 열람/구매한 아이템을 기반으로 다른 아이템 추천
  - User to Item: 사용자 개인에 최적화된 아이템 제안. 과거의 행동 또는 데모그래픽 정보를 기반으로 흥미와 기호를 유추하여 아이템 추천
- 모듈의 종류
  - 리마인드: 사용자의 과거 행동을 기반으로 아이템을 다시 제안해주는 것 ex) 재구매, 최근 본상품
  - 순위: 열람 수, 구매 수 등을 기반으로 인기있는 아이템 제안
  - 콘텐츠베이스: 아이템의 추가 정보를 기반으로 아이템 추천 ex) 해당 배우가 출연한 다른 작품
  - 추천: 사용자 전체의 행동 이력을 기반으로 아이템 제안
  - 개별 추천: 사용자 개인의 행동 이력을 기반으로 아이템 제안
- 추천의 효과
  - 다운셀: 가격이 높아 고민하는 구매자에게 더 저렴한 아이템 제안하여 판매 증가
  - 크로스셀: 관련 상품을 함께 구매하게 해서 구매 단가를 올림
  - 업셀: 상웨 모델이나 고성능 아이템을 제안해서 구매 단가를 올림
- 데이터의 명시적 획득과 암묵적 획득
  - 명시적 데이터 획득: 사용자에게 직접 기호를 물어봄. 데이터 양은 적지만 정확성은 높음 ex) 리뷰
  - 암묵적 데이터 획득: 사용자의 행동을 기반으로 시호를 추측. 데이터 양은 많지만 정확도가 떨어짐 ex) 구매, 열람 로그
- 추천 시스템에는 다양한 목적, 효과, 모듈이 있기 때문에 어떤 효과를 기대하는지 구체화를 한 후 시스템 구축하는 것을 추천!

### 23-2 특정 아이템에 흥미가 있는 사람이 함께 찾아보는 아이템 검색

- Item to Item
  - User to Item보다 상대적으로 쉬움. 사용자 유동성보다 아이템 유동성이 더 낮아 데이터 축적이 쉽기 때문
- 접근 로그를 사용해 아이템의 상관도 계산하기
  - 로그를 기반으로 사용자와 아이템 조합을 구하고 점수를 계산하는 쿼리
  - 아이템 열람수:구매수 를 3:7로 가중치를 줘서 평균을 구한후 `관심도 점수`로 활용
  - 열람 수와 구매수를 조합한 점수를 계산하는 쿼리

    ```sql
    WITH
    ratings AS (
    SELECT
        user_id
        , product
        -- 상품 열람 수
        , SUM(CASE WHEN action = 'view' THEN 1 ELSE 0 END) AS view_count
        -- 상품 구매 수
        , SUM(CASE WHEN action = 'purchase' THEN 1 ELSE 0 END) AS purchase_count
        -- 열람 수 : 구매 수 = 3:7 비율의 가중치 주어 평균 계산
        , 0.3 * SUM(CASE WHEN action='view' THEN 1 ELSE 0 END)
        + 0.7 * SUM(CASE WHEN action='purchase' THEN 1 ELSE 0 END)
        AS score
    FROM
        action_log
    GROUP BY
        user_id, product
    )
    SELECT *
    FROM
    ratings
    ORDER BY
    user_id, score DESC
    ;
    ```

- 아이템 사이의 유사도를 계산하고 순위를 생성하는 쿼리

    ```sql
    WITH
    ratings AS (
    -- CODE.23.1.
    )
    SELECT
    r1.product AS target
    , r2.product AS related
    -- 모든 아이템을 열람/구매한 사용자 수
    , COUNT(r1.user_id) AS users
    -- 스코어들을 곱하고 합계를 구해 유사도 계산
    , SUM(r1.socre * r2.score) AS score
    -- 상품의 유사도 순위 계산
    , ROW_NUMBER()
        OVER(PARTITION BY r1.product ORDER BY SUM(r1.score * r2.score) DESC)
        AS rank
    FROM
    ratings AS r1
    JOIN
        ratings AS r2
        -- 공통 사용자가 존재하는 상품의 페어 만들기
        ON r1.user_id = r2.user_id
    WHERE
    -- 같은 아이템의 경우에는 페어 제외하기
    r1.product <> r2.product
    GROUP BY
    r1.product, r2.product
    ORDER BY
    target, rank
    ;
    ```

- 점수 정규화하기
  - 벡터 내적은 정밀도에 문제가 있음 => `벡터 정규화` = 벡터를 모두 같은 길이로 만든다는 의미
    - 접근 수가 많은 아이템의 유사도가 상대적으로 높게 나옴
    - 점수의 상대적인 위치 파악 불가
  - 벡터 정규화
    - 벡터이 길이 = 유클리드 거리
    - norm = 벡터의 크기
    - L2 정규화 = norm으로 벡터의 각 수치를 나누면 벡터의 norm을 1로 만들 수 있음
    - 아이템 벡터를 L2 정규화하는 쿼리
      - `SUM` `OVER` `SQRT` 함수를 사용

    ```sql
    WITH
    ratings AS (
    -- CODE.23.1.
    )
    , product_base_normalized_ratings AS (
    -- 아이템 벡터 정규화하기
    SELECT
        user_id
        , product
        , score
        , SQRT(SUM(score * score) OVER(PARTITION BY product)) AS norm
        , score / SQRT(SUM(score * score) OVER(PARTITION BY product)) AS norm_score
    FROM
        ratings
    )
    SELECT *
    FROM
    prodduct_base_normalized_ratings
    ;
    ```

    - 정규화된 점수로 아이템의 유사도를 계산하는 쿼리
      - 자기 자신과 유사도는 1.0 임
      - 코사인 유사도 = 벡터를 L2정규화해서 내적한 값
    
    ```sql
    WITH
    ratings AS (
    -- CODE.23.1.
    )
    , product_base_normalized_ratings AS (
    -- CODE.23.3.
    )
    SELECT
    r1.product AS target
    , r2.product AS related
    -- 모든 아이템을 열람/구매 한 사용자 수
    , COUNT(r1.user_id) AS users
    -- 스코어들을 곱하고 합계를 구해 유사도 계산하기
    , SUM(r1.norm_score * r2.norm_score) As score
    -- 상품의 유사도 순위 구하기
    , ROW_NUMBER()
        OVER(PARTITION BY r1.product ORDER BY SUM(r1.norm_score * r2.norm_score) DESC)
        AS rank
    FROM
    product_base_normalized_ratings AS r1
    JOIN
        product_base_normalized_ratings AS r2
        -- 공통 사용자가 존재하면 상품 페어 만들기
        ON r1.user_id = r2.user_id
    GROUP BY
    r1.product, r2.product
    ORDER BY
    target, rank
    ;
    ```

### 23-3 당신을 위한 추천 상품

- 사용자와 관련된 추천이여서 페이지나 메시지 알림 등 다양하게 활용할 수 있음
- 사용자끼리의 유사도를 계산하는 쿼리

```sql
WITH
ratings AS (
  -- CODE.23.1.
)
, user_base_normalized_ratings AS (
  -- 사용자 벡터 정규화하기
  SELECT
    user_id
    , product
    , score
    -- PARTITION BY user_id으로 사용자별 벡터 노름 계산하기
    , SORT(SUM(score * score) OVER(PARTITION BY user_id)) AS more
    , score / SORT(SUM(score * score) OVER(PARTITION BY user_id)) AS norm_score
  FROM
    ratings
)
, related_users AS (
  -- 경향이 비슷한 사용자 찾기
  SELECT
    r1.user_id
    , r2.user_id AS related_user
    , COUNT(r1.product) AS products
    , SUM(r1.norm_score * r2.norm_score) AS score
    , ROW_NUMBER()
        OVER(PARTITION BY r1.user_id ORDER BY SUM(r1.norm_score * r2.norm_score) DESC) AS rank
  FROM
    user_base_normalized_ratings AS r1
    JOIN
      user_base_normalized_ratings AS r2
      ON r1.product = r2.product
  WHERE
    r1.user_id <> r2.user_id
  GROUP BY
    r1.user_id, r2.user_id
)
SELECT *
FROM
  related_users
ORDER BY
  user_id, rank
;
```

- 순위가 높은 유사 사용자를 기반으로 추천 아이템을 추출하는 쿼리
  - 유사도 상위 N명 추출
  - 기존 구매한 상품은 제외

```sql
WITH
ratings AS (
  -- CODE.23.1.
)
, user_base_normalized_ratings AS (
  -- CODE.23.5.
)
, related_users AS (
  -- CODE.23.5.
)
, related_user_base_products AS (
  SELECT
    u.user_id
    , r.product
    , SUM(u.score * r.score) * AS score
    , ROW_NUMBER()
        OVER(PARTITION BY u.user_id ORDER BY SUM(u.score * r.score) DESC)
    AS rank
  FROM
    related_users AS u
    JOIN
      ratings AS r
      ON u.related_user = r.user_id
  WHERE
    u.rank <= 1
  GROUP BY
    u.user_id, r.product
)
SELECT *
FROM
  related_user_base_products
ORDER BY
  user_id
;
``` 
- 이미 구매한 아이템을 필터링하는 쿼리
  - `LEFT JOIN` 해서 아이템의 구매가 0 또는 NULL인 아이템을 압축한 뒤 순위 생성

```sql
WITH
ratings AS (
  -- CODE.23.1.
)
, user_base_normalized_ratings AS (
  -- CODE.23.5.
)
, related_suers AS (
  -- CODE.23.5.
)
, related_user_base_products AS (
  -- CODE.23.6.
)
SELECT
  p.user_id
  , p.product
  , p.score
  , ROW_NUMBER()
      OVER(PARTITION BY p.user_id ORDER BY p.score DESC) AS rank
FROM
  related_user_base_products AS p
  LEFT JOIN
    ratings AS r
    ON p.user_id = r.user_id
    AND p.product = r.product
WHERE
  -- 대상 사용자가 구매하지 않은 상품만 추천하기
  COALESCE(r.purchase_count, 0) = 0
ORDER BY
  p.user_id
;
```

- `User to Item`의 경우 데이터가 충분하지 않으면 예측하기 어려움. 따라서, 등록한 지 얼마 안된 사용자는 다른 추천 로직(랭킹 또는 콘텐츠 기반)을 적용하는 것이 좋음
- 게스트 사용자 전체를 한명으로 두고 대응하는 방법도 있음

### 23-4 추천 시스템을 개선할 때의 포인트

- 지속적으로 운용할 때 어떻게 하면 정밀도를 높일 수 151512313123122312312321지의 관점
- 값과 리스트 조작에서 개선할 포인트
  - 가중치
    - 열람 로그 1포인트, 구매 로그 3포인트 처럼 가중치 두기
  - 필터
    - 스팸 공격 같은 비정상 로그나 카테고리나 지역등을 제한
  - 정렬
    - 어떤 점수를 내고 정렬할지 고민하기. 목적(신규, 기대구매수 등)에 따라 정렬해서 추천을 제공
  - 예시로는 별점이 높지만 거리가 먼 음식점의 경우 거리/별점으로 점수를 매겨 추천할 수 있음
- 구축 방법에 따른 개선 포인트
  - 데이터 수집
    - 다양한 데이터를 수집하여 정밀도를 높이기
  - 데이터 가공
    - 데이터를 계산하기 쉬운 상태로 가공하기
    - 비정상적인 데이터 필터링
    - 평가기간을 두거나 가중치 제한을 두기
  - 데이터 계산
    - 순위를 구하는 로직으로, 어떠한 로직을 적용할지 적용해보기
    - 때로는 간단한 매출 순서나 데모그래픽 정보등에 따라 점수계산해도 좋은 추천 시스템이 나올 수 있음
  - 데이터 정렬
    - 계산한 결과를 바로 활용하기 보다는 필터링이나 정렬 처리등을 추가해서 더 효율적으로 만들기
    - 데모그래픽 정보 활용

### 23-5 출력할 때 포인트

- 출력 페이지, 위치, 시점 검토하기
  - 추천출력이 상품보다 위에 있을 경우 추천을 더 중요시, 상품보다 아래 있을 경우 이탈을 방지
  - ex) 장바구니 화면에서 상품 추천은 이탈 막기와 새로운 선택지를 주는 추천방법
- 추천의 이유
  - '구매한 사람이 이러한 상품도 구매했다', '어떤 상품을 기반으로 추천합니다' 등 추천이유를 제공해야 효과가 있음
  - 일반적인 사람들은 인지편향과 편승효과가 있음
- 크로스셀을 염두한 추천하기
  - 같이 자주 구매하는 제품을 함께 장바구니에 담을 수 있게 하기
- 서비스와 함께 제공하기
  - '다음과 같은 상품을 구매하면 무료배송 서비스 제공' 출력

### 23-6 추천과 관련한 지표

- 추천시스템의 대표적인 평가 지표 목록
  - Microsofet Research에서 shani guy가 작성한 `Evaluating Recommender`
  - ![추천시스템 평가지표](https://miro.medium.com/max/1248/1*5L_T_-yH1yr-aEX5_tcPNA.png)

## 24강 점수 계산하기

---

### 24-1 여러 값을 균형있게 조합해서 점수 계산하기

- 재현율, 적합률 같이 트레이드 오프 관계가 있는 점수의 경우 조합해서 사용해야 함
- 평균 종류
  - 산술 평균
    - 일반적인 평균
  - 기하 평균
    - 값을 곱한뒤 개수만콤 제곱근을 걸은 값. 각각의 값은 양수여야 함
    - 어떤 지표를 여러번 곱해야하는 경우 기하 평균이 적합 ex) 월 이자  
  - 조화 평균
    - 각 값의 역수의 산술평균을 구하고 다시 역수를 취함
    - 비율을 나타내는 값의 평균을 계산할 때 유용. ex) 평균 속도 계산
- 평균 계산
  - where절에서 평균을 못구하는 레코드는 제거
  - `sqrt`함수가 아닌 여러값에 대응할 수 있는`power`함수의 매개변수에 1/2 넣어 계산
  - 두 값의 데이터가 다를 경우 `산술 > 기하 > 조화` 순이다.
  - 세로 데이터의 평균을 계산하는 쿼리

    ```sql
    SELECT
    *
    -- 산술 평균
    , (recall + precision) / 2 AS arithmetic_mean
    -- 기하 평균
    , POWER(recall * precision, 1.0 / 2) AS geometric_mean
    -- 조화 평균
    , 2.0 / ((1.0 / recall) + (1.0 / precision)) AS harmonic_mean
    FROM
    search_evaluation_by_col
    -- 값이 0보다 큰 것만으로 한정하기
    WHERE recall * precision > 0
    ORDER BY path
    ;
    ```

  - 가로 기반 데이터의 평균 계산하는 쿼리
  
    ```sql
    SELECT
    path
    -- 산술 평균
    , AVG(value) AS arithmetic_mean
    -- 기하 평균(대수의 산술 평균)
    -- PostgreSQL, Redshift, 상용 로그로 log함수 사용하기
    , POWER(10, AVG(log(value))) AS geometric_mean
    -- Hive, BigQuery, SparkSQL, 상용 로그로 log10 함수 사용
    , POWER(10, AVG(log10(value))) AS geometric_mean
    -- 조화 평균
    , 1.0 / (AVG(1.0 / value)) AS harmonic_mean
    FROM
    search_evaluation_by_row
    -- 값이 0보다 큰 것만으로 한정
    WHERE value > 0
    GROUP BY path
    -- 빠진 데이터가 없게 path로 한정
    HAVING COUNT(*) = 2
    GROUP BY path
    ;
    ```

- f1 score = 재현율과 적합률의 조화 평균
- 세로 기반 데이터의 가중 평균을 계산하는 쿼리 (재현3:7적합)

```sql
SELECT
  *
  -- 가중치가 추가된 산술 평균
  , 0.3 * recall + 0.7 * preicision AS weighted_a_mean
  -- 가중치가 추가된 기하 평균
  , POWER(recall, 0.3) * POWER(precision, 0.7) AS weighted_g_mean
  -- 가중치가 추가된 조화 평균
  , 1.0 / ((0.3) / recall) + (0.7 / precision)) AS weighted_h_mean
FROM
  search_evaluation_by_col
-- 값이 0보다 큰 것만으로 한정
WHERE recall * precision > 0
ORDER BY path
;
```

- 가로 기반 테이블의 가중 평균을 계산하는 쿼리
  - 임시 테이블을 만들어 조인하여 사용
  - 레코드수로 나누는 `AVG` 대신  `SUM` 함수 사용

```sql
WITH
weights AS (
  -- 가중치 마스터 테이블(가중치의 합계가 1.0이 되도록 설정)
            SELECT 'recall'   AS index, 0.3 AS weight
  UNION ALL SELECT 'precision' AS index, 0.7 AS weight
)
SELECT
  e.path
  -- 가중치가 추가된 산술 평균
  , SUM(w.weight * e.value) AS weighted_a_mean
  -- 가중치가 추가된 기하 평균
  -- PosgreSQL, Redshift, log 사용
  , POWER(10, SUM(w.weight * log(e.value))) AS weighted_g_mean
  -- Hive, BigQuery, SparkSQL, log10 함수 사용
  , POWER(10, SUM(w.weight * log10(e.value))) AS weighted_g_mean

  -- 가중치가 추가된 조화 평균
  , 1.0 / (SUM(w.weight / e.value)) AS weighted_h_mean
FROM
  search_evaluation_by_row AS e
  JOIN
    weights AS w
    ON e.index = w.index
  -- 값이 0보다 큰 것만으로 한정하기
WHERE e.value > 0
GROUP BY e.path
-- 빠진 데이터가 없도록 path로 한정
HAVING COUNT(*) = 2
ORDER BY e.path
;
```

### 24-2 값의 범위가 다른 지표를 정규화해서 비교 가능한 상태로 만들기

- 범위가 다른 지표를 결합할 때 정규화를 해주어야 함
- min-max 정규화
  - 각 지표의 최소값 최대값을 0~1의 스케일로 정규화
  - 열람 수와 구매 수에 min-max 정규화를 적용하는 쿼리
  
    ```sql
    SELECT
    user_id
    , product
    , view_count AS v_count
    , purchase_count AS p_count
    , 1.0 * (view_count - MIN(view_count) OVER())
        -- PostgreSQL, Redshift, BigQuery, SparkSQL의 경우 `NULLIF`로 0으로 나누는 것 피하기
        / NULLIF((MAX(view_count) OVER() - MIN(view_count) OVER()), 0)
        -- Hive의 경우 NULLIF 대신 CASE 사용
        / (CASE
            WHEN MAX(view_count) OVER() - MIN(view_count) OVER() = 0 THEN NULL
            ELSE MAX(view_count) OVER() - MIN(view_count) OVER()
            END
        )
        AS norm_view_count
    , 1.0 * (purchase_count - MIN(purchase_count) OVER())
        -- PostgreSQL, Redshift, BigQuery, SparkSQL의 경우 `NULLIF`로 0으로 나누는 것 피하기
        / NULLIF((MAX(purchase_count) OVER() - MIN(purchase_count) OVER()), 0)
        -- Hive의 경우 NULLIF 대신 CASE 사용
        / (CASE
            WHEN MAX(purchase_count) OVER() - MIN(purchase_count) OVER() = 0 THEN NULL
            ELSE MAX(purchase_count) OVER() - MIN(purchase_count) OVER()
            END
        )
        AS norm_p_count
    FROM action_counts
    ORDER BY user_id, product;
    ```

- 시그모이드 함수로 변화하기
  - min-max 정규화의 문제: 집단 변화에 따라 정규화 후 값이 다 바뀜
  - 0~1의 범위로 변환해주는 s자 곡선을 그리는 시그모이드 함수
  - 시그모이드 함수를 사용해 변환하는 쿼리

    ```sql
    SELECT
    user_id
    , product
    , view_count AS v_count
    , purchase_count AS p_count
    -- gain을 0.1로 사용한 sigmoid 함수
    , 2.0 / (1 + exp(-0.1 * view_count)) - 1.0 AS sigm_v_count
    -- gain을 10으로 사용한 sigmoid 함수
    , 2.0 / (1 + exp(-10 * purchase_count)) - 1.0 AS sigm_p_count
    FROM action_counts
    ORDER BY user_id, product;
    ``` 

  - 열람 수를 다룰 때 비선형 변환을 하는 것이 더 이해하기 편함

### 24-3 각 데이터의 편차값 계산하기

- 표쥰편차, 정규값, 편차값
  - 표준편차
    - 표준편차가 커질수록 데이터에 쏠림이 있다는 의미
    - 모집단의 표준편차를 구할 때
      - SQL : stddev_pop
      - 표준편차 = v((각 데이터 - 평균)^2을 모두 더한 것 / 데이터의 수)
    - 모집단의 일부를 기반으로 표준편차를 구할 때
      - SQL : sqddev
      - 표준편차 = v((각 데이터 - 평균)^2를 모두 더한것 / 데이터의 개수 - 1)
  - 정규값
    - 평균으로부터 얼마나 떨어져 있는지, 데이터의 쏠림 정도를 기반으로 점수의 가치를 평가하기 위해 변환
    - 정규값 = (각각의 데이터 - 평균) / 표준편차
    - 정규값의 평균은 0, 표준편차는 1이 됨
  - 편차값
    - 조건이 다른 데이터를 쉽게 비교할 때 사용하며, 정규값을 활용
    - 편차값 = 각 정규화된 데이터 * 10 + 50
    - 편차값의 평균은 50 표준편차는 10이 됨 
- 표준편차, 기본값, 편차값을 계산하는 쿼리

```sql
SELECT
  subject
  , name
  , score
  -- 과목별로 표준편차 구하기
  , stddev_pop(score) OVER(PARTITION BY subject) AS stddev_pop
  -- 과목별 평균 점수 구하기
  , AVG(score) OVER(PARTITION BY subject) AS avg_score
  -- 점수별로 기준 점수 구하기
  , (score - AVG(score) OVER(PARTITION BY subject))
    / stddev_pop(score) OVER(PARTITION BY subject) AS std_value
  -- 점수별로 편차값 구하기
  , 10.0 * (score - AVG(score) OVER(PARTITION BY subject))
    / stddev_pop(score) OVER(PARTITION BY subject) + 50 AS deviation
FROM exam_scores
ORDER BY subject, name;
```

- 표준편차를 따로 계산하고, 기본값과 편차값을 계산하는 쿼리

```sql
WITH
exam_stddev_pop AS (
  -- 다른 테이블에서 과목별로 표준편차 구해두기
  SELECT
    subject
    , stddev_pop(score) AS stddev_pop
  FROM exam_scores
  GROUP BY subject
)
SELECT
  s.subject
  , s.name
  , s.score
  , d.stddev_pop
  , AVG(s.score) OVER(PARTITION BY s.subject) AS avg_score
  , (s.score - AVG(s.score) OVER(PARTITION BY s.subject)) / d.stddev_pop AS std_value
  , 10.0 * (s.score - AVG(s.score) OVER(PARTITION BY s.subject)) / d.stddev_pop + 50 AS deviation
FROM
  exam_scores AS s
  JOIN
    exam_stddev_pop AS d
    ON s.subject = d.subject
ORDER BY s.subject, s.name;
```

### 24-4 거대한 숫자 지표를 직감적으로 이해하기 쉽게 가공하기

- 1과2와 차이와 10001과 10002 차이는 다름. 이럴때 `로그`를 사용해 데이터를 변환 
- 오래된 날짜의 액션일수록 가중치를 적게주는 방법
- 사용자들의 최종 접근일과 각 레코드와의 날짜 차이를 계산하는 쿼리

```sql
WITH
action_counts_with_diff_date AS (
  SELECT *
    -- 사용자별로 최종 접근일과 각 레코드의 날짜 차이 계산
    -- PostgreSQL, Redshift의 경우 날짜끼리 빼기 연산 가능
    , MAX(dt::date) OVER(PARTITION BY user_id) AS last_access
    , MAX(dt::date) OVER(PARTITION BY user_id) - dt::date AS diff_date
    -- BigQuery의 경우 date_diff 함수 사용하기
    , MAX(date(timestamp(dt))) OVER(PARTITION BY user_id) AS last_access
    , date_diff(MAX(date(timestamp(dt))) OVER(PARTITION BY user_id), date(timestamp(dt)), date) AS diff_date
    -- Hive, SparkSQL의 경우 datediff 함수 사용
    , MAX(to_date(dt)) OVER(PARTITION BY user_id) AS last_access
    , datediff(MAX(to_date(dt)) OVER(PARTITION BY user_id), to_date(dt)) AS diff_date
  FROM
    action_counts_with_date
)
SELECT *
FROM action_counts_with_diff_date;
```

- 날짜차이가 클수록 가중치를 적게하기 => 로그에 역수 취하기 
  - y = 1/(log2(ax+2)) {0 <= x}
- 날짜 차이에 따른 가중치를 계산하는 쿼리

```sql
WITH
action_counts_with_diff_date AS (
  -- CODE.24.9.
), action_counts_with_weight AS (
  SELECT *
    -- 날짜 차이에 따른 가중치 계산하기(a = 0.1)
    -- PostgreSQL, Hive, SparkSQL, log(<밑수>, <진수>) 함수 사용
    , 1.0 / log(2, 0.1 * diff_date + 2) AS weight
    -- Redshift의 경우 log는 상용 로그만 존재, `log2`로 나눠주기
    , 1.0 / ( log(CAST(0.1 * diff_date + 2 AS double precision)) / log(2) ) AS weight
    -- BigQuery의 경우, log(<진수>, <밑수>) 함수 사용
    , 1.0 / log(0.1 * diff_date + 2, 2) AS weight
  FROM action_counts_with_diff_date
)
SELECT
  user_id
  , product
  , v_count
  , p_count
  , diff_date
  , weight
FROM action_counts_with_weight
ORDER BY
  user_id, product, diff_date DESC
;
```

- 가중치를 열람 수와 구매수에 곱해 점수를 계산
- 일수차에 따른 중첩을 사용해 열람 수와 구매 수 점수를 계산하는 쿼리
  - 같은 구매수의 상품이라도 구매 날짜가 오래됐다면 점수가 낮게 나옴
 
```sql
WITH
action_counts_with_date AS (
  -- CODE.24.9.
)
, action_counts_with_weight AS (
  -- CODE.24.10.
)
, action_scores AS (
  SELECT
    user_id
    , product
    , SUM(v_count) AS v_count
    , SUM(v_count * weight) AS v_score
    , SUM(p_count) AS p_count
    , SUM(p_count * weight) AS p_score
  FROM action_counts_with_weight
  GROUP BY
    user_id, product
)
SELECT *
FROM action_scores
ORDER BY
  user_id, product;
```

### 24-5 독자적인 점수 계산 방법을 정의해서 순위 작성하기

- 순위 생성 방침
  - 1년 동안의 계절마다 주기적으로 팔리는 상품과 최근 트렌드 상품 상위에 위치
  - 1년전의 매출과 최근 1개월의 매출 값으로 가중평균을 냄
- 분기별 상품 매출액과 매출 합계를 집계하는 쿼리

```sql
WITH
item_sales_per_quarters AS (
  SELECT item
    -- 2016.1q의 상품 매출 모두 더하기
    , SUM(
      CASE WHEN year_month IN ('2016-01', '2016-02', '2016-03') THEN amount ELSE 0 END
    ) AS sales_2016_q1
    -- 2016.4q의 상품 매출 모두 더하기
    , SUM(
      CASE WHEN year_month IN ('2016-10', '2016-11', '2016-12') THEN amount ELSE 0 END
    ) AS sales_2016_q4
  FROM monthly_sales
  GROUP BY item
)
SELECT
  item
  -- 2016.1q의 상품 매출
  , sales_2016_q1
  -- 2016.1q의 상품 매출 합계
  , SUM(sales_2016_q1) OVER() AS sum_sales_2016_q1
  -- 2016.4q의 상품 매출
  , sales_2016_q4
  -- 2016.4q의 상품 매출 합계
  , SUM(sales_2016_q4) OVER() AS sum_sales_2016_q4
FROM item_sales_per_quaters
;
```

- 분기별 상품 매출액을 기반으로 점수를 계산하는 쿼리
  - min-max 정규화 사용

```sql
WITH
item_sales_per_quarters AS (
  -- CODE.24.12
)
, item_scores_per_quarters AS (
  SELECT
    item
    , sales_2016_q1
    , 1.0
      * (sales_2016_q1 - MIN(sales_2016_q1) OVER())
      -- PostgreSQL, Redshift, BigQuery, SparkSQL, NULLIF로 divide 0 회피
      / NULLIF(MAX(sales_2016_q1) OVER() - MIN(sale_2016_q1) OVER(), 0)
      -- Hive, CASE 식 사용
      / (CASE
          WHEN MAX(sales_2016_q1) OVER() - MIN(sales_2016_q1) OVER() = 0 THEN NULL
          ELSE MAX(sales_2016_q1) OVER() - MIN(sales_2016_q1) OVER() 
        END)
      AS score_2016_q1
    , sales_2016_q4
    , 1.0
      * (sales_2016_q4 - MIN(sales_2016_q4) OVER())
      -- PostgreSQL, Redshift, BigQuery, SparkSQL, NULLIF로 divide 0 회피
      / NULLIF(MAX(sales_2016_q4) OVER() - MIN(sales_2016_q4) OVER(), 0)
      -- Hive, CASE 식 사용
      / (CASE
          WHEN MAX(sales_2016_q4) OVER() - MIN(sales_2016_q4) OVER() = THEN NULL
          ELSE MAX(sales_2016_q4) OVER() - MIN(sales_2016_q4) OVER()
        END)
      AS score_2016_q4
  FROM item_sales_per_quarters
)
SELECT *
FROM item_scores_per_quarters
;
```

- 분기별 상품 점수 가중 평균으로 순위를 생성하는 쿼리 (1분기 7:3 4분기)

```sql
WITH
item_sales_per_quarters AS (
  -- CODE.24.12
),
item_scores_per_quarters AS (
  -- CODE.24.13
)
SELECT
  item
  , 0.7 * score_2016_q1 + 0.3 * score_2016_q4 AS score
  , ROW_NUMBER()
      OVER(ORDER BY 0.7 * score_2016_q1 + 0.3 * score_2016_q4 DESC)
    AS rank
FROM item_scores_per_quarters
ORDER BY rank
;
```

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

- 
```sql
```
```sql
```
```sql
```
```sql
``` 
```sql
```
```sql
```
```sql
```

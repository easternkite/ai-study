
![](https://firebasestorage.googleapis.com/v0/b/brocallie.appspot.com/o/pasterly%2Fimage_1744038289463_bnxkwe.png?alt=media&token=21bca231-0007-4d8f-abba-4af4f78a5247)

Search
retrive
generate

![](https://firebasestorage.googleapis.com/v0/b/brocallie.appspot.com/o/pasterly%2Fimage_1744038390972_v39uwv.png?alt=media&token=d7fa075b-e4fd-4b32-be3d-48f9de1a2c37)
임베딩 생성, 백터 서치, 향상된 리스폰스 얻기


![](https://firebasestorage.googleapis.com/v0/b/brocallie.appspot.com/o/pasterly%2Fimage_1744038667734_87r0ug.png?alt=media&token=23e9f39f-638b-4487-b2b6-809f81078f69)

텍스트 임베딩은 의미있는 백터로 변환하는 작업이라고 보면 된다.



## Vector Search 
>벡터들 간의 **밀접도(유사도 또는 거리)**를 측정하여, **가장 근접한 데이터**를 추출하는 것이 핵심입니다.

백터 스페이스 측정 매트릭 
1. 맨허튼 디스턴스 
2. 유클리드 디스턴스
3. 코사인
4. Dot product 


### top K
**Top K**는 **벡터 검색**에서 **가장 유사한 K개의 데이터**를 의미합니다

![](https://firebasestorage.googleapis.com/v0/b/brocallie.appspot.com/o/pasterly%2Fimage_1744039354810_gki75p.png?alt=media&token=47a2b693-432d-40c3-9376-6d82fb7f1f40)
### BigQuery와 LLM을 결합한 RAG 과정

RAG가 어떻게 활용될 수 있는지에 대해 좀 더 구체적으로 설명하겠습니다:

1. **쿼리 작성 (Retrieve)**
    
    - 사용자가 질문을 입력하면, 이 질문은 **LLM**에 의해 **벡터로 변환**됩니다.
        
    - 이 벡터는 **BigQuery**와 같은 데이터베이스에서 검색 쿼리를 생성하거나 실행하는 데 사용될 수 있습니다. 즉, **BigQuery**는 해당 벡터와 가장 관련성 높은 **데이터**를 검색하고 반환합니다.
        
2. **데이터 검색**
    
    - BigQuery는 대규모 데이터셋에서 빠르게 유의미한 **데이터를 검색**합니다. BigQuery의 SQL 쿼리 엔진은 대규모 데이터를 실시간으로 처리할 수 있기 때문에 **RAG의 검색 단계**를 처리하는 데 매우 적합합니다.
        
3. **LLM과 통합 (Generate)**
    
    - BigQuery에서 반환된 **검색된 데이터**는 **LLM**으로 전달됩니다. LLM은 이 데이터를 바탕으로 사용자가 원하는 **정확한 답변**을 생성합니다. 예를 들어, 특정 데이터베이스에서 가져온 정보를 기반으로 **설명**이나 **분석**을 할 수 있습니다.




## Console 사용 방법
1. Connection 생성

커넥션을 생성한다.
```sql
CREATE OR REPLACE MODEL `CustomerReview.Embeddings`
REMOTE WITH CONNECTION `us.embedding_conn`
OPTIONS (ENDPOINT = 'text-embedding-005');
```

CSV를 업로드한다.
```sql
LOAD DATA OVERWRITE CustomerReview.customer_reviews
(
    customer_review_id INT64,
    customer_id INT64,
    location_id INT64,
    review_datetime DATETIME,
    review_text STRING,
    social_media_source STRING,
    social_media_handle STRING
)
FROM FILES (
    format = 'CSV',
    uris = ['gs://spls/gsp1249/customer_reviews.csv']
);
```

`ML.GENERATE_EMBEDDING` 을 사용하여 테이블에 대해서 텍스트임베딩을 생성한다.
```SQL
CREATE OR REPLACE TABLE `CustomerReview.customer_reviews_embedded` AS
SELECT *
FROM ML.GENERATE_EMBEDDING(
    MODEL `CustomerReview.Embeddings`,
    (SELECT review_text AS content FROM `CustomerReview.customer_reviews`)
);
```

Vector Space 서치 !
```sQL
CREATE OR REPLACE VECTOR INDEX `CustomerReview.reviews_index`
ON `CustomerReview.customer_reviews_embedded`(ml_generate_embedding_result)
OPTIONS (distance_type = 'COSINE', index_type = 'IVF');
```
코사인 매트릭스를 이용하였다.


```SQL
CREATE OR REPLACE TABLE `CustomerReview.vector_search_result` AS

SELECT

query.query,

base.content

FROM

VECTOR_SEARCH(

TABLE `CustomerReview.customer_reviews_embedded`,

'ml_generate_embedding_result',

(

SELECT

ml_generate_embedding_result,

content AS query

FROM

ML.GENERATE_EMBEDDING(

MODEL `CustomerReview.Embeddings`,

(SELECT 'service' AS content)

)

),

top_k => 5,

options => '{"fraction_lists_to_search": 0.01}'

);
```

customer_reviews_embedded 테이블에서 생성한 임베딩을 기반으로 벡터서치를 사용하는 것을 볼 수 있다.
`top_k`는 **유사도를 기준으로 상위 k개의 결과**를 선택하는 매개변수

![](https://firebasestorage.googleapis.com/v0/b/brocallie.appspot.com/o/pasterly%2Fimage_1744040414434_lw13qx.png?alt=media&token=bb9d4558-97d7-49a8-a52c-56a66204ca54)

백터서치 테이블이 생성되었다. 

![](https://firebasestorage.googleapis.com/v0/b/brocallie.appspot.com/o/pasterly%2Fimage_1744040626977_07o0xc.png?alt=media&token=243b3524-f076-4f36-98c0-e632ee8c32f0)

## 제미나이 세팅
```SQL
CREATE OR REPLACE MODEL `CustomerReview.Gemini`

REMOTE WITH CONNECTION `us.embedding_conn`

OPTIONS (ENDPOINT = 'gemini-2.0-flash');
```

더 진보된 제비나이 응답을 뽑기위해, 이전에 백터 서치를 통해 생성한 테이블을 + LLM 자연어 처리능력을 이용하여 새로운 데이터를 생성한다.

```SQL
SELECT
    ml_generate_text_llm_result AS generated
FROM
    ML.GENERATE_TEXT(
        MODEL `CustomerReview.Gemini`,
        (
            SELECT
                CONCAT(
                    'Summarize what customers think about our services',
                    STRING_AGG(FORMAT('review text: %s', base.content), ',\n')
                ) AS prompt
            FROM
                `CustomerReview.vector_search_result` AS base
        ),
        STRUCT(
            0.4 AS temperature,
            300 AS max_output_tokens,
            0.5 AS top_p,
            5 AS top_k,
            TRUE AS flatten_json_output
        )
    );
```

백터 서치로 뽑은 데이블을 기반으로 다음과 같이 요약해주었다.
```
Overall, customer reviews are mixed. Many customers praise the **great service, friendly and helpful staff, and amazing coffee**. They also appreciate **short wait times**. However, some customers report **bad service, rude and unhelpful staff, and excessively long wait times**.
```

How can the code be improved? For example, instead of saving vector search results to a table (Task 3), could that process be embedded directly into answer generation (Task 4) for real-time retrieval?

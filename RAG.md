
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

## 어떻게 foundation Model의 정확성을 높여볼 수 있을까?
foundation Model은 표준화된 목적으로 훈련된 모델이다. 

### Model 튜닝
> 특정 Task에서 성능을 높일 수 있다.


## Grounding
> **"그라운딩(Grounding)"**은 AI가 **현실 세계의 정보나 신뢰할 수 있는 데이터에 기반해서 응답을 생성하는 것**

실제 문서를 기반으로 프롬프팅을 구성
* 특정 문서에 대한 인용구를 추가한다.
* 더 나은 검색 성능을 위해 search entry point를 활용한다. -> 질문에 관련된 부분만 뽑아내기


# RAG 다시 정리하기
> RAG는 model이 외부 신뢰된 문서를 기반으로 작성된 정보에 접근을 제공하는 기술이다. Grounding 방식을 사용한 대표적인 기술.


![](https://firebasestorage.googleapis.com/v0/b/brocallie.appspot.com/o/pasterly%2Fimage_1744646285262_jpvpgo.png?alt=media&token=f73b1b52-fa81-4688-959d-8d5ade61c8ac)


TextEmbedding을 사용하여 Vector Search로 정확성을 높힐 수 있다.

사줘 Ai에, 네이버 API 리스폰스들에 대해서 텍스트 embedding을 적용하고, 
백터서치로 더 정확한 정보를 서치해볼 수 있겠음


![](https://firebasestorage.googleapis.com/v0/b/brocallie.appspot.com/o/pasterly%2Fimage_1744646604403_zap3r4.png?alt=media&token=001f5e5f-4558-425b-b833-ec44298596ca)

The architecture contains the following components:

| Component                                   | **Purpose**                                                                                                                                                                                        | Interactions                                                                                                                                     |
| ------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Data ingestion subsystem**                | Prepare and process external data that's used to enable RAG capability.                                                                                                                            | The data ingestion subsystem only interacts with other subsystems through the database layer.                                                    |
| **Serving subsystem**                       | Handle the request-response flow between the generative AI application and its users.                                                                                                              | The serving subsystem accesses ingested data through the database layer.                                                                         |
| **Quality evaluation subsystem (optional)** | Evaluate the quality of responses generated by the serving subsystem.                                                                                                                              | The quality evaluation subsystem interacts with the serving subsystem directly and with the data ingestion subsystem through the database layer. |
| **Databases**                               | Store the following data:  <br>• Prompts  <br>• Vectorized embeddings of the data used for RAG  <br>• Configuration of the serverless jobs in the data ingestion and quality evaluation subsystems | All of the subsystems in the architecture interact with the databases.                                                                           |
이런 무시무시한 그림도 있다
![](https://firebasestorage.googleapis.com/v0/b/brocallie.appspot.com/o/pasterly%2Fimage_1744646782694_903rea.png?alt=media&token=dbc42814-7965-4184-9a3e-0dc48a706438)

## **Data ingestion subsystem**
> **Data Ingestion Subsystem**은 외부 소스(파일, 데이터베이스, 스트리밍 서비스 등)에서 데이터를 수집하며, 이 데이터에는 품질 평가를 위한 프롬프트가 포함된다. 이 서브시스템은 **RAG 기능**을 제공하는 역할을 한다.

![](https://firebasestorage.googleapis.com/v0/b/brocallie.appspot.com/o/pasterly%2Fimage_1744647616826_6g0hv9.png?alt=media&token=3fd86d76-e296-46e8-87f9-8625196a8212)

1. Data Upload : 클라우드 버킷에 업로드
2. Upload Notification : 데이터 업로드되면 알림
3. Cloud Run job is triggered : 데이터 업로드를 위한 job 실행
4. Retrieve job configuration data : 설정 정보를 기반으로 cloud job 이 실행된다. Cloud Run starts the job by using configuration data that's stored in an **AlloyDB** for **PostgreSQL** database.
5. Parse and chunk data : For example, the preparation can include parsing the data, converting the data to the required format, and dividing the data into chunks.
6. Create Vector embeddings
7. Store embeddings


## **Serving subsystem**
> **Serving Subsystem**은 생성적 AI 애플리케이션과 사용자 간의 요청-응답 흐름을 처리한다.


![](https://firebasestorage.googleapis.com/v0/b/brocallie.appspot.com/o/pasterly%2Fimage_1744647585521_hn254r.png?alt=media&token=29e34cf7-a4ac-4a25-9b19-43ae15c68f9c)
* Request sent to the app
* Create Embedding : 자연어 요청을 embedding 한다.
* RAG retrieval
* Send augmented(강화된) prompt
* Response from LLM
* Store logs
* Save for offline analytics
* Responsible AI checks
* Return Response


## **Quality evaluation subsystem**
> **Optional Quality Evaluation Subsystem**은 생성된 응답의 품질을 자동으로 평가하는 방법을 제공한다

![](https://firebasestorage.googleapis.com/v0/b/brocallie.appspot.com/o/pasterly%2Fimage_1744647483571_tetm9x.png?alt=media&token=df65bdaf-9041-4d9e-ba0f-bf0bcb462d7e)

1. Trigger an evaluation
2. Start evaluator job
3. Retrieve job config
4. pull evaluation prompts
5. Assess prompt response quality
6. Store prompt scores

아래 링크에서 더 학습해보는거 추천
https://www.cloudskillsboost.google/paths/1282/course_templates/1120/documents/507356

### What is the role of vector embeddings in a RAG system?
> To represent text and other data types in a way that captures semantic meaning.

### What is the purpose of using semantic search in the RAG serving subsystem?
> To retrieve documents based on the meaning and intent of the query.


### Which of the following types of data should be stored in a location other than the Vertex AI Search data store?
> Training Data

**Training data**는 머신러닝 모델을 훈련시키기 위한 **대규모 데이터**로, 일반적으로 **Google Cloud Storage**나 **BigQuery** 같은 별도의 대용량 데이터 저장소에 저장됩니다.

### What is the purpose of the pgvector extension in AlloyDB for PostgreSQL?
> To enable the storage and querying of vector embeddings.

**pgvector** 확장은 **AlloyDB for PostgreSQL**에서 **벡터 임베딩**을 저장하고 쿼리하는 기능을 제공합니다. 벡터 임베딩은 유사도 검색과 머신러닝 작업에서 중요한 역할을 하며, 이 확장을 통해 벡터 데이터를 저장하고 효율적으로 **가장 유사한 벡터 검색**을 할 수 있습니다.


### Which of the following is a technique used to improve the performance of a foundation model for specific tasks?
> Supervised tuning

**Supervised tuning**은 특정 작업(task)에 맞게 **기초 모델(Foundation Model)**의 성능을 향상시키기 위한 대표적인 기법입니다.

- 사람이 라벨링한 데이터(정답이 있는 데이터)를 사용해서 모델을 다시 학습시키는 방식입니다.
    
- 예: 챗봇이 고객 응대에 더 잘 대답하도록 하기 위해, 실제 대화 예시와 정답을 학습시키는 것.

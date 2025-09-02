# 빅쿼리에서 멀티모달 임베딩모델 및 벡터 검색을 통한 얼굴 유사도 검색 

해당 데모의 목적은 빅쿼리를 통하여 오브젝트 스토리지에 저장된 얼굴 이미지에 대하여 **Multi Modal Embedding Model** 을 통하여 벡터화하여, 빅쿼리에서 제공하는 **VECTOR_SEARCH** 펑션을 사용하여 찾으려는 소스 얼굴 이미지와 검색 대상 얼굴 이미지에 대한 유사도분석(정확히는 cosine distance 비교)을 통하여 가장 유사한 인물 이미지를 검색하여 주는 시나리오입니다.     

**VECTOR_SEARCH** 펑션에 대하여 해당 데모에서는 얼굴에 대한 유사도 검색에 사용하였으나, 온라인 마켓에서의 상품검색에도 적용이 가능합니다. 해당 데모는 아래와 같이 5단계로 이루어집니다. 

1. 얼굴 이미지 저장용 **오브젝트 스토리지 버킷 생성** 및 **샘플 이미지 복사** 
2. 빅쿼리 **데이터셋 생성** 및 **임베딩 모델 커넥션 생성**
3. **원격 모델(Remote ML Model) 생성** 및 비정형 데이터(얼굴 이미지)를 위한 **외부 테이블 생성**
4. **임베딩 모델**을 통한 얼굴 이미지에 대한 벡터화
5. **VECTOR_SEARCH** 펑션을 통하여 벡터값 검색(코사인 거리 비교)을 통하여 유사한 얼굴 이미지 찾기

## 아키텍처 다이어그램
       
  <p align="center"><img width="900" alt="Screenshot 2024-07-28 at 4 21 13 PM" src="https://github.com/user-attachments/assets/5a43cb8d-db5e-4277-8ad3-af0fd3e7c21b">    
      

## 1단계: 얼굴 이미지 저장용 버킷 생성 및 샘플 이미지 복사

1. 얼굴 이미지 저장용 오브젝트 스토리지 버킷 생성
    ```sh
    ## Cloud Shell 에서 실행을 추천 드립니다. 
    gsutil mb -l asia-northeast3 gs://[your-bucket-name]
    ```   

2.  첨부된 샘플 얼굴 이미지에 대하여 버킷에 복사
    ```sh
    unzip sample_faces.zip
    gsutil cp -r sample_faces gs://[your-bucket-name]
    ```   
      <img width="950" alt="Screenshot 2024-07-28 at 4 21 13 PM" src="https://github.com/user-attachments/assets/1cc9340a-0d3f-46ee-bb43-443462fbef8d">    
   

## 2단계: 빅쿼리 데이터셋 생성 및 임베딩 모델 커넥션 생성

1. 서울리전에 데이터셋 생성 
    ```sh
    bq mk --location=asia-northeast3 [your-dataset-name]]
    ```   

2. 임베딩 모델 커넥션 생성   
    (1) 빅쿼리 스튜디오에서 **+Add data**, **verte** 검색, **Vertex AI** 선택, **BigQuery Federation** 선택
    <img width="950" alt="Screenshot 2024-07-28 at 4 35 07 PM" src="https://github.com/user-attachments/assets/ed5863ca-6854-4988-97f7-cdb9fbd8edbf">

    <img width="950" alt="Screenshot 2024-07-28 at 4 35 07 PM" src="https://github.com/user-attachments/assets/06a1325b-6b37-48bc-9356-800ec61d285c">

    (2) **Connection ID**: [your-coonnection-id], **Location Type**: Region asia-northeast3(Seoul) 선택후 **Create connection** 
    
    <img width="950" alt="Screenshot 2024-07-28 at 4 35 07 PM" src="https://github.com/user-attachments/assets/62186f1b-4f18-4439-9cd5-c47c4824b10c">

    (3) Service Account ID 확인 및 권한 부여: 빅쿼리 스튜디오에서 **Externnal Connections** 에서 확인, `bqcx로 시작되는 id` 에 대하여 클립보드에 복사
    
    <img width="950" alt="Screenshot 2024-07-28 at 4 35 07 PM" src="https://github.com/user-attachments/assets/aaefd544-c3df-48b8-9931-670b90f3392a">

    (4) 콘솔 **IAM & Admin** 메뉴에 **IAM** 화면, **+Grant access** 선택, **New Principals** 에 `bqcx 로 시작되는 id` 입력 
    
    (5) 생성한 컨넥션이 임베딩 모델을 사용할수 있는 권한과, 얼굴 이미지가 저장된 오브젝트 스토리지 버킷을 읽을수 있는 권한 할당: **+Add roles** 선택 **Vertex AI User**, **Storage Object User** 추가 
    
    <img width="350" alt="Screenshot 2024-07-28 at 4 35 07 PM" src="https://github.com/user-attachments/assets/69347bfe-5c6a-48de-9669-d06153c0a6af">

## 3단계: 원격 모델(Remote ML Model) 생성 및 얼굴 이미지를 위한 외부 테이블 생성

1. 빅쿼리 스튜디오에서 아래 쿼리 입력: 최초 생성한 데이터셋에 생성한 컨넥션을 사용하여 **face_search_model** 생성  
      ```
      CREATE OR REPLACE MODEL `[your-dataset-name].face-search_model`
      REMOTE WITH CONNECTION `[your-project-id]].asia-northeast3.[your-connection-id]`
      OPTIONS (ENDPOINT = 'multimodalembedding@001');
      ```   
   
2. 버킷에 저장된 얼굴 이미지를 위한 **외부 테이블(External Table)** 생성
    ```  
    CREATE OR REPLACE EXTERNAL TABLE `[your-dataset-name].external_faces_table`
    WITH CONNECTION `[your-project-id].asia-northeast3.[your-connection-id]`
    OPTIONS(
    object_metadata = 'SIMPLE',
    uris = ['gs://[your-bucket-name]/sample_faces/women/*'],
    max_staleness = INTERVAL 1 DAY,
    metadata_cache_mode = 'AUTOMATIC');
      ```

    <img width="900" alt="Screenshot 2024-07-28 at 4 35 07 PM" src="https://github.com/user-attachments/assets/a1b6a3a0-6126-484e-8e4b-7e73e68cec76">

3. 버킷에 저장된 이미지에 대하여 **임베딩화** 및 테이블에 저장 및 **벡터값** 확인 
    ```  
    ##버킷에 저장된 이미지에 대한 임베딩
    CREATE OR REPLACE TABLE `[your-dataset-name].face_embeddings` AS
    SELECT *
    FROM ML.GENERATE_EMBEDDING(
    MODEL `[your-dataset-name].face-search_model`,
    TABLE `[your-dataset-name].external_faces_table`,
    STRUCT(TRUE AS flatten_json_output,
    512 AS output_dimensionality)
    );
      ```

    ```  
    ##face-embeddings 테이블에 저장된 벡터값 확인
    SELECT * FROM '[your-dataset-name].face-embeddings';
      ```

    <img width="900" alt="Screenshot 2024-07-28 at 4 35 07 PM" src="https://github.com/user-attachments/assets/4d92039d-205d-42c2-932f-f05ec8e1551d">

## 4단계: 벡터값 검색을 통하여 유사한 얼굴 이미지 찾기 

1. 유사도 검색을 희망하는 소스이미지 에 대한 외부 테이블  **external_face_test_table** 생성 및 임베딩 적용 테이블 생성 **test_face_embeddings** 
      ```
      ## 찾으려는 얼굴 소스 이미지(박소담) 저장을 위한 외부 테이블 생성
      CREATE OR REPLACE EXTERNAL TABLE `[your-dataset-name].external_face_test_table`
      WITH CONNECTION `[your-project-id].asia-northeast3.[your-model-id]`
      OPTIONS(
      object_metadata = 'SIMPLE',
      uris = ['gs://[your-bucket-name]/sample_faces/sodam/*'],
      max_staleness = INTERVAL 1 DAY,
      metadata_cache_mode = 'AUTOMATIC');
      
      ## 찾으려는 얼굴 소스 이미지에 대한 임베딩화 및 벡터값 저장
      CREATE OR REPLACE TABLE `[your-dataset-name].test_face_embeddings` AS
      SELECT *
      FROM ML.GENERATE_EMBEDDING(
      MODEL `[your-dataset-name].face-search_model`,
      TABLE `[your-dataset-name].external_face_test_table`,
      STRUCT(TRUE AS flatten_json_output,
      512 AS output_dimensionality)
      );
      ```   

2. **VECTOR_SEARCH** 펑션을 통하여 소스 이미지의 벡터값 **test_face_embeddings**과 유사도 검색 대상 얼굴 이미지 벡터값 테이블에 벡터값 **face_embeddings** 검색(vector distance 비교)

    ```
    ## 소스 이미지 벡터값과 대상 이미지 벡터값이 저장된 테이블에서 유사도 검색(vector distance)
    SELECT base.uri AS image_link, distance
    FROM
    VECTOR_SEARCH(
    TABLE  `face_search.face_embeddings`,
    'ml_generate_embedding_result',
    (
    SELECT * FROM `face_search.test_face_embeddings`
    ),
    ## tok_k 값을 통하여 상위 몇개의 값을 결과로 출력할지 정의
    top_k => 5,
    distance_type => 'COSINE',
    options => '{"use_brute_force":true}'
    );  
    ```   


3. 박소담(sodam_test.png)이미지와 벡터값 유사도가 가장 높은 순서도 이미지 검색(vector distance 가 가장 작은 순서로 유사도가 높음)

<img width="900" alt="Screenshot 2024-07-28 at 4 35 07 PM" src="https://github.com/user-attachments/assets/c0bc0142-4f01-4065-ae49-6ca6ca451ee4">


## 참고자료

- [Introduction to vector search](https://cloud.google.com/bigquery/docs/vector-search-intro)
- [Get multimodal embeddings](https://cloud.google.com/vertex-ai/generative-ai/docs/embeddings/get-multimodal-embeddings)
- [The ML.GENERATE_EMBEDDING function](https://cloud.google.com/bigquery/docs/reference/standard-sql/bigqueryml-syntax-generate-embedding)
- [Search embeddings with vector search](https://cloud.google.com/bigquery/docs/vector-search)
- [Generate and search multimodal embeddings](https://cloud.google.com/bigquery/docs/generate-multimodal-embeddings)
- [Introduction to object tables](https://cloud.google.com/bigquery/docs/object-table-introduction)
- [Introduction to external tables](https://cloud.google.com/bigquery/docs/external-tables)

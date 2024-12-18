# 겨울방학 Briefly 프로젝트

## 프로젝트 주제
- 3줄 요약 뉴스 서비스 백엔드 구현
---

## 프로젝트 목적
- 다양한 카테고리(키워드)의 뉴스 데이터를 수집하고 이를 처리하는 대규모 데이터 처리 경험
- Java Spring boot 프레임워크 학습 및 활용 역량 향상
- AI 적용 및 활용 능력
- 한국에서 취업 경쟁력을 증진하기 위함
---

## 사용 기술 스택
가장 중요한것은, 역량 향상이나, 비용 문제를 해결해야함

-   **Spring Boot**: REST API 제공
-   **FastAPI**: AI 모델 서빙 (ONNX Runtime, 로컬 GPU)
-   **PostgreSQL (RDS)**: 요약된 데이터 저장,  서비스 메인 데이터베이스
-   **Redis (ElastiCache)**: 요약된 데이터 캐싱 - 여러 사용자가 동일한 키워드를 사용하여 동일한 뉴스 데이터를 요청하는 경우 효율적 -> 추후 사용자 늘어나는 경우 도입
-   **AWS Lambda**: 데이터 수집 및 중복 검출
-   **AWS DynamoDB**: 중복 검출용 **해시값**만 저장 (키-값 쌍).
-   **AWS SQS**: 비동기 데이터 전달
-   **AWS EventBridge**: Lambda 트리거
-   **AWS CloudWatch**: 모니터링 및 알림
---

## 작업 순서

### 0. 데이터베이스 설계 및 AWS 아키텍쳐 설계 및 구성 (12/16 ~ 12/22)

> 반드시 프리티어 범위에서 사용
- AWS EC2 t2.micro - Springboot 웹 어플리케이션 올릴 때 사용
- AWS RDS PostgreSQL db.t3.micro - 메인 데이터베이스
- AWS ElasticCache Redis cache.t2.micro - 자주 사용되는 데이터 캐싱 용도
- AWS S3 : 모델 파일 및 뉴스 데이터 백업용
- AWS CloudWatch : 프리티어 한도를 초과해서는 안되므로 모니터링 및 알람 설정
- AWS Lambda : 데이터 수집 서버리스 서비스
- AWS EventBridge : 이벤트 트리거 -> 매일 자정마다 데이터 수집 Lamda 호출 트리거
- AWS SQS - 작업 큐 용도
- AWS Route53 : 웹 서비스 도메인 연결 (추후 프론트 구현 시 설정할 예정)
- AWS CloudFront : 웹 서비스 도입 시 정적 파일 배포를 빠르게 수행하기 위해 추가 예정
- AWS ACM : HTTPS 인증서 (추후 프론트 구현 시 설정할 예정)

### 1. Spring boot 개발 환경 구성

사용할 의존성
- Spring Web : API 어플리케이션 역할을 수행하므로
- Spring Data JPA : ORM 용도
- Spring Boot Actuator
- Spring Security : 인증기능 구현을 위해 사용 - 나중에 웹 서비스 구현 시 도
- Lombok : 편의 도구
- Redis : Redis 사용을 위해 사용

### 2. 데이터 수집 파이프라인 구현
1. 데이터 수집 키워드 선정 : IT, 세계, 시사, 경제, 정치, 스포츠, 연예, ...
2. AWS Eventbridge
	- 매일 자정마다 (개발 시 10분마다) Lambda 함수 트리거
3. AWS Lambda
	- 네이버 뉴스 API, 뉴스 오피스 API를 호출해서 데이터 수집
	- 수집한 뉴스 데이터에 중복을 검출하기 위해 AWS DynamoDB 활용
	- 뉴스 제목과 URL을 해싱하여 DynamoDB에 저장
	- 해시값이 중복된다면 수집한 데이터는 제외.
	- 중복이 아닌 데이터는 SQS로 전송
	- Cold Start가 발생하지 않는지 확인하고 메모리 최적화를 수행
		- Cold Start란? AWS Lambda 함수가 처음 실행될 때 또는 오랫동안 사용되지 않을 때 새롭게 인스턴스를 생성하는 지연 시간
		- 근데 프리티어 사용해야 해서 어쩔 수 없음 -> 메모리만 512MB로 설
4. AWS SQS
	- 수집한 데이터 (중복 검사 통과한 데이터)는 Lambda에서 SQS로 전송됨
	- 비동기적으로 요약 처리를 위한 데이터 전달하는 역할 -> FastAPI에서 꺼내감
5. FastAPI
	- SQS에서 작업 읽어옴
	- 로컬 컴퓨터에서 AI 모델 돌려서 3줄 요약
	- 요약된 데이터를 AWS RDS PostgreSQL에 최종 저장

### 3. koBART 및 FastAPI 구현
> AI 모델 관련 어플리케이션은 비용 문제로 인해 로컬 컴퓨터를 사용한다.

- 데이터 요약을 해주기 위해 AI 구성 및 FastAPI 구현
- FastAPI가 한 번에 여러 뉴스 기사를 요약할 수 있도록 배치 API 구현
- SpringBoot에서도 FastAPI에 구현된 API를 호출하는 API 개발해야함
- ONNX Runtime : koBART 모델을 ONNX로 변환하면 추론 속도가 크게 개선된다
	- ONNX Runtime은 AI 모델의 추론 속도를 크게 개선하는 최적화된 엔진
	- 
- 게임 잘 안하니까 GPU 사용하도록 모델 설정 구성
- AI 모델 서버는 FastAPI와 분리하는게 좋음
	- AI 모델은 **GPU 메모리를 많이 소모**하므로 별도의 서버로 분리해 관리하는 것이 효율적

### 4. 데이터 처리 파이프라인 검증
- 데이터 수집 -> 요약 -> 사용 과정이 올바르게 동작하는지 검증
- **테스트 전략**:
	1.  **단위 테스트**: Spring Boot와 FastAPI 각각에 대해 단위 테스트 작성. 
		- Junit5, Mockito, Pytest, unittest.mock
	2.  **통합 테스트**: 전체 데이터 흐름을 검증하는 통합 테스트 수행.
	    -   뉴스 수집 → SQS 전달 → FastAPI 모델 서빙 → 요약 데이터 저장.
	3.  **성능 테스트**: JMeter나 k6를 활용하여 API 부하 테스트 수행.
		-  Apache JMeter 사용하여 부하 테스트
		- FastAPI 배치 API에 동시 요청 1000개 보내고 응답 시간 측정 수행.

### 5. Spring Boot REST API 구현
- 웹 서비스 제공을 위한 API 구현
- 문서는 직접 Notion에 작성 -> Naver, 쿠팡, 넥슨 OpenAPI 구조를 참고하며 작성
- 각 API 구현 이전 테스트 코드 먼저 작성하고 이를 만족시키기 위한 API 구현 수행 -> TDD 개발 방법론 경험
---

# ERD

---

# Cloud Architecture
![image](https://github.com/user-attachments/assets/a7dd6db7-ab11-4fdc-b912-b5d6b5a54135)


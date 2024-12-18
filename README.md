# 겨울방학 AutoReview 프로젝트

## 프로젝트 주제
- Open AI를 사용하여 Pull request 이벤트 발생 시,
  GPT가 자동으로 변경된 파일에 대해 코드 리뷰를 수행하고,
  코멘트를 달아주는 서비스.
  코드 리뷰 시 이전 커밋 내역을 참고한다.
---

## 프로젝트 목적
- 코드 리뷰 자동화: 코드 리뷰의 번거로움을 줄이고, 개발 프로세스의 효율성을 향상.
- Java Spring Boot 프레임워크 학습 및 활용 역량 향상: 백엔드 개발 역량 강화.
- AI 적용 및 활용 능력 배양: AI 모델을 실제 서비스에 통합하는 경험.
---

## 사용 기술 스택
- Spring Boot: REST API 제공 및 백엔드 서비스 구현.
- PostgreSQL (RDS): 코드 리뷰 기록 저장 및 서비스 메인 데이터베이스.
- AWS Lambda: GitHub 웹훅 이벤트 처리 및 서버리스 데이터 처리, OpenAI와 통신
- AWS SQS: 비동기 데이터 전달.
- AWS EventBridge: Lambda 트리거.
- AWS CloudWatch: 모니터링 및 알림.
- GitHub Webhooks: Pull Request 이벤트 감지.
- OpenAI API: 코드 리뷰 생성.
- Docker: 프로젝트 컨테이너화 및 배포.
- Jira 또는 Slack API (선택사항): 리뷰 결과 알림.
---

## 작업 순서

### 0. 데이터베이스 설계 및 AWS 아키텍쳐 설계 및 구성 (12/16 ~ 12/22)

> 반드시 프리티어 범위에서 사용
- AWS EC2 t2.micro - Springboot 웹 어플리케이션 올릴 때 사용
- AWS RDS PostgreSQL db.t3.micro - 메인 데이터베이스
- AWS CloudWatch : 프리티어 한도를 초과해서는 안되므로 모니터링 및 알람 설정
- AWS Lambda : 데이터 수집 서버리스 서비스
- AWS EventBridge : Lamda 호출 트리거
- AWS SQS - 작업 큐 용도

### 1. Spring boot 개발 환경 구성

사용할 의존성
- Spring Web: API 어플리케이션 역할.
- Spring Data JPA: ORM 용도.
- Spring Boot Actuator: 모니터링.
- Spring Security: 인증 및 권한 관리.
- Lombok: 코드 간결화.
- GitHub API: 웹훅 이벤트 처리 및 Pull Request 정보 수집.

### 2. GitHub Webhook 및 Lambda 함수 설정 (MM/DD ~ MM/DD)
1. GitHub Webhook 설정:
   Pull Request 이벤트 발생 시 AWS Lambda로 트리거.

2. AWS Lambda 함수 개발:
   GitHub Webhook에서 이벤트 데이터 수신.
   필요한 데이터 (변경된 파일, 커밋 내역 등) 추출.
   데이터 추출 후 AWS SQS로 전송.

### 3. SQS와 FastAPI 연동 (MM/DD ~ MM/DD)
1. AWS SQS 설정:
   Lambda에서 전송된 메시지를 FastAPI가 비동기적으로 처리.

2. FastAPI 개발:
   SQS에서 메시지 수신.
   변경된 코드 파일과 커밋 내역을 기반으로 OpenAI API 호출하여 코드 리뷰 생성.
   생성된 코드 리뷰를 PostgreSQL에 저장.
   코드 리뷰 생성 결과를 Slack 또는 이메일로 알림.

### 4. OpenAI API 통합 및 코드 리뷰 생성 (MM/DD ~ MM/DD)
1. AWS SQS 설정
- SQS 큐 생성 (AutoReviewQueue)
- IAM 역할 생성 및 SQS 접근 권한 부여
- Lambda 함수에 IAM 역할 할당

2. Lambda 함수 구현
- Lambda 함수 생성 (Python)
- 메시지 수신 및 파싱 로직 구현
- OpenAI API 호출 및 코드 리뷰 생성
- PostgreSQL에 리뷰 저장 로직 구현
- GitHub PR에 코멘트 추가 로직 구현
- 에러 핸들링 및 DLQ 설정 (이건 나중에)

### 5. Spring Boot REST API 구현
- 웹 서비스 제공을 위한 API 구현
- 문서는 직접 Notion에 작성 -> Naver, 쿠팡, 넥슨 OpenAPI 구조를 참고하며 작성
- 각 API 구현 이전 테스트 코드 먼저 작성하고 이를 만족시키기 위한 API 구현 수행 -> TDD 개발 방법론 경험
---

### 6. 코드 리뷰 결과 알림 시스템 구현 (MM/DD ~ MM/DD)
Slack API 또는 Jira API를 활용하여 리뷰 결과 자동 알림.
---


# ERD

---

# Project Architecture

![image](https://github.com/user-attachments/assets/a7dd6db7-ab11-4fdc-b912-b5d6b5a54135)


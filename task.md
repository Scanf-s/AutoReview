# **기능 요구사항 및 작업 흐름**

## **1. 전체 흐름**

1. **사용자 접속 및 소셜 로그인**  
   - **Spring Boot** 웹 서비스에 접속.
   - GitHub **소셜 로그인**을 통해 사용자를 인증 및 식별.

2. **GitHub 정보 입력 및 저장**  
   - 사용자로부터 **자동 코드 리뷰**를 수행할 GitHub 리포지토리 정보(레포 이름, 소유자 등)를 입력받음.  
   - 입력받은 정보를 **PostgreSQL** 데이터베이스에 저장.

3. **GitHub PR 이벤트 감지**  
   - 사용자의 리포지토리에 PR(Pull Request)이 발생하면 GitHub **Webhook**이 트리거됨.  
   - Webhook 요청이 **AWS Lambda**를 호출.

4. **OpenAI API를 사용한 코드 리뷰 생성**  
   - **AWS Lambda**는 GitHub에서 변경된 코드 파일과 커밋 내역을 수집.  
   - **OpenAI API**를 호출하여 코드 리뷰를 생성.  

5. **PR에 코드 리뷰 추가**  
   - 생성된 리뷰 내용을 **GitHub API**를 사용하여 PR의 Comment로 자동 추가.  
   - 이 작업은 **Spring Boot**를 통해 처리.

6. **알림 기능**  
   - 코드 리뷰 생성 완료 후, 사용자가 설정한 채널로 알림 전송 (카카오톡, Slack, Discord, Email).

---

# **2. 기능별 구현 계획**

## **2.1. GitHub 소셜 로그인**

- **기능:**  
   사용자가 GitHub 계정으로 로그인하여 자신의 레포지토리를 등록할 수 있도록 구현.

- **사용 기술:**  
   - **Spring Security**: OAuth 2.0 기반 GitHub 소셜 로그인 구현.  
   - **GitHub OAuth App**: GitHub에서 OAuth App을 등록하고, Client ID와 Secret Key를 발급.  

- **작업 흐름:**  
   1. 사용자가 로그인 요청 → GitHub 로그인 페이지 리다이렉트.  
   2. GitHub 인증 후 Access Token 반환.  
   3. Access Token을 사용하여 사용자 정보를 가져옴 (이름, 이메일, GitHub 계정 ID).  
   4. 사용자 정보를 PostgreSQL에 저장 (회원 테이블에 저장).  

---

## **2.2. 사용자 GitHub 정보 입력 및 저장**

- **기능:**  
   사용자로부터 코드 리뷰를 수행할 GitHub 리포지토리 정보를 입력받고 저장.  

- **저장 정보:**  
   - GitHub 사용자 ID  
   - 리포지토리 이름  
   - GitHub Personal Access Token (PR 작성 권한이 필요)  

- **작업 흐름:**  
   1. 사용자 입력 화면 (프론트엔드)에서 리포지토리 정보를 입력받음.  
   2. 입력된 정보를 Spring Boot에서 PostgreSQL에 저장.  
   3. Access Token을 **AWS Secret Manager**에 저장하여 보안 강화.  

---

## **2.3. GitHub Webhook 설정**

- **기능:**  
   사용자의 리포지토리에서 PR이 생성될 때 Webhook 이벤트를 통해 자동으로 코드 리뷰를 실행.  

- **작업 흐름:**  
   1. Spring Boot에서 **GitHub API**를 호출해 사용자의 리포지토리에 Webhook을 자동 설정.  
   2. PR 이벤트 발생 시 Webhook이 **AWS Lambda**를 호출.

---

## **2.4. OpenAI API를 사용한 코드 리뷰 생성**

- **기능:**  
   GitHub Webhook 이벤트로 전달된 PR의 변경된 코드 내용을 OpenAI API에 보내 코드 리뷰를 생성.  

- **작업 흐름:**  
   1. **AWS Lambda**가 Webhook 이벤트 데이터를 수신.  
   2. PR의 변경된 파일과 코드 Diff 데이터를 추출.  
   3. OpenAI API로 코드 리뷰 요청:  
      ```plaintext
      You are a code reviewer. Please review the following code changes:  
      - Repository: {repository_name}  
      - Changed Files: {file_names}  
      - Diff: {code_diff}
      ```  
   4. 생성된 리뷰 결과를 AWS SQS 또는 Spring Boot에 전달.  

---

## **2.5. PR에 코드 리뷰 추가**

- **기능:**  
   생성된 코드 리뷰 결과를 GitHub PR의 Comment로 자동 추가.  

- **사용 기술:**  
   - **GitHub REST API**: Comment 작성 API 사용.  

- **작업 흐름:**  
   1. Spring Boot가 GitHub PR 정보를 기반으로 Comment를 추가하는 API 호출.  
      ```http
      POST /repos/{owner}/{repo}/issues/{issue_number}/comments
      ```
   2. OpenAI로부터 생성된 코드 리뷰를 Comment의 `body`로 전달.  

---

## **2.6. 알림 기능 구현**

- **기능:**  
   코드 리뷰가 생성되면 사용자에게 알림 전송.  

- **알림 채널:**  
   - **카카오톡**: KakaoTalk **Messaging API** 사용.  
   - **Slack**: Slack **Incoming Webhooks** 사용.  
   - **Discord**: Discord **Webhook API** 사용.  
   - **Email**: Spring Boot에서 JavaMailSender를 사용해 이메일 발송.  

- **작업 흐름:**  
   1. Spring Boot에서 PR Comment 추가 완료 후, 알림 전송 작업을 비동기적으로 처리.  
   2. 사용자의 알림 설정 정보에 따라 각 채널 API를 호출.  

---

# **3. 데이터베이스 설계**

### **주요 테이블**
1. **사용자 테이블** (`users`)  
   - `id` (PK)  
   - `github_id`  
   - `github_access_token`  
   - `email`  
   - `created_at`  

2. **리포지토리 테이블** (`repositories`)  
   - `id` (PK)  
   - `user_id` (FK → users)  
   - `repository_name`  
   - `webhook_url`  
   - `created_at`  

3. **코드 리뷰 테이블** (`code_reviews`)  
   - `id` (PK)  
   - `repository_id` (FK → repositories)  
   - `pull_request_number`  
   - `review_comment`  
   - `created_at`  

4. **알림 설정 테이블** (`notifications`)  
   - `id` (PK)  
   - `user_id` (FK → users)  
   - `slack_webhook_url`  
   - `kakao_api_key`  
   - `discord_webhook_url`  
   - `email`  

---

# **4. 시스템 아키텍처 요약**

1. **Spring Boot**  
   - 사용자 관리 (GitHub 로그인, 리포지토리 정보 입력 및 저장)  
   - GitHub API 호출 (Webhook 설정, PR에 Comment 추가)  
   - 알림 전송 (Slack, KakaoTalk, Discord, Email)  

2. **AWS Lambda**  
   - GitHub Webhook 이벤트 처리  
   - OpenAI API 호출 및 코드 리뷰 생성  

3. **PostgreSQL**  
   - 사용자 정보, 리포지토리 정보, 코드 리뷰 결과 저장  

4. **OpenAI API**  
   - 변경된 코드 리뷰 생성  

5. **알림 채널**  
   - Slack, KakaoTalk, Discord, Email  

---

# **5. 작업 순서**

1. **GitHub OAuth 로그인 및 사용자 정보 저장**  
2. **GitHub 리포지토리 등록 및 Webhook 설정**  
3. **PR 이벤트 감지 (Webhook → Lambda)**  
4. **OpenAI API를 통한 코드 리뷰 생성**  
5. **GitHub API로 PR Comment 추가**  
6. **알림 시스템 구현 및 테스트**  
7. **최종 테스트 및 배포 (Docker, CI/CD)**  

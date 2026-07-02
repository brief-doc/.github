# 📄 BriefDoc — 로컬 AI 기반 문서 요약·분류 RAG 서비스

## 1. 프로젝트 소개

본 프로젝트는 사용자가 업로드한 문서(PDF, DOCX, HWP 등)를 자동으로 OCR 처리하고, 로컬 LLM(Ollama)을 활용해 요약·분류 초안을 생성하는 **문서 처리 RAG 서비스**입니다.

사용자는 문서를 업로드하기만 하면 되고, 시스템이 텍스트 추출 → 벡터화 → 요약/분류 → 결재(승인) 초안 생성까지 백그라운드에서 자동으로 처리합니다.

단순히 파일을 저장하는 구조가 아니라,
**OCR 기반 원문 추출 + 벡터 검색(Chroma) + 로컬 LLM 요약/질의응답 + 결재(초안) 워크플로우**를 함께 사용하는 구조를 목표로 합니다.


## 시연영상
https://github.com/user-attachments/assets/dca96f44-58fe-4dda-b58a-30802719c052

---

## 2. 핵심 목표

* 다양한 포맷(PDF, DOCX, HWP)의 문서 업로드 및 OCR 텍스트 추출
* 쿠키, 세션 기반 사용자 인증 및 역할(Role) 관리
* 문서 자동 분류(카테고리) 및 LLM 기반 요약
* Chroma 기반 벡터 검색 및 RAG 질의응답
* 답변 출처(참조 문서/페이지) 표시
* SSE 기반 파이프라인 진행률 실시간 알림
* 초안(Draft) 작성 및 승인/반려 결재 워크플로우
* 전체 서비스를 Docker Compose로 로컬에서 완전히 구동 (외부 API 의존 없는 로컬 LLM)

---

## 3. Repository 구성

현재 프로젝트는 인프라(Docker Compose)를 기준으로 프론트엔드와 백엔드를 서브모듈로 분리하여 관리합니다.

| Repository | Description |
| --- | --- |
| [Docker](https://github.com/brief-doc/docker) | Docker Compose 기반 통합 실행 환경 (본 저장소) |
| [Backend](https://github.com/brief-doc/backend) | FastAPI 기반 API 서버, OCR/RAG 파이프라인 |
| [Frontend](https://github.com/brief-doc/frontend) | React + Vite 기반 사용자 화면 UI |

### Backend Repository

API 서버와 AI 처리 흐름을 담당합니다.

* 쿠키 인증 및 권한(Role) 처리
* 문서 업로드 API 및 파이프라인 Job 관리
* OCR을 통한 텍스트 추출 (PaddleOCR, docling, PyMuPDF 등)
* 텍스트 청킹 및 임베딩 생성
* Chroma 벡터DB 저장 및 검색
* Ollama 기반 LLM 요약/분류/RAG 답변 생성
* 초안(Draft) 작성 및 승인/반려 처리
* SSE 기반 실시간 알림

### Frontend Repository

사용자 화면을 담당합니다.

* 로그인 / 회원가입 화면
* 문서 업로드 및 처리 상태(진행률) 표시
* 문서 목록 조회 및 상세/요약 확인
* RAG 챗봇 질의응답 화면
* 초안 작성 및 결재(승인/반려) 화면

---

## 4. 전체 서비스 구조

```text
사용자
  ↓
프론트엔드 (React + Vite)
  ↓
백엔드 FastAPI 서버
  ↓
PostgreSQL / Redis / ChromaDB(로컬 볼륨) / Ollama
```

Docker Compose 기준 컨테이너 구조는 다음과 같습니다.

```text
frontend  ──▶  backend  ──▶  db (PostgreSQL)
                  │      ──▶  redis
                  │      ──▶  ollama  ──▶  ollama-init (최초 1회, 커스텀 모델 생성)
```

모든 서비스는 외부 클라우드 API 없이 로컬 환경(Docker)에서 완결됩니다.

---

## 5. 주요 기술 스택

### Frontend

* React, TypeScript, Vite
* Radix UI, MUI, Tailwind CSS
* React Router, React Hook Form, Axios

### Backend

* Python, FastAPI, SQLAlchemy
* cookie Authentication (python-jose, passlib)
* Docker

### Database / Storage

* PostgreSQL 16
* Redis 7 (세션/이벤트 캐시)
* ChromaDB (로컬 벡터 스토어)

### AI / RAG / OCR

* LangChain (langchain-chroma, langchain-ollama, langchain-huggingface)
* Sentence-Transformers / BGE-M3 임베딩
* Ollama (로컬 LLM 서빙, gemma2 기반 커스텀 모델)
* PaddleOCR, docling, PyMuPDF, pypdf (문서 파싱/OCR)

---

## 6. 문서 업로드·파이프라인 처리 흐름

사용자가 문서를 업로드하면 다음 흐름으로 처리됩니다. (`document_pipeline_router.py`, `document_pipeline_service.py`)

```text
1. 사용자 로그인 / 쿠키 인증
2. 파일 업로드 → 임시 저장(pdf_files/) 후 Job 레코드 생성
3. 202 Accepted 즉시 응답 (파이프라인은 백그라운드 asyncio Task로 실행)
4. [OCR] 원문 텍스트 추출
5. Document(doc) 테이블에 원문 저장
6. [임베딩] 텍스트 청킹 → 임베딩 생성 → Chroma 벡터DB 저장
7. [요약/분류] 키워드 기반 자동 분류 + LLM 요약 생성
8. Job 상태를 completed로 갱신
9. SSE(/notifications/subscribe)로 각 단계 진행률(pipeline_progress) 실시간 전송
10. 실패 시 트랜잭션 롤백 및 실패 알림 전송, 취소 요청 시 단계 사이에서 즉시 중단
```

---

## 7. RAG 질의응답 처리 흐름

사용자가 챗봇에 질문을 입력하면 다음 흐름으로 처리됩니다. (`rag_router.py`, `rag_service.py`)

```text
1. 사용자 질문 입력
2. 쿠키 인증
3. Retriever를 통해 Chroma 벡터DB에서 관련 청크 검색 (Top-K)
4. 검색된 문서(doc_name, category, page, snippet)를 컨텍스트로 조립
5. RAG_PROMPT 기반 프롬프트 구성 후 Ollama LLM 호출
6. 답변 생성 (스트리밍 응답 지원: /rag/query/stream)
7. 답변과 함께 참조 문서 출처 반환
8. rag_query / rag_query_ref 테이블에 질의·응답·출처 이력 저장
```

---

## 8. RAG 구조

본 프로젝트는 단일 로컬 ChromaDB 컬렉션을 벡터 스토어로 사용합니다. (`app/llm/vectorstore.py`)

* 임베딩 함수: BGE-M3 계열 HuggingFace 임베딩
* 유사도 척도: 코사인 거리(cosine)
* 각 청크는 다음과 같은 메타데이터와 함께 저장됩니다.

```json
{
  "doc_id": 123,
  "doc_name": "contract.pdf",
  "category": "법령·조례",
  "page_num": 3
}
```

문서가 새로 인제스트(ingest)되면 `reset_vectorstore()`로 싱글톤을 초기화하여 검색 결과에 즉시 반영합니다.

---

## 9. ERD

<img width="1262" height="577" alt="ERD_trimmed" src="https://github.com/user-attachments/assets/1a95a758-ac4f-4c3f-92ac-c47410a5c34e" />


---

## 10. 주요 상태값

파이프라인 Job의 상태는 다음과 같이 관리합니다. (`job.job_status`, `job.pipeline_stage`)

| job_status | 설명 |
| --- | --- |
| pending | Job 생성, 처리 대기 중 |
| running | 파이프라인 처리 중 |
| success / completed | 처리 완료 |
| failed | 처리 실패 |
| cancelled | 사용자 취소 요청으로 중단 |

| pipeline_stage | 설명 |
| --- | --- |
| uploaded | 업로드 완료, 처리 대기 |
| ocr | OCR 원문 추출 중 |
| embedding | 청킹 및 벡터DB 저장 중 |
| summarizing | LLM 요약·분류 생성 중 |
| completed | 전체 파이프라인 완료 |

프론트엔드는 SSE로 수신한 이 상태값을 기반으로 사용자에게 처리 진행률을 표시합니다.

---

## 11. API 문서

주요 API는 다음과 같습니다. (FastAPI Swagger UI: `http://localhost:8000/docs`)

### Auth (`/auth`)

* 회원가입 / 로그인 / 로그아웃
* 내 정보 조회, 비밀번호 변경
* 사용자 목록/활성화 관리, 강제 로그아웃 (관리자)
* 세션 조회/삭제

### Users (`/users`)

* 사용자 활동 조회

### Documents (`/documents`)

* 문서 목록 조회 (카테고리/키워드 필터, 정렬, 페이지네이션)
* 문서 상세 조회 / 수정
* 문서 삭제 (Soft Delete)

### Document Pipeline (`/documents/pipeline`)

* 문서 업로드 및 파이프라인 시작
* Job 목록/상세 조회
* Job 취소

### Drafts (`/drafts`)

* 초안 생성 / 목록 조회 / 상세 조회 / 수정 / 삭제
* 결재 목록 조회 및 승인/반려 결정

### RAG (`/rag`)

* 질의응답 (`/query`, 스트리밍: `/query/stream`)
* 질의 이력 조회 (`/history`)

### Notifications (`/notifications`)

* SSE 구독 (`/subscribe`)
* 알림 목록 조회 / 읽음 처리

### Admin (`/admin`)

* 통계 조회, 전체 Job 목록 조회

---

## 12. Docker 실행 구조

전체 서비스는 루트의 `docker-compose.yml` 하나로 실행합니다.

```bash
docker compose up --build
```

| 서비스 | 설명 | 포트 |
| --- | --- | --- |
| `frontend` | React + Vite 개발 서버 | 5173 |
| `backend` | FastAPI (uvicorn --reload) | 8000 |
| `db` | PostgreSQL 16 | - |
| `redis` | Redis 7 | - |
| `ollama` | 로컬 LLM 추론 서버 | 11434 |
| `ollama-init` | 최초 1회 커스텀 모델(`gemma2_9b`, `ko-gemma-2-9b`) 생성 | - |

`ollama-init`은 `llm_models/Modelfile_rag_summary`, `llm_models/Modelfile_catagory`를 기반으로 커스텀 모델을 생성하는 1회성 초기화 컨테이너입니다.

---

## 13. 환경 변수 예시

루트 `.env` 파일:

```env
POSTGRES_DB=
POSTGRES_USER=
POSTGRES_PASSWORD=
DATABASE_URL=
REDIS_URL=
OLLAMA_BASE_URL=
CURRENT_MODEL=
SUMMARY_MODEL=
CLASSIFY_MODEL=
EMBEDDING_MODEL=
SUMMARY_NUM_PREDICT=
SECRET_KEY=
```

---

## 14. 문서 삭제 흐름

사용자가 업로드한 문서를 삭제할 경우 다음과 같이 처리합니다. (`document.py` soft delete)

```text
1. 사용자 삭제 요청
2. 쿠키 인증
3. 문서 소유자(user_id) 확인
4. doc 테이블의 is_deleted 플래그 처리 (Soft Delete)
5. 목록/상세 조회 API에서 삭제된 문서 자동 제외
```

가장 중요한 부분은 권한 확인입니다.

```text
doc.user_id == current_user.user_id
```

현재 로그인한 사용자가 문서의 소유자가 아니라면 조회·수정·삭제를 거부합니다.

---

## 15. 보안 설계

주요 보안 기준은 다음과 같습니다.

* 모든 문서 업로드·조회·RAG 질의 요청은 쿠키 인증 후 처리
* 비밀번호는 해시(passlib)로 저장, 쿠키는 python-jose로 서명
* 문서 조회/수정/삭제 시 소유자(user_id) 검증
* 역할(Role) 기반 결재(승인/반려) 권한 분리
* `.env` 파일 Git 업로드 금지
* 파이프라인 실패 시 트랜잭션 롤백 및 에러 단계/메시지 로깅

---

## 16. 프로젝트 완성 기준

본 프로젝트의 완성 기준은 다음과 같습니다.

* 사용자가 로그인/회원가입할 수 있다.
* 다양한 포맷의 문서를 업로드하고 OCR로 원문을 추출할 수 있다.
* 문서가 자동으로 분류되고 LLM 기반 요약이 생성된다.
* 파이프라인 진행 상태가 실시간(SSE)으로 표시된다.
* 업로드된 문서를 기반으로 RAG 질의응답이 가능하다.
* 답변에 참조 문서 출처가 표시된다.
* 요약을 기반으로 초안을 작성하고 승인/반려 결재가 가능하다.
* 전체 서비스가 `docker compose up` 한 번으로 로컬에서 완전히 실행된다.

---

## 17. 최종 목표

이 프로젝트는 단순한 문서 뷰어가 아니라, 외부 클라우드 LLM API 없이 로컬 환경에서 완결되는 문서 요약·검색·결재 시스템 구현을 목표로 합니다.

핵심은 다음과 같습니다.

* OCR 기반 다양한 포맷 지원
* 로컬 LLM(Ollama) 기반 요약/분류/RAG
* 벡터DB 기반 근거 검색 및 출처 표시
* 실시간 파이프라인 진행률(SSE)
* 초안-결재 워크플로우
* Docker Compose 기반 원클릭 실행 환경

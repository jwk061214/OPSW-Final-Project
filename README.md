# OPSW Final Project — Backend (FastAPI)

복약 관리 서비스의 백엔드 입니다.  
시니어(노인) 정보를 등록/조회하고, 약 봉투/처방전 이미지를 업로드하면 **AI 분석(Gemini 이미지 입력 기반)** 또는 **OCR(Google Vision) 기반**으로 복약 정보를 구조화한 뒤 **Firestore에 저장**합니다.

- Base URL (local): `http://127.0.0.1:8000`
- Swagger: `/docs`
- OpenAPI JSON: `/openapi.json`

---

## 1) Tech Stack

- **FastAPI** (Python) — REST API 서버
- **Firestore** (Google Cloud / Firebase) — 데이터 저장소
- **Firebase Admin SDK** — 인증 토큰 검증(선택 적용)
- **Google Cloud Vision** — 이미지 OCR(텍스트 추출)
- **Gemini (google-genai)** — 이미지 입력 기반 처방/약봉투 정보 구조화
- **Ollama** *(옵션)* — OCR 텍스트 기반 LLM 분석(로컬)

---

## 2) Backend Team R&R (요약)

- **20225104 강지우 (jwk061214)**
  - OCR 업로드 API 및 분석 엔드포인트(초기 구축)
  - 복약 스케줄/복약 로그 생성 및 액션 API(완료/스킵)
  - API 명세서(문서화) 정리
  - 시니어(elders) 생성/조회 API 정비 및 전체 플로우 버그 수정 지원

- **20235175 박태양 (ILYsun-hub)**
  - OCR/AI 결과를 Firestore에 저장하는 파이프라인 및 연동 로직
  - 보호자 연결/알림 기능 관련 백엔드 API 및 서비스 로직
  - 백엔드 기능 통합 과정에서의 구조 정리 및 연동 안정화

> 세부 PR/이슈 근거는 리포지토리의 Pull Requests 및 Issues 기록을 기준으로 확인할 수 있습니다.

---

## 3) Project Structure

```bash
backend/
├─ app/
│  ├─ main.py
│  ├─ core/
│  │  ├─ firestore_db.py        # Firestore client getter
│  │  ├─ firebase_init.py       # firebase_admin initialize
│  │  └─ auth.py                # get_current_user (optional)
│  ├─ routes/
│  │  ├─ elder_routes.py        # /api/elders
│  │  ├─ ocr_routes.py          # /api/ocr
│  │  └─ intake_routes.py       # /api/intake (복약 완료/스킵 등)
│  ├─ services/
│  │  ├─ elder_service.py
│  │  ├─ firestore_store.py
│  │  ├─ ocr_service.py         # Google Vision OCR
│  │  ├─ gemini_service.py      # Gemini image analysis
│  │  ├─ ollama_service.py      # (optional) text-based LLM
│  │  └─ ollama_prompts.py
│  └─ utils/
│     ├─ parser.py              # regex parser (OCR text)
│     └─ json_extractor.py      # safe JSON extraction from LLM output
├─ requirements.txt
└─ .env (local only, DO NOT COMMIT)
```

> ⚠️ `app/core/*.json` (Vision key, Firebase admin key) 는 **절대 커밋 금지** (gitignore 필수)  
> ⚠️ `.env`도 공개 저장소에 업로드하지 않습니다.

---

## 4) Environment Variables (.env)

`backend/.env` 파일을 생성하고 아래 값을 채우세요.

```dotenv
# Firestore (Firebase Admin SDK key)
FIREBASE_ADMIN_KEY=/absolute/path/to/firebase-adminsdk.json

# Google Vision OCR key (서비스 계정)
GOOGLE_APPLICATION_CREDENTIALS=/absolute/path/to/google-vision-key.json

# Gemini
GEMINI_API_KEY=YOUR_GEMINI_API_KEY
# 선택: 모델 바꿀 때
GEMINI_MODEL=gemini-2.0-flash
```

### Key 파일 위치 추천 (macOS)
- 예: `backend/app/core/<key>.json`에 두고
- `.env`에는 **절대 경로**를 넣는 방식을 추천합니다.

---

## 5) Installation & Run

### 5-1. 가상환경 생성 & 의존성 설치
```bash
cd backend
python -m venv env
source env/bin/activate

pip install -r requirements.txt
```

### 5-2. 서버 실행
```bash
uvicorn app.main:app --reload
```

### 5-3. 동작 확인
- `GET http://127.0.0.1:8000/`
- `GET http://127.0.0.1:8000/docs`

---

## 6) Architecture Overview

### 6-1. 권장 플로우: Gemini 이미지 입력 기반 분석

```text
[Flutter] 이미지 업로드
   ↓
[FastAPI] /api/ocr/upload/ai/save
   ↓
[Gemini] 이미지 → 복약정보 JSON 추출
   ↓
[Firestore]
  - prescription_scans 저장
  - prescriptions 저장
  - schedules 생성
  - intake_logs(날짜×시간대) 자동 생성
```

### 6-2. 대체 플로우: OCR 기반 분석

```text
이미지 업로드
  ↓
Google Vision OCR (raw_text 추출)
  ↓
(선택1) regex parser (parser.py)
(선택2) Ollama(text LLM) 분석
  ↓
Firestore 저장
```

---

## 7) API Endpoints (핵심)

> 최신 목록/스키마는 Swagger(`/docs`)를 기준으로 확인하세요.

### 7-1. Elders (시니어)

#### ✅ 시니어 생성
- **POST** `/api/elders`

Request (JSON)
```json
{
  "user_id": "test_user_001",
  "name": "홍길동",
  "birth": "1948-03-01",
  "gender": "M"
}
```

Response
```json
{
  "elder_id": "uuid..."
}
```

#### ✅ 사용자 기준 시니어 목록 조회
- **GET** `/api/elders?user_id=test_user_001`

Response
```json
{
  "elders": [
    {
      "elder_id": "uuid...",
      "user_id": "test_user_001",
      "name": "홍길동",
      "birth": "1948-03-01",
      "gender": "M",
      "created_at": "..."
    }
  ]
}
```

---

### 7-2. OCR / AI 분석

#### ✅ OCR(텍스트 추출 + 파서)
- **POST** `/api/ocr/upload`
- `multipart/form-data`로 `file=<image>`

Response (예시)
```json
{
  "success": true,
  "data": {
    "raw_text": "....",
    "parsed": {
      "drugs": [
        {
          "drug_name": "...",
          "dose": "300mg",
          "amount_per_dose": "1",
          "times_per_day": "3",
          "days": "7"
        }
      ]
    }
  }
}
```

#### ✅ OCR + Ollama 분석(옵션)
- **POST** `/api/ocr/upload/ai`
- OCR 후 텍스트를 Ollama에 보내 `drugs[]` JSON을 받는 방식

Response (예시)
```json
{
  "success": true,
  "raw_text": "...",
  "ai_parsed": {
    "drugs": []
  }
}
```

#### ✅ Gemini 이미지 분석 + Firestore 저장 (권장)
- **POST** `/api/ocr/upload/ai/save?elder_id=elder_001`
- `multipart/form-data`로 `file=<image>`

동작:
1) Gemini로 이미지 분석 → `drugs[]` 생성  
2) Firestore에 scan/prescription/schedule/intake_logs 저장

Response (예시)
```json
{
  "success": true,
  "message": "OCR + AI + 복약 스케줄 생성 완료",
  "elder_id": "elder_001",
  "scan_id": "...",
  "prescription_id": "...",
  "medicine_ids": ["...", "..."],
  "schedule_ids": ["...", "..."],
  "ai_parsed": {
    "drugs": [
      {
        "drug_name": "",
        "dose": "",
        "amount_per_dose": "",
        "times_per_day": "",
        "days": ""
      }
    ]
  }
}
```

---

## 8) Firestore Data Model (요약)

- `elders`
  - `user_id`, `name`, `birth`, `gender`, `created_at`
- `prescription_scans`
  - `user_id`, `elder_id`, `raw_ocr_text`, `created_at`
- `prescriptions`
  - `user_id`, `elder_id`, `scan_id`, `medicine_ids[]`, `created_at`
- `schedules`
  - `user_id`, `elder_id`, `medicine_id`, `drug_name`, `dose`, `times[]`, `start_date`, `end_date`
- `intake_logs`
  - `user_id`, `elder_id`, `schedule_id`, `medicine_name`, `dose`,
  - `target_date`, `planned_time`, `slot`,
  - `status: pending | taken | skipped`, `taken_time`, `skipped_reason`

---

## 9) Authentication (Optional)

Firebase Auth 토큰 사용 시:
- `Authorization: Bearer <token>`
- `app/core/auth.py`의 `get_current_user()`로 uid 추출

> 개발 단계에서는 테스트를 위해 `user_id="test_user_001"` 같은 값이 임시로 사용될 수 있습니다.  
> 실제 연동 시 해당 부분을 `get_current_user()` 기반으로 교체하세요.

---

## 10) Troubleshooting

### 10-1. `FileNotFoundError: FIREBASE_ADMIN_KEY ...`
- `.env` 경로가 실제 파일 경로와 일치하는지 확인
- `~` 대신 절대경로 권장

```bash
python - <<EOF
import os
print("FIREBASE_ADMIN_KEY =", os.getenv("FIREBASE_ADMIN_KEY"))
EOF
ls -al /absolute/path/to/firebase-adminsdk.json
```

### 10-2. `ImportError: cannot import name 'genai' from 'google'`
- `google` 이름의 다른 패키지와 충돌 가능성
- 아래처럼 정리 후 재설치 권장

```bash
pip uninstall -y google
pip install -U google-genai
```

### 10-3. Firestore 관련 `NameError: db is not defined`
- 전역 `db`를 사용하지 말고, 각 함수 내부에서 `db = get_db()`로 통일하세요.

---

## 11) License / Third Party Notices

- 프로젝트 루트의 `LICENSE` 및 `THIRD_PARTY_NOTICES` 문서를 확인하세요.
- 키 파일(`*.json`)과 `.env`는 공개 저장소에 업로드하지 않습니다.

---

_Last updated: 2025-12-22_

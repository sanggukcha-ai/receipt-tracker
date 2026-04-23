# CLAUDE.md

이 파일은 이 저장소에서 작업하는 Claude Code(claude.ai/code)에게 프로젝트 안내를 제공합니다.

## 응답 언어

항상 **한국어**로 답변한다. 전문 용어(라이브러리명, 기술 용어, 명령어 등)는 영어 그대로 사용 가능.

## 프로젝트 개요

**영수증 지출 관리 앱 (Receipt Expense Tracker)** — 영수증 이미지/PDF를 업로드하면 Upstage Vision LLM이 자동으로 파싱하고, 결과를 JSON 지출 기록으로 저장합니다. 1인 개인 가계부 도구, 1일 스프린트 프로젝트.

- **현황**: 기획/명세 완료, 구현 미착수
- **스택**: React 18 + Vite 5 + TailwindCSS 3 (프론트엔드) | Python FastAPI 0.111 + LangChain 0.2 + Upstage Vision LLM (백엔드) | JSON 파일 저장 | Vercel 배포

## 예정 디렉토리 구조

```
receipt-tracker/
├── frontend/
│   ├── src/
│   │   ├── pages/          # Dashboard.jsx, UploadPage.jsx, ExpenseDetail.jsx
│   │   ├── components/     # DropZone, ParsePreview, ExpenseCard, SummaryCard, FilterBar, Badge, Modal, Toast
│   │   └── api/axios.js    # Axios 인스턴스 + API 함수
│   ├── package.json
│   └── vite.config.js
├── backend/
│   ├── main.py             # FastAPI 진입점, CORS, 라우터 등록
│   ├── routers/            # upload.py, expenses.py, summary.py
│   ├── services/
│   │   ├── ocr_service.py      # LangChain 체인 + Upstage Vision LLM
│   │   └── storage_service.py  # expenses.json 읽기/쓰기
│   ├── data/expenses.json
│   └── requirements.txt
├── vercel.json
└── .env                    # UPSTAGE_API_KEY (이미 존재)
```

## 개발 명령어

### 백엔드
```bash
cd backend
python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate
pip install -r requirements.txt
uvicorn main:app --reload       # http://localhost:8000 | Swagger: /docs
```

### 프론트엔드
```bash
cd frontend
npm install
npm run dev          # http://localhost:5173
npm run build        # Vercel 정적 호스팅용 dist/ 생성
npm run preview      # 프로덕션 빌드 로컬 테스트
```

## 아키텍처

### OCR 처리 흐름
```
파일 업로드 → PIL/pdf2image → Base64 인코딩
  → LangChain Chain (ChatUpstage + PromptTemplate + JsonOutputParser)
  → Upstage document-digitization-vision 모델
  → 구조화 JSON → expenses.json (append 저장)
```

### 핵심 설계 제약
- **Vercel 서버리스**: `/tmp`는 요청 간 유지되지 않음. MVP 해결책: 브라우저 `localStorage`에 병행 저장. 장기적으로는 Railway/Render(파일 시스템 유지) 또는 Supabase 도입.
- **OCR 시스템 프롬프트**: LLM에게 "JSON 형식으로만 응답"을 명시해야 함. 이 지시가 없으면 `JsonOutputParser`가 예외를 발생시킴.
- **파일 유효성 검사**: 확장자뿐 아니라 MIME 타입을 서버 측에서 검증. 10MB 제한 강제 적용.

## API 엔드포인트

| 메서드 | 경로 | 설명 |
|--------|------|------|
| POST | `/api/upload` | 영수증 업로드, OCR 실행, 파싱된 JSON 반환 |
| GET | `/api/expenses?from=&to=` | 지출 목록 조회 (날짜 필터 선택사항) |
| DELETE | `/api/expenses/{id}` | UUID로 삭제 |
| PUT | `/api/expenses/{id}` | 부분 수정 |
| GET | `/api/summary?month=YYYY-MM` | 합계 및 카테고리별 통계 |

## 지출 데이터 스키마

```json
{
  "id": "uuid-v4",
  "created_at": "ISO 8601 UTC",
  "store_name": "string (필수)",
  "receipt_date": "YYYY-MM-DD (필수)",
  "receipt_time": "HH:MM (선택)",
  "category": "식료품|외식|교통|쇼핑|의료|기타",
  "items": [{"name": "", "quantity": 0, "unit_price": 0, "total_price": 0}],
  "subtotal": 0,
  "discount": 0,
  "tax": 0,
  "total_amount": 0,
  "payment_method": "선택",
  "raw_image_path": "선택"
}
```

## 환경변수

| 변수명 | 위치 | 용도 |
|--------|------|------|
| `UPSTAGE_API_KEY` | 백엔드 `.env` + Vercel 환경변수 | Upstage API 인증 |
| `VITE_API_BASE_URL` | Vercel 프론트엔드 빌드 | 백엔드 기본 URL (비어있으면 동일 도메인) |
| `DATA_FILE_PATH` | Vercel 백엔드 | expenses.json 저장 경로 (`/tmp/` 사용) |

프론트엔드 환경변수는 Vite 빌드 시 주입되려면 반드시 `VITE_` 접두사가 필요합니다.

## Vercel 배포

```json
// vercel.json
{
  "builds": [
    { "src": "frontend/package.json", "use": "@vercel/static-build" },
    { "src": "backend/main.py", "use": "@vercel/python" }
  ],
  "routes": [
    { "src": "/api/(.*)", "dest": "backend/main.py" },
    { "src": "/(.*)", "dest": "frontend/dist/$1" }
  ]
}
```

## UI 디자인 토큰 (TailwindCSS)

- 주요 색상: `indigo-600` / hover `indigo-700`
- 배경: `gray-50`, 카드 표면: `white`, 테두리: `gray-200`
- 성공 `green-500`, 경고 `amber-500`, 오류 `red-500`
- 폰트: Pretendard → Noto Sans KR 폴백
- 레이아웃: `max-w-4xl mx-auto`, 반응형 그리드 `grid-cols-1 sm:grid-cols-2 lg:grid-cols-3`

`tailwind.config.js`에 커스텀 애니메이션 등록 필요: `slide-up` (Toast), `scale-in` (Modal), `fade-in` (페이지 전환).

## 바이브 코딩 3원칙 (PRD 13장)

**원칙 1 — 코딩 전에 "완료 기준" 체크리스트를 먼저 작성** — 각 Phase 시작 전, 3~5개의 수락 기준을 Claude에게 먼저 요청한다. 체크리스트가 명확해진 후에 구현을 시작한다.

**원칙 2 — 새로운 기술은 "조사 먼저, 구현 나중"** — 낯선 라이브러리(`langchain-upstage`, `pdf2image`, Vercel Python 서버리스)를 사용하기 전에 context7으로 최신 문서를 먼저 조회한다. 버전 호환성과 Vercel 제약사항을 확인한 후 코드를 작성한다.

**원칙 3 — 버그는 "분석 먼저, 수정 나중"** — 오류 발생 시 수정 전에 원인 분석을 먼저 요청한다. 어떤 흐름에서 왜 발생했는지 설명하고 수정 방향을 제안받은 후 구현한다. 표면적인 패치를 피한다.

## 사용 가능한 MCP 서버

- **context7**: 라이브러리 최신 문서 조회 (LangChain, Upstage, FastAPI, Vite 등에 활용)
- **github**: GitHub API 작업
- **playwright**: E2E 테스트용 브라우저 자동화

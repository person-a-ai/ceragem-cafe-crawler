# Ceragem Naver Cafe Crawler (Selenium Pipeline)

네이버 카페 비공개·등업 권한 게시판 대상 데이터 수집 및 정제 파이프라인.  
크롬 세션 보존, 개편된 네이버 로그인 규격 대응, 타겟 게시판(화이트리스트) 필터링, 대댓글 계층·익명화 해시가 포함된 표준 JSONL 및 정제 CSV 데이터셋 생성 시스템.

---

## 기술 스택

| 구분 | 기술 |
|------|------|
| **Language & Runtime** | Python 3.11 · JupyterLab / Jupyter Notebook |
| **Automation & Scraping** | Selenium WebDriver 4.51 · Chrome Profile · BeautifulSoup4 · lxml |
| **Data Processing** | Pandas · hashlib (SHA-256) · unicodedata (NFC) · JSON / JSONLines |
| **Authentication** | JavaScript DOM Injection · Persistent Session Cookie |

---

## 프로젝트 구조

```
crawling_selenium/
├── README.md                                      # 프로젝트 종합 문서 (본 파일)
├── [코인원]크롤링_네이버카페_ver4.ipynb             # 베이스라인 원본 노트북
├── [코인원]크롤링_네이버카페_ver4_주석해설.ipynb     # 전 라인 주석 해설 노트북
│
├── 개선안_csv/                                     # CSV 기반 크롤러 파이프라인
│   ├── [세라젬]크롤링_네이버카페_csv_ver1.ipynb     # 게시판명 정제 + TARGET_BOARDS + ./data_csv 분리 저장
│   ├── data_csv/                                  # 키워드별 CSV 및 TOTAL 병합 파일
│   └── naver_selenium_profile/                    # 네이버 영구 로그인 세션 프로필
│
└── 개선안_jsonl/                                   # DCX 표준 JSONL 크롤러 파이프라인
    ├── [세라젬]크롤링_네이버카페_jsonl_ver1.ipynb   # 구조화 댓글 + doc_id + 해시 익명화 ./data_selenium 저장
    ├── data_selenium/                             # 키워드별 JSONL 및 TOTAL 병합 파일
    └── naver_selenium_profile/                    # 네이버 영구 로그인 세션 프로필
```

---

## 파이프라인 흐름

```
세션 확인/로그인 → f-e 검색 URL 조합 → 검색 결과 대기 → 링크 추출 → 본문/댓글 파싱 → 타겟 게시판 검증 → 데이터 저장(CSV/JSONL) → 통합(TOTAL) 병합
```

| 단계 | 설명 |
|------|------|
| **1. 인증** | 로컬 크롬 프로필(`naver_selenium_profile`) 세션 점검, 미로그인 시 자바스크립트 주입으로 안전 로그인 |
| **2. 검색 쿼리** | 네이버 카페 f-e 신규 규격(`menus/0`) 기반 키워드(AND 연산), 기간, 페이지 파라미터 조합 |
| **3. 목록 수집** | 검색 결과 로딩 대기 후 쿼리 파라미터를 정제한 순수 URL(`clean_url`) 기준 중복 없이 추출 |
| **4. 본문 파싱** | 구형 iframe(`cafe_main`) 및 스마트에디터 ONE 자동 감지, 본문·댓글 동적 스크롤 로딩 |
| **5. 타겟 필터링** | `TARGET_BOARDS` 화이트리스트 검사 (지정된 핵심 게시판만 수집, 비대상 게시판 자동 스킵) |
| **6. 예외 방어** | 비공개 글/권한 부족 Alert(`alert("접근 권한이 없습니다")`) 자동 감지 및 닫기(`accept`) 스킵 |
| **7. 데이터 저장** | 실행 충돌 방지를 위해 포맷별 디렉토리(`data_csv`, `data_selenium`) 분리 저장 (CSV는 `utf-8-sig`, JSONL은 UTF-8 1라인 1레코드) |
| **8. 최종 병합** | 개별 키워드 파일들을 디스크에서 탐색(`glob`)하여 `_TOTAL_` 통합 파일 자동 생성 |

---

## 대상 카페 정보 (CAFE_TARGETS)

네이버 신규 f-e API는 문자열 주소(Slug) 대신 숫자 고유 ID(`CAFE_ID`)를 사용합니다.

| 카페명 | 카페 주소 (Slug) | 고유 `CAFE_ID` | 비고 |
| :--- | :--- | :---: | :--- |
| **은퇴 후 50년** | `https://cafe.naver.com/dlxogns01` | **`23676262`** | 노후 준비, 은퇴 라이프스타일 니즈 파악 |
| **지성 아빠의 나눔세상** | `https://cafe.naver.com/kimyoooo` | **`11289639`** | 귀농·귀촌, 영농 경험, 일상 Pain Point 파악 |
| **귀인** | `https://cafe.naver.com/makeho` | **`21770850`** | 귀촌 정착 과정 생활 맥락 및 주거 니즈 파악 |

---

## 데이터 스키마 명세

### 1. JSONL 레코드 구조 (`개선안_jsonl`)

DCX 및 페르소나 분석 자동화 파이프라인 연동용 표준 규격입니다 (`board_title`을 제외한 모든 요소가 `dcx_agent` 카페 크롤링 결과 레코드 구조와 100% 동일합니다):

```json
{
  "doc_id": "c1_cafe_naver_com_12345678",
  "source": "naver_cafe",
  "src_meta": {
    "cafe": "은퇴 후 50년",
    "board_title": "귀촌 정착 이야기",
    "comment_count": 2,
    "comment_threads": [
      {"id": "1", "parent": null},
      {"id": "2", "parent": "1"}
    ]
  },
  "kw": "수면 빛",
  "kw_axis": "",
  "kw_sub": "",
  "kw_hits": ["수면", "빛"],
  "title": "수면 시 암막 커튼 관련 질문드립니다",
  "body": "새로 이사 온 집 침실에 가로등 빛이 들어와서 수면에 방해가 되네요...",
  "comments": [
    {
      "text": "100% 암막 커튼 달면 확실히 깊게 잠듭니다.",
      "depth": 0,
      "date": "2026-10-24",
      "author_hash": "a8f3b291"
    },
    {
      "text": "저도 같은 문제로 고생하다가 암막 설치하고 해결했어요.",
      "depth": 1,
      "date": "2026-10-24",
      "author_hash": "c4d1e892"
    }
  ],
  "date": "2026-10-24",
  "url": "https://cafe.naver.com/dlxogns01/12345678",
  "fetch_level": "full",
  "access": "public",
  "snippet": "",
  "author_hash": "f4c91d8a",
  "crawled_at": "2026-10-24T12:30:00Z"
}
```

---

### 2. CSV 레코드 구조 (`개선안_csv`)

엑셀 및 데이터프레임 기반 통계 분석용 2차원 표 형식입니다 (`utf-8-sig` 인코딩, 게시글 1개당 1개 Row):

| 컬럼명 | 타입 | 설명 | 추출 형식 및 비고 |
|:---|:---:|:---|:---|
| **`date`** | String | 게시글 작성일자 | `YYYY-MM-DD` (상대시간은 당일 날짜로 통일) |
| **`keyword`** | String | 수집 시 사용한 검색 키워드 | 예: `수면 빛`, `귀촌 주거` |
| **`title`** | String | 게시글 제목 | HTML 태그 정제된 순수 제목 텍스트 |
| **`contents`** | String | 본문 전체 원문 | 연속 공백 정제된 순수 본문 텍스트 |
| **`comments`** | String | 전체 댓글 본문 | 여러 댓글을 **줄바꿈(`\n`)**으로 연결하여 1개 셀에 저장 |
| **`board_titles`**| String | 게시판 카테고리 명칭 | 상위 헤더가 아닌 순수 게시판명 (예: `자유게시판`) |
| **`url`** | String | 원본 게시글 상세 URL | 파라미터가 정제된 순수 URL (중복 제거 기준 키) |
| **`author_nickname`** | String | 본문 작성자 닉네임 | 작성자 닉네임 (미노출 시 `닉네임 없음`) |
| **`comment_nicknames`** | String | 댓글 작성자 닉네임 목록 | 여러 댓글 작성자를 **파이프(`\|`)** 기호로 연결 |

#### CSV 레코드 예시 (1개 행)
```csv
date,keyword,title,contents,comments,board_titles,url,author_nickname,comment_nicknames
"2026-10-24","수면 빛","수면 시 암막 커튼 관련 질문드립니다","새로 이사 온 집 침실에 가로등 빛이 들어와서 수면에 방해가 되네요...","100% 암막 커튼 달면 확실히 깊게 잠듭니다.
저도 같은 문제로 고생하다가 암막 설치하고 해결했어요.","귀촌 정착 이야기","https://cafe.naver.com/dlxogns01/12345678","달빛사냥꾼","숙면요정 | 잠이솔솔"
```

---

## 환경 설정 및 실행 가이드

### 1. 사전 요구사항
* **Python 3.11** (권장 표준 런타임)
* **Google Chrome** 최신 브라우저

### 2. 패키지 설치
터미널에서 아래 명령어로 필수 라이브러리를 설치합니다:

```bash
pip3 install selenium beautifulsoup4 lxml pandas tqdm pyperclip pyautogui webdriver-manager ipykernel
```

### 3. JupyterLab 실행
```bash
cd "/Users/gunmandu/Desktop/대학원/연구실/세라젬/crawling_selenium"
jupyter lab
```

### 4. 핵심 설정값 가이드 (Cell 5)

```python
# 1. 로그인 정보
my_id = 'your_naver_id'
my_pw = 'your_naver_password'

# 2. 수집 대상 카페 설정
CAFE_URL = 'https://cafe.naver.com/dlxogns01'
CAFE_ID = '23676262'
FILE_PREFIX = '은퇴 후 50년'
FILE_AUTHOR = '담당자명'

# 3. 수집 대상 게시판 화이트리스트 (빈 리스트 [] 이면 전체 수집)
TARGET_BOARDS = []  # 특정 게시판만 원할 경우: ['은퇴준비', '귀촌일기']

# 4. 수집 기간 및 페이징
DATE_START = '20240101'
DATE_END   = '20260420'
MAX_PAGE  = 40
PAGE_SIZE = 50

# 5. 저장 경로 (CSV는 data_csv, JSONL은 data_selenium 기본값)
SAVE_DIR = './data_csv'  # 또는 './data_selenium'
```

### 5. 키워드 설정 가이드 (Cell 7)
네이버 카페 검색 엔진은 **띄어쓰기(공백)를 기본 AND 연산**으로 처리합니다:

CSV 노트북은 메인 키워드와 서브 키워드의 모든 조합을 공백으로 연결해 순서대로 수집합니다.

```python
main = ["귀농"]
sub = ["건강식", "아침", "운동"]
# 생성되는 검색어: 귀농 건강식, 귀농 아침, 귀농 운동
```

메인 키워드가 여러 개면 각 메인마다 모든 서브 키워드를 검색합니다. 빈 항목과 중복은 제외하며,
메인 또는 서브 목록이 비어 있으면 설정 오류로 중단합니다. CSV의 `keyword`와 파일명에도 조합된
검색어가 들어갑니다. 설정 변경 후에는 키워드 셀과 크롤링 실행 셀을 다시 실행하세요.
기본 설정은 기존 검색어를 유지하도록 `main = ["수면"]`, `sub = ["부정맥", "빛"]`입니다.
JSONL 노트북은 기존 `keyword` 목록 설정을 사용합니다.

---

## 트러블슈팅 및 운영 팁

* **자동입력방지문자(캡차)가 반복해서 뜰 때**:
  * 네이버 로그인 화면에서 **`[v] 로그인 상태 유지`** 체크박스를 반드시 체크하고 최초 1회 로그인해 주세요. 영구 쿠키가 `naver_selenium_profile`에 저장되어 이후에는 로그인 없이 즉시 통과합니다.
* **Mac에서 정지(Stop)가 안 먹힐 때**:
  * 주피터 상단 툴바의 **`Restart Kernel (🔄)`** 버튼을 누르시면 안전하게 프로세스가 리셋됩니다.
* **주석 토글 단축키**:
  * 여러 줄을 블록 지정한 뒤 **`Cmd + /`**를 누르면 한 번에 주석 처리/해제됩니다.
* **노션 변경 이력**:
  * 모든 개선 및 수정 내역은 [세라젬 크롤링 노션 문서](https://app.notion.com/p/402596920133801b8970dbaf2bff3532)에 실시간으로 기록 및 관리되고 있습니다.

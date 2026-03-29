# 영어 초대규모 단어 수집 시스템 통합 설계서

## 1. 작업 컨텍스트 문서

### 1.1 작업명
영어 초대규모 단어 수집 및 누적 확장 시스템

### 1.2 배경
일반 영어사전 수준의 단어집은 규모와 갱신성이 제한적이다. 이번 프로젝트의 목적은 뜻·품사·예문 설명 없이도, 영어로 쓰이는 문자열을 가능한 한 많이 수집하여 초대규모 단어 저장소를 구축하는 것이다. 목표는 1만~10만 단어 수준이 아니라, 반복 실행을 통해 수억 개 이상의 고유 영어 단어/표기 후보를 누적하는 것이다.

### 1.3 목적
- 전 세계 웹, 공개 사전, 공개 덤프, 공개 코퍼스, 공개 워드리스트에서 영어 단어를 최대한 많이 수집한다.
- 한 번 실행으로 끝나는 수집기가 아니라, 매번 실행할 때마다 기존 저장소에 **중복되지 않은 단어만 추가**되는 구조를 만든다.
- 로컬 PC에서 운용 가능해야 하며, 향후 소스 확장 및 반복 실행에 유리해야 한다.
- LLM/AI가 꺼내 쓰기 쉬운 형식으로 저장해야 한다.

### 1.4 범위
#### 포함
- 일반 단어
- 활용형, 복수형, 파생형
- 고유명사
- 약어
- 하이픈 단어
- 붙여쓴 합성어
- 인터넷 슬랭
- 고어/방언
- 실제 웹에 등장하는 변형 표기
- 오타처럼 보이지만 실제 대규모 코퍼스에 반복 등장하는 문자열

#### 제외
- 단어 의미/예문/사전식 정의 작성
- 사람이 직접 단어를 수동 정리하는 편집 워크플로우
- 완전무결한 “진짜 영어 단어만” 정제하는 보수적 사전 구축

### 1.5 입력
- 공개 소스 목록 정의 파일
- 각 소스의 다운로드 URL / 크롤링 URL / API endpoint / 덤프 패턴
- 이전 실행에서 누적 저장된 단어 레지스트리
- 실행 설정 파일(배치 크기, 소스 우선순위, 필터 규칙, 저장 경로 등)

### 1.6 출력
- 영구 누적 단어 저장소
- 이번 실행에서 새로 추가된 단어 목록
- 소스별 수집 통계
- 실패 로그 및 재시도 로그
- AI/스크립트가 쉽게 조회 가능한 단어 샤드 파일
- 전체 메타데이터 DB

### 1.7 주요 제약조건
- 로컬 PC에서 돌아가야 한다.
- 데이터 규모는 수억 개 이상을 목표로 하므로 메모리 내 전체 적재 방식은 금지한다.
- 반복 실행 시 이미 수집된 단어를 다시 저장하지 않아야 한다.
- “영어 단어”를 엄격히 제한하지 않고, 수집량 최대화를 최우선으로 한다.
- 고품질 단어층과 대규모 웹 후보층을 분리해 관리해야 한다.

### 1.8 핵심 설계 원칙
1. **수집량 우선**: 정제 품질보다 누적 고유 문자열 확보를 우선한다.
2. **증분 수집**: 매 실행마다 새 단어만 추가한다.
3. **계층 저장**: 신뢰도 높은 사전층과 대규모 웹층을 분리 저장한다.
4. **해시 기반 중복 제거**: 문자열 원본 비교가 아니라 해시 키 기반으로 빠르게 중복 제거한다.
5. **대용량 친화성**: 스트리밍 처리, 샤딩, 압축 저장을 기본으로 한다.
6. **재현 가능성**: 어떤 소스에서 어떤 규칙으로 추출되었는지 기록한다.

### 1.9 용어 정의
- **단어(word)**: 이번 프로젝트에서는 영어권에서 쓰인 것으로 보이는 문자열 토큰 전반을 의미한다.
- **후보(candidate)**: 아직 최종 저장소에 반영되지 않은 추출 문자열.
- **정규화(normalization)**: 비교를 위한 최소 변환(유니코드 정규화, 공백/제어문자 제거 등).
- **캐노니컬 키(canonical key)**: 중복 제거용 표준화 키.
- **원본 토큰(surface form)**: 실제 소스에서 추출된 원형 문자열.
- **레지스트리(registry)**: 이미 본 단어인지 확인하는 영구 고유 키 저장소.
- **샤드(shard)**: 대규모 저장소를 여러 파일로 분할한 단위.

---

## 2. 수집 전략 개요

### 2.1 전체 전략
최종적으로는 아래 4계층을 함께 사용한다.

1. **고품질 사전 계층**
   - Wiktionary dump / Kaikki 추출 데이터 / SCOWL / WordNet 등
   - 장점: 품질이 높고 기본 어휘층 확보에 유리
   - 단점: 수억 개 규모 확장에는 한계

2. **대규모 웹 코퍼스 계층**
   - Common Crawl WET/WARC 기반 대량 텍스트 추출
   - 장점: 규모가 압도적이며 신조어/약어/오타/고유명사/인터넷 표기까지 확보 가능
   - 단점: 노이즈가 매우 많음

3. **타이틀/엔티티 계층**
   - Wikipedia titles, 위키 계열 title dump, 공개 entity list
   - 장점: 고유명사와 합성어 확장에 유리

4. **증분 확장 계층**
   - 실행 이력 기반 신규 소스만 추가 수집
   - 기존 저장소와 비교하여 미수집 단어만 누적

### 2.2 왜 이 구조가 필요한가
사전/워드리스트만으로는 “많이 모아도 수백만~수천만 수준”에 머무를 가능성이 크다. 반면, Common Crawl 같은 웹 전체 코퍼스를 쓰면 수집 후보를 폭발적으로 늘릴 수 있다. 따라서 실제 목표가 “수억 개 이상의 고유 문자열 확보”라면 웹 코퍼스 계층이 필수다.

### 2.3 현실적 한계
“전 세계 사람들이 쓰는 영어단어를 다 수집”하는 것은 종료 가능한 완결 과제가 아니라, 계속 확장되는 오픈엔드 수집 문제다. 따라서 이 프로젝트는 **완전 수집 보장 시스템**이 아니라 **반복 실행으로 계속 커지는 누적 수집 시스템**으로 정의한다.

---

## 3. 데이터 소스 설계

### 3.1 1차 우선 소스 (반드시 구현)

#### A. Common Crawl
- 목적: 초대규모 웹 텍스트에서 영어 토큰 대량 확보
- 방식: WET 파일(본문 텍스트), 필요 시 WARC/인덱스 보조 사용
- 주소
  - `https://commoncrawl.org/`
  - `https://commoncrawl.org/get-started`
  - 크롤 단위 경로 예시: `https://data.commoncrawl.org/crawl-data/CC-MAIN-YYYY-NN/`
- 추출 대상
  - 일반 웹문서 본문
  - 블로그/포럼/문서 사이트/쇼핑몰/뉴스/개발문서 등
- 비고
  - 규모 확보의 핵심 소스
  - 월별 신규 크롤을 증분 대상으로 삼는다.

#### B. English Wiktionary dump
- 목적: 사전형 고품질 단어층 확보
- 주소
  - `https://dumps.wikimedia.org/enwiktionary/latest/`
  - 예: `enwiktionary-latest-pages-articles-multistream.xml.bz2`
  - 예: `enwiktionary-latest-all-titles-in-ns0.gz`
- 추출 대상
  - 표제어
  - 변형 표기
  - 영어 항목 내 단어형

#### C. Kaikki raw data (Wiktionary 기계 판독 추출본)
- 목적: Wiktionary 원본 XML보다 파싱 부담을 줄이고 빠르게 구조화 데이터 확보
- 주소
  - `https://kaikki.org/dictionary/rawdata.html`
  - `https://kaikki.org/dictionary/English/index.html`
- 추출 대상
  - English entries
  - forms / redirects / variant spellings

#### D. SCOWL / English Speller Database
- 목적: 신뢰도 높은 철자 기반 워드리스트 초기 seed 확보
- 주소
  - `https://wordlist.aspell.net/`
  - `https://wordlist.aspell.net/dicts/`
- 추출 대상
  - dialect별 word list
  - inflection/variant 관련 리스트

#### E. WordNet
- 목적: 기본 영단어망 및 파생 확장 보조
- 주소
  - `https://wordnet.princeton.edu/homepage`
- 추출 대상
  - lemma
  - multiword expression

#### F. English Wikipedia titles
- 목적: 고유명사·합성어·엔티티명 확장
- 주소
  - `https://dumps.wikimedia.org/enwiki/latest/`
  - 예: `enwiki-latest-all-titles-in-ns0.gz`
- 추출 대상
  - article title
  - redirect title

### 3.2 2차 확장 소스 (후속 반복 실행용)
- GitHub 공개 워드리스트 저장소
- 공개 subtitle / forum / news dump
- 공개 domain-specific glossary
- 오픈소스 spell dictionary
- 웹사이트 sitemap + 본문 크롤링 기반 도메인 확장

### 3.3 소스 우선순위
1. Wiktionary / Kaikki / SCOWL / WordNet / Wikipedia titles로 seed 구축
2. Common Crawl로 대규모 확장
3. 도메인 특화 웹/워드리스트로 롱테일 확장
4. 반복 실행 시 신규 월별 크롤 및 신규 덤프만 추가

---

## 4. 워크플로우 정의

## 4.1 단계별 전체 흐름

```text
[소스 목록 로드]
    ↓
[증분 대상 소스 판정]
    ↓
[다운로드/크롤링]
    ↓
[텍스트/엔트리 파싱]
    ↓
[토큰 추출]
    ↓
[정규화 + 캐노니컬 키 생성]
    ↓
[빠른 사전 필터(Bloom/Cuckoo)]
    ↓
[정확 중복 검사(영구 레지스트리)]
    ↓
[신규 단어만 저장]
    ↓
[샤드/메타DB 업데이트]
    ↓
[통계/로그/리포트 생성]
```

## 4.2 상태 전이
- `PENDING_SOURCE` → 아직 처리 전
- `DOWNLOADING` → 소스 다운로드 중
- `DOWNLOADED` → 로컬 저장 완료
- `PARSING` → 구조 해석 중
- `TOKENIZING` → 토큰 추출 중
- `DEDUP_CHECKING` → 중복 확인 중
- `COMMITTING` → 신규 토큰 저장 중
- `DONE` → 정상 완료
- `FAILED_RETRYABLE` → 재시도 가능 실패
- `FAILED_ESCALATED` → 수동 확인 필요
- `SKIPPED` → 선택적 단계 생략


## 4.3 다중 AI 더블체크 공통 프로토콜

본 프로젝트는 정확도를 높이기 위해, 각 단계의 **판단이 필요한 지점마다** 아래 합의 구조를 공통 적용한다.

### 공통 역할
- `main-orchestrator`: 전체 단계 진행과 최종 승인권 보유
- `domain-primary-agent`: 해당 단계의 1차 판단 담당 에이전트
- `consensus-reviewer-a`: 1차 결과 독립 검토
- `consensus-reviewer-b`: 2차 독립 검토
- `consensus-tiebreaker`: 1차/검토 결과가 충돌할 때 최종 판정

### 공통 절차
1. 스크립트가 단계 산출물 초안을 생성하거나, 1차 담당 에이전트가 판단안을 만든다.
2. `consensus-reviewer-a`, `consensus-reviewer-b`가 **서로의 답을 보지 않은 상태**로 독립 검토한다.
3. 세 판단이 모두 일치하거나, 3개 중 2개가 일치하면 합의 승인한다.
4. 불일치가 크면 `consensus-tiebreaker`가 로그/샘플/산출물을 근거로 최종 판정한다.
5. 합의 실패 시 해당 단계는 `REVIEW_CONFLICT`로 표시하고 재시도 또는 에스컬레이션한다.

### 적용 범위
- AI 판단이 필요한 모든 단계에 기본 적용
- 결정론적 처리 단계도 예외적으로 **이상 징후 탐지/로그 해석/샘플 검수**는 다중 에이전트 검토를 붙인다.
- QA 단계에서는 특히 다중 에이전트 검토를 강제하여, 실제 실행 결과 해석 오류를 줄인다.

## 4.4 단계별 정의

### 단계 1. 소스 인벤토리 로드
- 입력: source_registry.yaml
- 처리 내용: 사용 가능한 사전/API/덤프/크롤 타겟 목록 로드
- LLM 판단 vs 코드 처리
  - 에이전트가 직접 수행: 소스 우선순위 해석, 신규 소스 채택 여부 판단
  - 사용할 에이전트: `main-orchestrator`, `source-strategy-planner`, `consensus-reviewer-a`, `consensus-reviewer-b`, `consensus-tiebreaker`
  - 스크립트로 처리: 설정 파일 로드, 형식 검증
- 성공 기준: 모든 소스가 고유 ID, 버전/날짜, URL, 수집 방식, 라이선스 메타데이터를 가지며, 소스 채택 판단에 대해 다중 에이전트 합의가 완료됨
- 검증 방법: 스키마 검증
- 실패 시 처리: 자동 재시도 1회 후 에스컬레이션

### 단계 2. 증분 대상 판정
- 입력: source_registry, harvest_history, last_success markers
- 처리 내용: 이번 실행에서 새로 가져올 덤프/크롤/배치를 결정
- LLM 판단 vs 코드 처리
  - 에이전트가 직접 수행: 우선순위 조정, 저장 공간 대비 가치 판단
  - 사용할 에이전트: `main-orchestrator`, `incremental-harvest-planner`, `consensus-reviewer-a`, `consensus-reviewer-b`, `consensus-tiebreaker`
  - 스크립트로 처리: 이전 실행 이력 비교, 날짜 비교, 버전 비교
- 성공 기준: 이미 처리한 소스를 재다운로드하지 않고 신규/미완료 소스만 선별되며, 이번 실행 우선순위에 대해 다중 에이전트 합의가 완료됨
- 검증 방법: 규칙 기반
- 실패 시 처리: 자동 재시도 1회, 계속 실패 시 에스컬레이션

### 단계 3. 다운로드/크롤링
- 입력: 선택된 소스 목록
- 처리 내용: dump/API/web page/WET list 다운로드
- LLM 판단 vs 코드 처리
  - 에이전트가 직접 수행: 없음(원칙적으로 결정론적)
  - 사용할 에이전트: `download-integrity-reviewer`, `consensus-reviewer-a`, `consensus-reviewer-b`, `consensus-tiebreaker`
  - 스크립트로 처리: HTTP 다운로드, 압축 해제, 체크섬 검증, 재개 다운로드
- 성공 기준: 대상 파일이 손상 없이 저장되고 메타데이터가 기록되며, 체크섬/크기/샘플 오픈 결과에 대해 다중 에이전트 검토 합의가 완료됨
- 검증 방법: 규칙 기반 + 체크섬 검증
- 실패 시 처리: 자동 재시도 최대 3회

### 단계 4. 파싱
- 입력: XML/JSONL/TXT/WET/WARC/GZ/BZ2 파일
- 처리 내용: 각 포맷별 엔트리/본문 추출
- LLM 판단 vs 코드 처리
  - 에이전트가 직접 수행: 포맷 이상치 분류, 신규 포맷 대응 방침 결정
  - 사용할 에이전트: `parser-policy-agent`, `consensus-reviewer-a`, `consensus-reviewer-b`, `consensus-tiebreaker`
  - 스크립트로 처리: 스트리밍 파서, decompressor, line reader, WARC/WET parser
- 성공 기준: 파일 포맷별로 텍스트/엔트리를 누락 없이 스트리밍 추출하고, 포맷 이상치 대응 판단에 대해 다중 에이전트 합의가 완료됨
- 검증 방법: 규칙 기반(레코드 수, 샘플 파싱 성공률)
- 실패 시 처리: 자동 재시도 2회, 포맷 불명확 시 에스컬레이션

### 단계 5. 토큰 추출
- 입력: 파싱된 텍스트/표제어/타이틀
- 처리 내용: 영어 후보 문자열 분리
- LLM 판단 vs 코드 처리
  - 에이전트가 직접 수행: 토큰 경계 규칙 개선안 제안, 노이즈 패턴 분석
  - 사용할 에이전트: `token-policy-agent`, `consensus-reviewer-a`, `consensus-reviewer-b`, `consensus-tiebreaker`
  - 스크립트로 처리: regex/tokenizer 기반 추출, 길이 제한, 유니코드 범위 처리
- 성공 기준: 설정된 토큰화 규칙에 따라 후보가 대량 생성되고, 경계 규칙 예외 처리에 대해 다중 에이전트 합의가 완료됨
- 검증 방법: 규칙 기반 + 샘플 검수
- 실패 시 처리: 자동 재시도 1회, 규칙 문제면 에스컬레이션

### 단계 6. 정규화 및 캐노니컬 키 생성
- 입력: 후보 토큰
- 처리 내용: 비교용 표준화
- LLM 판단 vs 코드 처리
  - 에이전트가 직접 수행: 어떤 정규화가 과도한 병합을 일으키는지 판단
  - 사용할 에이전트: `normalization-reviewer`, `consensus-reviewer-a`, `consensus-reviewer-b`, `consensus-tiebreaker`
  - 스크립트로 처리: Unicode NFKC, 제어문자 제거, trim, case variant key 생성, 해시 생성
- 성공 기준: 동일 토큰 변형이 일관된 canonical key 체계로 변환되며, 과병합/과분리 여부에 대해 다중 에이전트 합의가 완료됨
- 검증 방법: 규칙 기반
- 실패 시 처리: 자동 재시도 1회

### 단계 7. 중복 제거
- 입력: canonical key, token hash
- 처리 내용:
  1. Bloom/Cuckoo filter로 빠른 1차 체크
  2. 영구 exact registry에서 최종 확인
  3. 미등록 토큰만 신규 반영
- LLM 판단 vs 코드 처리
  - 에이전트가 직접 수행: false positive/merge 문제 분석
  - 사용할 에이전트: `dedupe-auditor`, `consensus-reviewer-a`, `consensus-reviewer-b`, `consensus-tiebreaker`
  - 스크립트로 처리: key lookup, batch insert, shard merge
- 성공 기준: 기존 수집 단어가 다시 추가되지 않으며, dedupe 이상 징후 해석에 대해 다중 에이전트 합의가 완료됨
- 검증 방법: 규칙 기반 + 샘플 재주입 테스트
- 실패 시 처리: 자동 재시도 1회, registry 손상 시 에스컬레이션

### 단계 8. 저장소 커밋
- 입력: 신규 토큰 배치
- 처리 내용: 영구 단어 저장소 반영
- LLM 판단 vs 코드 처리
  - 에이전트가 직접 수행: 샤딩 재배치 정책 판단
  - 사용할 에이전트: `storage-strategy-agent`, `consensus-reviewer-a`, `consensus-reviewer-b`, `consensus-tiebreaker`
  - 스크립트로 처리: Parquet shard append/rewrite, LMDB/RocksDB insert, DuckDB metadata update
- 성공 기준: 신규 토큰과 메타데이터가 원자적으로 반영되며, 저장 전략 판단에 대해 다중 에이전트 합의가 완료됨
- 검증 방법: 스키마 검증 + 카운트 검증
- 실패 시 처리: 자동 재시도 1회, 실패 시 롤백 후 에스컬레이션

### 단계 9. 품질/노이즈 분석
- 입력: 이번 실행에서 새로 추가된 단어 샘플
- 처리 내용: 노이즈 비율, URL 조각, 코드 토큰, 깨진 문자열 비율 측정
- LLM 판단 vs 코드 처리
  - 에이전트가 직접 수행: 노이즈 유형 분석, 필터 규칙 개선 제안
  - 사용할 에이전트: `quality-auditor`, `consensus-reviewer-a`, `consensus-reviewer-b`, `consensus-tiebreaker`
  - 스크립트로 처리: 정규식 기반 통계, 샘플링
- 성공 기준: 이번 실행의 품질 리포트가 생성되고, 노이즈 유형 분류에 대해 다중 에이전트 합의가 완료됨
- 검증 방법: 규칙 기반 + LLM 자기 검증
- 실패 시 처리: 스킵 + 로그(선택적 단계)

### 단계 10. 리포트 및 증분 마커 기록
- 입력: 모든 실행 결과
- 처리 내용: 완료한 소스, 추가된 고유 단어 수, 누적 총량, 실패 내역 기록
- LLM 판단 vs 코드 처리
  - 에이전트가 직접 수행: 없음(요약 문장 생성만 선택적)
  - 사용할 에이전트: `report-generator`, `consensus-reviewer-a`, `consensus-reviewer-b`, `consensus-tiebreaker`
  - 스크립트로 처리: JSON/Markdown 보고서 생성
- 성공 기준: 다음 실행에서 증분 처리에 필요한 마커가 정확히 기록되고, 요약/이상 여부 판단에 대해 다중 에이전트 합의가 완료됨
- 검증 방법: 스키마 검증
- 실패 시 처리: 자동 재시도 1회

---

## 5. 판단과 코드의 역할 분리

본 섹션의 모든 판단 업무는 기본적으로 `도메인 담당 에이전트 1개 + 독립 리뷰 에이전트 2개 + 필요 시 tie-breaker 1개` 구조를 따른다.

| 업무 | 에이전트가 직접 수행 | 사용할 에이전트 | 스크립트로 처리 |
|---|---|---|---|
| 소스 우선순위 판단 | O | main-orchestrator, source-strategy-planner, consensus-reviewer-a, consensus-reviewer-b, consensus-tiebreaker | 소스 목록 로드, 이력 비교 |
| 신규 소스 채택 판단 | O | source-strategy-planner, consensus-reviewer-a, consensus-reviewer-b, consensus-tiebreaker | URL 검증, 메타데이터 저장 |
| 증분 수집 계획 결정 | O | incremental-harvest-planner, consensus-reviewer-a, consensus-reviewer-b, consensus-tiebreaker | last_success 비교, 배치 생성 |
| 파서 정책 선택 | O | parser-policy-agent, consensus-reviewer-a, consensus-reviewer-b, consensus-tiebreaker | XML/JSONL/WET/WARC 스트리밍 파싱 |
| 토큰화 규칙 개선 | O | token-policy-agent, consensus-reviewer-a, consensus-reviewer-b, consensus-tiebreaker | regex tokenizer 실행 |
| 정규화 정책 검토 | O | normalization-reviewer, consensus-reviewer-a, consensus-reviewer-b, consensus-tiebreaker | unicode 정규화, 해시 생성 |
| 중복 제거 이상 분석 | O | dedupe-auditor, consensus-reviewer-a, consensus-reviewer-b, consensus-tiebreaker | Bloom/Cuckoo lookup, exact registry lookup |
| 저장 전략 최적화 | O | storage-strategy-agent, consensus-reviewer-a, consensus-reviewer-b, consensus-tiebreaker | shard write, DB update |
| 품질/노이즈 정성 분석 | O | quality-auditor, consensus-reviewer-a, consensus-reviewer-b, consensus-tiebreaker | 규칙 기반 품질 통계 |
| 실행 요약 생성 | O | report-generator, consensus-reviewer-a, consensus-reviewer-b, consensus-tiebreaker | JSON/Markdown 리포트 생성 |

### 외부 에이전트 사용 여부
- 기본 설계는 Claude Code 내 메인 오케스트레이터 + 필요 시 서브에이전트 구조다.
- 외부에서 가져와 쓰는 에이전트는 필수가 아니다.
- 이유: 이 프로젝트는 대부분이 결정론적 데이터 파이프라인이고, AI는 정책/규칙 개선과 품질 감사에만 제한적으로 쓰는 것이 적절하다.

---

## 6. 반복 실행 시 중복 없이 누적 추가하는 구조

### 6.1 핵심 요구사항
사용자가 이번 대화에서 추가로 요구한 핵심 조건은 다음이다.
- 프로젝트를 매번 다시 실행해도 기존 단어를 중복 저장하지 않아야 한다.
- 매번 **기존에 없던 단어만** 추가해야 한다.

### 6.2 해결 방식
이를 위해 저장 구조를 3층으로 나눈다.

#### 1) Fast Seen Filter
- 역할: 이미 본 적 있는 해시인지 매우 빠르게 1차 판정
- 구현 후보: Bloom Filter 또는 Cuckoo Filter
- 특징: 메모리 효율이 높고 빠르지만 오탐(false positive)이 있을 수 있음

#### 2) Exact Seen Registry
- 역할: 최종 중복 판정의 기준
- 구현 후보: LMDB 또는 RocksDB key-value store
- 저장 키: `token_hash64` 또는 `token_hash128`
- 저장 값: 최소 메타데이터(최초 발견 소스, 최초 발견 일시, token length, flags)
- 특징: 오탐 없이 정확한 중복 제거 가능

#### 3) Token Warehouse
- 역할: 실제 단어 문자열 원본 저장
- 구현 후보: prefix/hash 기반 Parquet shard + Zstandard 압축
- 특징: 대용량 분석, export, 샘플링, AI 조회에 유리

### 6.3 신규 단어 추가 알고리즘

```text
for each extracted token:
    normalize token
    make canonical key
    hash = stable_hash(canonical key)

    if bloom_filter says "seen":
        check exact_registry
        if exists:
            skip
        else:
            insert exact_registry
            append token warehouse
            update bloom_filter
    else:
        insert exact_registry
        append token warehouse
        update bloom_filter
```

### 6.4 왜 exact registry가 필요한가
Bloom filter만 쓰면 false positive 때문에 실제 신규 단어가 누락될 수 있다. 따라서 **Bloom/Cuckoo는 속도용**, **LMDB/RocksDB exact registry는 정확성용**으로 함께 사용한다.

### 6.5 실행 재개 방식
- 각 소스별 `last_processed_offset`, `last_processed_file`, `last_processed_line`, `crawl_id` 저장
- 다운로드 중단 시 재시작 가능
- 토큰 커밋은 배치 단위로 수행하여 중간 실패 시 일부만 롤백 가능

### 6.6 증분 소스 선택 규칙
- Common Crawl: 이전에 처리하지 않은 신규 crawl ID만 대상
- Wiktionary/Kaikki/Wikipedia dump: 최신 dump가 이전 버전보다 새로울 때만 대상
- 워드리스트 저장소: 체크섬/버전이 달라졌을 때만 대상
- 사용자 추가 크롤링 대상: sitemap/etag/last-modified 변화가 있을 때만 대상

---

## 7. 저장 구조 설계

### 7.1 저장 방식 선정
로컬 PC에서 수억 개 이상의 단어를 저장하고 AI가 꺼내 쓰기 쉽게 하려면, 단일 txt 파일만으로는 비효율적이다. 다음의 혼합 구조를 채택한다.

#### A. 원본/중간/최종 분리
- raw: 다운로드 원본
- stage: 파싱/토큰화 중간 결과
- warehouse: 최종 누적 단어 저장소

#### B. 최종 저장 형식
1. **Parquet shard**
   - 용도: 대량 저장, 분석, 압축, 배치 처리
   - 이유: 컬럼형 저장이라 메타데이터 조회와 집계가 빠름

2. **LMDB/RocksDB exact registry**
   - 용도: 고속 중복 제거
   - 이유: 수억 key를 다뤄도 비교적 현실적

3. **DuckDB metadata catalog**
   - 용도: 통계/샘플/검색/리포트
   - 이유: 로컬 분석성이 매우 좋음

### 7.2 권장 스키마

#### token warehouse parquet columns
- `token_raw` : 원본 문자열
- `token_canonical` : 비교용 canonical string
- `token_hash` : 64/128bit hash
- `source_id` : 최초 발견 소스 ID
- `source_type` : crawl/dump/api/list
- `first_seen_at` : 최초 발견 시각
- `run_id` : 어떤 실행에서 들어왔는지
- `token_len` : 길이
- `char_class` : alpha/alnum/hyphen/apostrophe/mixed
- `quality_flags` : url_like, code_like, weird_unicode 등
- `tier` : lexicon / web / entity / noisy

#### catalog tables
- `runs`
- `sources`
- `source_files`
- `run_stats`
- `quality_reports`
- `shard_manifest`

### 7.3 AI가 꺼내 쓰기 좋은 방식
AI 활용을 고려하면 최종 export를 3종으로 만든다.

1. **canonical_wordlist shards**
   - prefix 기준 txt/zst 분할
   - 예: `a/aa.txt.zst`, `a/ab.txt.zst`
   - 장점: 빠른 exact lookup

2. **parquet analytics store**
   - 대량 질의, 샘플링, 품질 분석용

3. **deduped flat export**
   - `all_words_deduped.txt.zst`
   - 장점: LLM/RAG 전처리 입력으로 쉬움

### 7.4 추천 결론
**주 저장소는 Parquet + LMDB(RocksDB) + DuckDB 조합**으로 하고, 배포/활용용으로 압축 txt shard를 별도 export 한다.

---

## 8. 토큰화 및 정규화 정책

### 8.1 기본 정책
수집량 최대화가 목적이므로, 보수적 필터링이 아니라 **넓게 잡고 후속 티어링**하는 정책을 쓴다.

### 8.2 토큰 추출 규칙 예시
- 알파벳만: `hello`
- 알파벳+숫자 혼합: `h264`, `mp3`, `i18n`
- 아포스트로피 포함: `don't`, `rock'n'roll`
- 하이픈 포함: `state-of-the-art`
- 대문자 유지형: `NASA`, `OpenAI`
- 점 포함 약어는 별도 규칙으로 보존 여부 선택: `u.s.a.`

### 8.3 정규화 레벨
#### level 0: raw preserve
- 원문 그대로 저장

#### level 1: compare normalization
- Unicode NFKC
- 앞뒤 공백 제거
- 제어문자 제거
- 연속 내부 공백 collapse(선택)

#### level 2: canonical compare key
- casefold key 별도 생성
- 특정 문장부호 정리 key 별도 생성(옵션)

### 8.4 중복 기준
- 원칙: `token_canonical` 기준 dedupe
- 단, raw variant도 별도 저장 가능
- 예
  - `OpenAI`와 `openai`는 canonical 동일 처리 가능
  - `co-operate`와 `cooperate`는 동일 처리 여부를 정책으로 분리

### 8.5 품질 티어링
모든 것을 수집하되, 저장소 안에서 티어를 나눈다.
- `tier_1_lexicon`: 사전/고품질 소스
- `tier_2_entity`: 위키 타이틀/고유명사
- `tier_3_web_clean`: 웹에서 자주 등장하는 일반 토큰
- `tier_4_web_longtail`: 드문 토큰
- `tier_5_noisy`: 노이즈 가능성 높음

이 구조를 쓰면 수집량은 최대화하면서도, 나중에 AI가 필요에 따라 특정 티어만 선택해 사용할 수 있다.

---

## 9. 구현 스펙 (구조 개요만)

## 9.1 폴더 구조

```text
/project-root
 ├── CLAUDE.md
 ├── /.claude
 │   ├── /skills
 │   │   ├── /source-inventory-manager
 │   │   │   ├── SKILL.md
 │   │   │   ├── /scripts
 │   │   │   └── /references
 │   │   ├── /dump-fetcher
 │   │   │   ├── SKILL.md
 │   │   │   ├── /scripts
 │   │   │   └── /references
 │   │   ├── /wet-warc-parser
 │   │   │   ├── SKILL.md
 │   │   │   ├── /scripts
 │   │   │   └── /references
 │   │   ├── /token-extractor
 │   │   │   ├── SKILL.md
 │   │   │   ├── /scripts
 │   │   │   └── /references
 │   │   ├── /normalizer-deduper
 │   │   │   ├── SKILL.md
 │   │   │   ├── /scripts
 │   │   │   └── /references
 │   │   ├── /warehouse-writer
 │   │   │   ├── SKILL.md
 │   │   │   ├── /scripts
 │   │   │   └── /references
 │   │   └── /quality-audit
 │   │       ├── SKILL.md
 │   │       ├── /scripts
 │   │       └── /references
 │   └── /agents
 │       ├── /source-strategy-planner
 │       │   └── AGENT.md
 │       ├── /incremental-harvest-planner
 │       │   └── AGENT.md
 │       ├── /parser-policy-agent
 │       │   └── AGENT.md
 │       ├── /token-policy-agent
 │       │   └── AGENT.md
 │       ├── /normalization-reviewer
 │       │   └── AGENT.md
 │       ├── /dedupe-auditor
 │       │   └── AGENT.md
 │       ├── /storage-strategy-agent
 │       │   └── AGENT.md
 │       ├── /quality-auditor
 │       │   └── AGENT.md
 │       └── /report-generator
 │           └── AGENT.md
 ├── /config
 │   ├── source_registry.yaml
 │   ├── normalization_rules.yaml
 │   ├── tokenization_rules.yaml
 │   └── runtime.yaml
 ├── /data
 │   ├── /raw
 │   ├── /stage
 │   ├── /warehouse
 │   ├── /registry
 │   └── /catalog
 ├── /output
 │   ├── /reports
 │   ├── /exports
 │   └── /samples
 └── /docs
     ├── source_catalog.md
     ├── token_policy.md
     ├── storage_policy.md
     └── operations_runbook.md
```

## 9.2 CLAUDE.md 핵심 섹션 목록
- 프로젝트 목적 및 범위
- 절대 원칙(수집량 우선, 증분 누적, 중복 금지)
- 다중 AI 합의 정책
- 판단과 코드의 역할 분리
- 소스 우선순위 정책
- 증분 수집 정책
- 토큰화/정규화 정책
- dedupe 정책
- 저장 정책
- 품질 티어 정책
- 실패 처리 정책
- 리포트 정책

## 9.3 에이전트 구조
기본 권장안은 **서브에이전트 분리 구조**다.

### 이유
- 소스 전략, 토큰 정책, 정규화 리뷰, dedupe 감사, 품질 분석이 서로 다른 판단 업무다.
- Common Crawl / Wiktionary / 워드리스트 처리 지침을 항상 메인에 모두 넣으면 컨텍스트가 비대해진다.
- 따라서 메인 에이전트가 orchestrator 역할을 하고, 정책 판단이 필요한 시점에만 서브에이전트를 호출한다.

## 9.4 서브에이전트 정의

### source-strategy-planner
- 역할: 신규 소스 채택 및 우선순위 정책
- 입력: source registry, harvest history, disk budget
- 출력: 이번 실행 대상 소스 리스트
- 데이터 전달 방식: 파일 기반

### incremental-harvest-planner
- 역할: 이전 실행 대비 증분 대상만 선택
- 입력: runs metadata, source versions
- 출력: incremental harvest plan
- 데이터 전달 방식: 파일 기반

### parser-policy-agent
- 역할: 포맷 이상치 대응 정책 판단
- 입력: parser error logs, sample records
- 출력: parser rule decision memo
- 데이터 전달 방식: 프롬프트 인라인 + 로그 파일

### token-policy-agent
- 역할: 토큰 경계/유지 규칙 개선
- 입력: 샘플 텍스트, 품질 리포트
- 출력: rule change proposal
- 데이터 전달 방식: 파일 기반

### normalization-reviewer
- 역할: 정규화 과병합 여부 판단
- 입력: collision samples
- 출력: normalization adjustment proposal
- 데이터 전달 방식: 파일 기반

### dedupe-auditor
- 역할: false positive / registry integrity 점검
- 입력: dedupe sample logs
- 출력: audit result
- 데이터 전달 방식: 파일 기반

### storage-strategy-agent
- 역할: shard 재배치 및 압축 정책 판단
- 입력: shard manifest, size report
- 출력: compaction/rebalance plan
- 데이터 전달 방식: 파일 기반

### quality-auditor
- 역할: 노이즈 유형 정성 분석
- 입력: sampled new tokens
- 출력: quality note, policy proposal
- 데이터 전달 방식: 파일 기반

### report-generator
- 역할: 사람이 보기 쉬운 실행 요약 작성
- 입력: run stats, failures, source coverage
- 출력: markdown report
- 데이터 전달 방식: 파일 기반

### download-integrity-reviewer
- 역할: 다운로드/파일 무결성 이상 징후 검토
- 입력: checksum 결과, file size, 샘플 open 결과, retry logs
- 출력: integrity review note
- 데이터 전달 방식: 파일 기반

### consensus-reviewer-a
- 역할: 각 단계의 1차 독립 더블체크
- 입력: 단계 산출물, 로그, 규칙 문서
- 출력: approve/revise 의견과 근거
- 데이터 전달 방식: 파일 기반

### consensus-reviewer-b
- 역할: 각 단계의 2차 독립 더블체크
- 입력: 단계 산출물, 로그, 규칙 문서
- 출력: approve/revise 의견과 근거
- 데이터 전달 방식: 파일 기반

### consensus-tiebreaker
- 역할: reviewer 간 충돌 시 최종 판정
- 입력: primary 판단안, reviewer-a 의견, reviewer-b 의견, 산출물 샘플
- 출력: final decision memo
- 데이터 전달 방식: 파일 기반


## 9.5 작업 단계별 처리 방식
- 소스 선택: 에이전트 판단 + 다중 검토 + 스크립트 실행
- 다운로드: 스크립트 + 무결성 검토 에이전트 + 다중 검토
- 파싱: 스크립트 중심, 이상 시 에이전트 판단 + 다중 검토
- 토큰 추출: 스크립트 중심, 규칙 개선은 에이전트 + 다중 검토
- dedupe: 스크립트 + 이상 징후 다중 검토
- 저장: 스크립트 + 저장 정책 다중 검토
- 품질 분석: 스크립트 + 에이전트 + 다중 검토
- 리포트: 스크립트 + 에이전트 + 다중 검토

## 9.6 스킬/스크립트 파일 목록, 역할, 트리거 조건

### source-inventory-manager
- 역할: 소스 레지스트리 관리
- 트리거 조건: 실행 시작 시

### dump-fetcher
- 역할: dump/API/원격 파일 다운로드
- 트리거 조건: incremental plan 생성 후

### wet-warc-parser
- 역할: Common Crawl WET/WARC 스트리밍 파싱
- 트리거 조건: Common Crawl 소스 처리 시

### token-extractor
- 역할: 텍스트에서 후보 토큰 추출
- 트리거 조건: 파싱 결과 생성 후

### normalizer-deduper
- 역할: 정규화, 해시 생성, 중복 제거
- 트리거 조건: 후보 토큰 배치 생성 후

### warehouse-writer
- 역할: shard 저장, catalog update, export update
- 트리거 조건: 신규 단어 배치 확정 후

### quality-audit
- 역할: 샘플 품질 분석 및 리포트
- 트리거 조건: run 종료 직전

## 9.7 주요 산출물 파일 형식
- `.yaml` : 설정
- `.jsonl` : 중간 스트리밍 레코드
- `.parquet` : 최종 단어 저장 샤드
- `.mdb` 또는 KV store 파일 : exact registry
- `.duckdb` : 메타데이터 카탈로그
- `.md` : 리포트
- `.txt.zst` : AI/배포용 deduped word export

---

## 10. 검증 패턴

### 단계별 검증 방식 요약
| 단계 | 성공 기준 | 검증 방법 | 실패 시 처리 |
|---|---|---|---|
| 소스 인벤토리 로드 | 필수 필드 완비 + 다중 에이전트 승인 | 스키마 검증 + 합의 검토 | 자동 재시도 1회 후 에스컬레이션 |
| 증분 대상 판정 | 중복 대상 재선정 없음 + 다중 에이전트 승인 | 규칙 기반 + 합의 검토 | 자동 재시도 1회 |
| 다운로드 | 파일 손상 없음 + 다중 에이전트 승인 | 체크섬/크기 검증 + 샘플 오픈 + 합의 검토 | 자동 재시도 최대 3회 |
| 파싱 | 샘플 레코드 정상 추출 + 다중 에이전트 승인 | 규칙 기반 + 합의 검토 | 자동 재시도 2회 |
| 토큰 추출 | 후보 토큰 생성량/샘플 정상 + 다중 에이전트 승인 | 규칙 기반 + 샘플 검수 + 합의 검토 | 자동 재시도 1회 |
| 정규화 | canonical key 일관성 유지 + 다중 에이전트 승인 | 규칙 기반 + 합의 검토 | 자동 재시도 1회 |
| dedupe | 기존 단어 재삽입 없음 + 다중 에이전트 승인 | 재주입 테스트 + 합의 검토 | 자동 재시도 1회 |
| 저장 | 카운트/메타데이터 일치 + 다중 에이전트 승인 | 스키마 + 규칙 기반 + 합의 검토 | 롤백 후 에스컬레이션 |
| 품질 분석 | 리포트 생성 + 다중 에이전트 승인 | 규칙 기반 + LLM 자기 검증 + 합의 검토 | 스킵 + 로그 |
| 리포트 작성 | 다음 실행에 필요한 marker 기록 + 다중 에이전트 승인 | 스키마 검증 + 합의 검토 | 자동 재시도 1회 |

---

## 11. 실패 처리 정책

### 11.1 자동 재시도
사용 시점
- 네트워크 실패
- 일시적 파일 잠금
- gzip/bz2 해제 실패(재다운로드로 복구 가능)
- 임시 DB write 실패

### 11.2 에스컬레이션
사용 시점
- 신규 포맷 출현
- 정규화 규칙이 과도한 병합을 유발
- exact registry 손상 의심
- 디스크 용량 부족
- reviewer 간 합의 실패 또는 tie-breaker 판정 불가

### 11.3 스킵 + 로그
사용 시점
- 선택적 품질 분석 단계 실패
- 비핵심 통계 생성 실패
- 일부 보조 export 실패

---

## 12. 로컬 PC 운영 설계

### 12.1 권장 환경
- Python 기반 구현
- 스트리밍 처리 필수
- 멀티프로세스/배치 처리 사용
- 저장소는 SSD 권장

### 12.2 디스크 운영 원칙
- raw와 warehouse를 분리
- 다운로드 원본은 보관 기간 정책 적용
- stage 결과는 재생성 가능하면 주기적으로 정리
- shard compaction 주기 운영

### 12.3 실행 모드
- `seed-build` : 사전/워드리스트 기반 초기 구축
- `crawl-expand` : Common Crawl 기반 대규모 확장
- `incremental-update` : 신규 소스만 누적 추가
- `reindex-export` : export만 재구성
- `quality-audit` : 샘플 품질 분석 전용

---

## 13. 권장 기술 스택

### 13.1 스크립트/파이프라인
- Python
- requests/httpx
- warcio 또는 유사 WARC parser
- lxml / streaming XML parser
- regex / custom tokenizer
- DuckDB
- PyArrow / Parquet
- LMDB 또는 RocksDB 바인딩
- zstandard

### 13.2 선택 이유
- Python은 크롤링/압축/텍스트/DB 생태계가 넓다.
- DuckDB + Parquet는 로컬 대용량 분석에 유리하다.
- LMDB/RocksDB는 exact dedupe registry에 적합하다.
- zstd는 대용량 텍스트 export 압축에 유리하다.

---

## 14. 실행 로드맵

### Phase 1. Seed 구축
- Wiktionary
- Kaikki
- SCOWL
- WordNet
- Wikipedia titles
- 목표: 고품질/엔티티 기초층 확보

### Phase 2. 대규모 확장
- Common Crawl 월별 크롤 처리
- 목표: 고유 단어 수를 폭발적으로 증가

### Phase 3. 증분 자동화
- 매 실행마다 신규 crawl/dump만 처리
- exact registry 기반 누적 저장

### Phase 4. 품질 티어링 고도화
- 노이즈 패턴 분리
- 티어별 export 생성

---

## 15. 최종 권고안

이 프로젝트는 “영어사전 만들기”가 아니라 “초대규모 영어 문자열 수집 인프라 만들기”로 정의하는 것이 맞다.

가장 적절한 최종 구조는 아래와 같다.

1. **수집 소스**
   - Seed: Wiktionary / Kaikki / SCOWL / WordNet / Wikipedia titles
   - Scale: Common Crawl
   - Expansion: 기타 공개 워드리스트 및 도메인 특화 웹

2. **중복 제거 구조**
   - Bloom/Cuckoo filter + LMDB/RocksDB exact registry

3. **저장 구조**
   - Parquet shard + DuckDB catalog + txt.zst export

4. **운영 구조**
   - 매 실행 시 incremental harvest plan 생성
   - 기존에 없는 단어만 누적 추가

5. **에이전트 구조**
   - 메인 오케스트레이터 + 정책 판단용 서브에이전트
   - 실제 대용량 처리 대부분은 스크립트 수행

이 구조가 로컬 PC 기준에서 가장 현실적이면서도, 수억 개 이상의 고유 영어 단어/표기 후보를 장기적으로 누적하는 데 가장 적합하다.

---

## 16. 참고 소스 주소 목록

- Common Crawl 메인: `https://commoncrawl.org/`
- Common Crawl 시작 가이드: `https://commoncrawl.org/get-started`
- Common Crawl 데이터 루트: `https://data.commoncrawl.org/`
- English Wiktionary latest dump: `https://dumps.wikimedia.org/enwiktionary/latest/`
- English Wikipedia latest dump: `https://dumps.wikimedia.org/enwiki/latest/`
- Kaikki raw data: `https://kaikki.org/dictionary/rawdata.html`
- Kaikki English dictionary: `https://kaikki.org/dictionary/English/index.html`
- SCOWL / English Speller Database: `https://wordlist.aspell.net/`
- SCOWL dictionaries: `https://wordlist.aspell.net/dicts/`
- WordNet: `https://wordnet.princeton.edu/homepage`


---
name: navis-exam
description: |
  BC카드 교육시스템 NAVIS(bccard.hunet.co.kr) 자동 시험 응시 스킬.
  claude-in-chrome MCP 도구를 사용하여 교육과정 시험을 자동으로 응시하고 문제를 풀어 제출한다.
  트리거: "나비스 시험", "NAVIS 시험", "교육 시험 응시", "hunet 시험", "시험 응시해줘",
  "교육과정 시험", "나비스 응시", "BC카드 교육", "진행중 교육 시험"
---

# NAVIS 자동 시험 응시

BC카드 교육시스템(NAVIS/Hunet) 시험 자동 응시 브라우저 자동화 스킬.

## 사전 조건

- `claude-in-chrome` MCP 도구 활성화 필수
- Chrome/Edge 브라우저에 Claude in Chrome 확장 프로그램 설치 필수

## 핵심 성능 원칙

1. **JS 호출 통합**: alert/confirm override + 비즈니스 로직을 항상 단일 javascript_tool 호출로 합친다
2. **페이지 이동 후 첫 JS에 항상 alert override 포함**: 페이지 이동 시 JS 컨텍스트가 초기화되므로 매번 재설정
3. **find() 우선**: read_page(filter:"all") 대신 find()로 타겟 요소만 조회 (출력 최소화)

## 실행 Flow

### Step 0: 도구 일괄 로드

**반드시 첫 턴에 4개 ToolSearch를 병렬 호출**하여 모든 도구를 선로드한다:

```
// 단일 메시지에서 4개 병렬 호출
ToolSearch("select:mcp__claude-in-chrome__tabs_context_mcp,mcp__claude-in-chrome__tabs_create_mcp,mcp__claude-in-chrome__navigate")
ToolSearch("select:mcp__claude-in-chrome__javascript_tool,mcp__claude-in-chrome__computer,mcp__claude-in-chrome__find")
ToolSearch("select:mcp__claude-in-chrome__get_page_text,mcp__claude-in-chrome__read_page")
ToolSearch("select:WebSearch")
```

이후 ToolSearch 호출 불필요.

### Step 1: 브라우저 초기화 + 로그인 확인

Home 페이지를 **경유하지 않고** 바로 진행중 교육 페이지로 이동한다.

```
tabs_context_mcp(createIfEmpty: true)  → tabId 획득
navigate(tabId, "https://bccard.hunet.co.kr/Classroom/StudyIng?onnoffType=on")
```

페이지 로드 후 **단일 JS 호출**로 alert override + 로그인 확인:

```javascript
javascript_tool(tabId, `
  window.alert = function(msg) { console.log('[ALERT]', msg); };
  window.confirm = function(msg) { console.log('[CONFIRM]', msg); return true; };
  document.querySelector('a[href*="Logout"]') ? 'logged_in' : 'not_logged_in'
`)
```

- `logged_in` → Step 2 진행
- `not_logged_in` → 사용자에게 직접 로그인 요청 후 대기

### Step 2: 과정 스캔

`find(tabId, "응시하기")`로 응시 가능한 시험 탐색.

- 결과 없음 → "응시 가능한 시험이 없습니다." 보고 후 종료
- 결과 있음 → `get_page_text(tabId)`로 과정명/교육일정 추출 → 사용자에게 보고 후 진행 여부 확인

### Step 3: 팝업 인터셉트 및 시험 페이지 진입

**중요**: "응시하기" 버튼은 `window.open`으로 팝업을 생성한다.
팝업은 MCP 탭 그룹에 포함되지 않으므로 반드시 인터셉트해야 한다.

**단일 JS 호출**로 모든 준비를 완료:

```javascript
javascript_tool(tabId, `
  window.alert = function(msg) { console.log('[ALERT]', msg); };
  window.confirm = function(msg) { console.log('[CONFIRM]', msg); return true; };
  window._capturedPopupUrl = null;
  window.open = function(url, name, features) {
    window._capturedPopupUrl = url;
    const a = document.createElement('a');
    a.href = url;
    a.target = '_blank';
    a.id = 'navis_popup_link';
    a.textContent = 'Open Exam';
    a.style.cssText = 'position:fixed;top:10px;left:10px;z-index:99999;padding:20px;background:red;color:white;font-size:20px;cursor:pointer;';
    document.body.appendChild(a);
    return { closed: false, close(){}, focus(){}, document: document.createElement('div') };
  };
  'ready'
`)
```

그 다음 순서대로:

```
computer(left_click, ref: 응시하기_버튼_ref)       // 팝업 인터셉트 발동
find(tabId, "Open Exam") → ref 획득
computer(left_click, ref: Open_Exam_ref, modifiers: "ctrl")  // 새 MCP 탭으로 열기
```

새 탭이 생성되면 `examTabId`로 저장. 이후 모든 작업은 이 탭에서 진행.

### Step 4: 시험 진입 (3단계)

시험 페이지는 `smartlearning.hunet.co.kr`로 리다이렉트된다.

**4-1. Exam/Index.aspx** - "확인 후 응시하기" 페이지

```
wait(3)  // 리다이렉트 대기
javascript_tool(examTabId, `
  window.alert = function(msg) { console.log('[ALERT]', msg); };
  window.confirm = function(msg) { console.log('[CONFIRM]', msg); return true; };
  'alert overridden'
`)
find(examTabId, "확인 후 응시하기") → ref
computer(left_click, ref)
```

**4-2. Exam/View_new.aspx** - 시험 안내 페이지

```
wait(2)  // 페이지 이동 대기
javascript_tool(examTabId, `
  window.alert = function(msg) { console.log('[ALERT]', msg); };
  window.confirm = function(msg) { console.log('[CONFIRM]', msg); return true; };
  'alert overridden'
`)
find(examTabId, "응시하기 button") → ref
computer(left_click, ref)  // fn_showExamModel() → 모달 표시
```

**4-3. 모달 확인**

"지금 시험에 응시하시겠습니까?" 모달 표시 후:

```
wait(1)  // 모달 렌더링 대기
read_page(examTabId, filter: "interactive") → 모달 내 "확인" 버튼 ref 식별
computer(left_click, 확인_ref)
```

→ `Exam/Exam.aspx`로 이동 (실제 문제 풀이 화면)

### Step 5: 문제 풀이

**페이지 구조 (Exam/Exam.aspx)**:
- 좌측: 답안작성현황 (문제번호 1~N 링크)
- 상단: 남은시간 카운트다운
- 중앙: 문제 텍스트 + 보기 (radio 버튼)
- 하단: "다음문제 >" 버튼, "미리보기 후 답안제출" 버튼

**문제 유형**:
- **객관식 (4지선다)**: radio 버튼 4개 (value 1~4)
- **OX**: radio 버튼 2개 (O, X)

**풀이 전략: 확신도 기반 WebSearch 검증**

모든 문제에 WebSearch를 사용하면 시간이 초과될 수 있으므로, 확신도 기반 분기 처리를 적용한다:

| 확신도 | 행동 | 예시 |
|--------|------|------|
| **높음** (명확한 사실/상식) | LLM 판단으로 바로 답 선택 | "HTTP 상태코드 200은?", "SQL Injection이란?" |
| **보통** (도메인 특화/최신 정보) | WebSearch로 검증 후 답 선택 | "개인정보보호법 시행령 제X조", "2024년 금융 규제" |
| **낮음** (사내 정책/모르는 주제) | WebSearch 필수, 다중 검색 | "BC카드 내부 정책", "회사 특화 프로세스" |

**풀이 루프** (문제당 최소 호출):

```
for 각 문제 (1 ~ N):
  1. get_page_text(examTabId) → 문제 텍스트 + 보기 추출

  2. [1차 판단] LLM이 문제 분석:
     - 정답 후보 선정
     - 확신도 판정 (높음/보통/낮음)
     - 검색 키워드 도출

  3. [WebSearch 검증] 확신도가 "보통" 또는 "낮음"인 경우:
     WebSearch("문제 핵심 키워드 + 보기 핵심어")
     → 검색 결과에서 근거 추출
     → 1차 판단과 비교하여 최종 정답 확정
     * 검색 결과와 1차 판단이 상충하면 → 검색 근거 우선
     * 검색 결과가 불충분하면 → 키워드 변경하여 1회 재검색
     * 시간 제약: 문제당 WebSearch 최대 2회로 제한

  4. find(examTabId, "정답 보기 텍스트") → 정답 radio ref 획득
  5. computer(left_click, 정답_radio_ref)
  6. find(examTabId, "다음문제") → computer(left_click, ref)
     (마지막 문제면 스킵)
```

**WebSearch 쿼리 작성 가이드**:
- 문제 전체를 검색하지 않고, **핵심 키워드 2~4개**로 축약
- 보기 중 헷갈리는 선택지를 포함하여 검색
- 예: 문제 "개인정보 처리방침에 반드시 포함되어야 하는 항목은?" 
  → 검색: `"개인정보 처리방침" 필수 포함 항목`
- 법률/규제 문제: 법령명 + 조항 키워드로 검색
- OX 문제: 핵심 명제를 그대로 검색하여 사실 여부 확인

### Step 6: 답안 제출

```
javascript_tool(examTabId, `
  window.alert = function(msg) { console.log('[ALERT]', msg); };
  window.confirm = function(msg) { console.log('[CONFIRM]', msg); return true; };
  'alert overridden'
`)
find(examTabId, "미리보기 후 답안제출") → ref
computer(left_click, ref)
// 제출 확인 모달이 뜨면 "확인" 클릭
```

### Step 7: 결과 보고

사용자에게 보고:
- 과정명
- 응시한 문항 수
- 선택한 답안 요약 (문제별: 선택 답, 확신도, WebSearch 사용 여부)
- WebSearch 활용 통계 (검색 문항 수 / 전체 문항 수)
- 제출 완료 여부

## 주의사항

- **alert/confirm 차단**: 매 페이지 이동 후 첫 JS 호출에 반드시 override 포함 (페이지 이동 시 초기화됨)
- **팝업 인터셉트**: window.open override → 자동 `<a>` 태그 주입 → Ctrl+클릭 패턴 사용 (단일 JS로 통합)
- **cross-origin**: 메인(`bccard.hunet.co.kr`)과 시험(`smartlearning.hunet.co.kr`)은 다른 도메인
- **타이머**: 시험 제한 시간 (보통 50분) 있으므로 빠르게 진행
- **임시저장**: 문제 풀이 중 `link "임시저장"` 버튼으로 중간 저장 가능
- **재응시**: 이미 응시한 시험은 "재응시" 모달이 뜰 수 있음 (ref가 다를 수 있음)

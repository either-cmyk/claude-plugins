---
name: n-value-calculator
description: CN인사이더 발주번호로부터 1위안당 원화(N값)를 계산하여 수입원가 구글시트의 지정된 행에 자동 입력하는 스킬. CN인사이더 발주 상세에서 위안화 비용을 수집하고, Gmail에서 팝빌 청구서/정산서 또는 CN인사이더 직접 발송 청구서 PDF를 추출하여 환율·해상운임·관세·부가세를 산출, N = M/C 공식으로 1위안당 통관 완료 원화를 계산한다. 공식 M = (C+D)×P + SUM(E:L), N = M/C, K(1% 수수료) = C × 0.01 × P. C = 상품+중중배송비+부가서비스(CNY) = 총금액 − 수수료, D = 수수료(CNY). 환율 P는 청구서의 '구매대행수수료 1%' 행 환율. F(해상운임) = 96,000원/CBM × 발주 CBM. G(DOC+원산지+통관, VAT 포함)는 BL이 발주 전용이면 전액, 공유면 발주 CBM/BL 총 CBM 비율. 정산서가 한 줄 합산이면 I/J를 출고 PC 비율로 분배. 베개+커버 세트류는 정산서 PC가 세트의 2배일 수 있으나 발주 전체에 해당. 반드시 이 스킬을 사용해야 하는 경우 '1위안당금액 계산해줘', '위안당원화 계산해줘', 'N값 계산', '원가 시트 입력', '수입원가 행 입력', '발주번호 N값', '청구서 정산서 계산', '원가 행 입력', '이 발주 원가', 'S-QEC 원가 계산해줘 X행에 넣어줘', '발주번호 줄게 N값 구해줘', '1위안당원화 계산', '베개 1위안당' 등 발주번호로 1위안당 금액을 산출하고 원가 시트 특정 행에 입력하는 모든 요청. 사용자가 발주번호(S-QEC로 시작)와 '1위안당' 또는 'N값' 키워드를 함께 언급하면 이 스킬을 반드시 사용할 것.
---

# n-value-calculator 스킬

## 개요
CN인사이더 발주의 **1위안당 원화(N값)** 를 계산하여 수입원가 구글시트의 지정 행에 자동 입력.

**주 트리거: `1위안당금액 계산해줘`, `위안당원화 계산해줘`**

---

## 🔴 Step 0: 사용자 정보 확인 (세션당 1회)

### Case A: 세션에서 처음 호출되는 경우
`AskUserQuestion` 도구로 다음 3가지를 한 번에 물어본다:

**질문 1: 사업자**
- 이든코퍼레이션 (Recommended for 첫 사용자)
- 이더컴퍼니
- 클린인테크
- 마인플로

**질문 2: Gmail 계정** (청구서/팝빌 확인용)
- hg.kim@either.co.kr (이더컴퍼니)
- hh.kim@either.co.kr (이든코퍼레이션)
- dh.choi@either.co.kr (마인플로·클린인테크)
- Other (직접 입력)

> ⚠️ 구 계정 `kimhh0522@gmail.com`, `njcommerce2@gmail.com`은 더 이상 사용하지 않음. 청구서/팝빌은 위 either.co.kr 계정에서만 확인.

**질문 3: 팝빌 인증용 사업자번호** (하이픈 제외 10자리로 입력)
- 401-88-02816 (이더컴퍼니)
- 578-85-02944 (이든코퍼레이션)
- 772-85-02468 (클린인테크)
- 186-49-01262 (마인플로, 간이과세자)
- Other (직접 입력)

받은 정보를 세션 메모리에 기록하고 모든 후속 작업에 사용:
```
✅ 사업자: [받은값]
✅ Gmail: [받은값]
✅ 사업자번호: [받은값]
✅ 시트 ID: [매핑 또는 사용자 입력]
```

### Case B: 이미 받은 정보가 있는 경우
재질문 없이 한 줄로 확인만:
> 📌 [사업자명] 시트에 [Gmail]로 진행합니다.

사용자가 "다른 사업자", "이번엔 ○○사로" 등 명시적 변경 요청 시에만 다시 묻는다.

### 사업자별 매핑

모든 사업자가 **같은 워크북**(시트 ID `1X7UsQ5WFZmT1XiuVJwFVjZlb_mxb9AqNBaQIUkz9n44`)의 각 탭을 사용한다.

| 사업자 | 청구서/팝빌 Gmail | 사업자번호 | 시트 탭 (gid) |
|--------|-------|----------|---------|
| 이더컴퍼니 | hg.kim@either.co.kr | 401-88-02816 | 이더 수입원가계산 (gid 1761153393) |
| 이든코퍼레이션 | hh.kim@either.co.kr | 578-85-02944 | 이든 수입원가계산 |
| 클린인테크 | dh.choi@either.co.kr | 772-85-02468 | 클린인 수입원가계산 |
| 마인플로 | dh.choi@either.co.kr | 186-49-01262 | 마인플로 수입원가계산 |

> 시트 ID는 4개사 공통. 탭만 사업자별로 다름. 이더 외 탭의 gid는 첫 작업 시 확인해 채울 것.

### 행 번호도 확인
- "다음 빈 행 자동" (Recommended) — gviz CSV로 마지막 데이터 행 + 1
- "행 번호 직접 지정" (사용자 입력)

---

## 🎯 핵심 공식

```
M (총비용 KRW)   = (C + D) × P + SUM(E:L)
N (1위안당 원화) = M ÷ C
K (1% 수수료)    = C × 0.01 × P    ← 수식으로 입력
```

- **C** = 상품 위안화 + 중중배송비 + 부가서비스 합 (CNY) **= 총금액 − 수수료**
- **D** = 수수료 (CNY) — 대행수수료 7% (검증: D ÷ 0.07 ≒ 상품+물류)
- **P** = 환율 (CNY → KRW) — **청구서의 '구매대행수수료 1%' 행 환율**
- **E~L** = 모든 KRW 비용 합 (E·L은 보통 빈칸)

⚠️ **P(환율) 누락 시 N값이 1/3 수준으로 잘못 계산됨. 반드시 입력 확인!**

---

## 📋 시트 컬럼 구조 (⚠️ G:H 셀 병합)

| 셀 | 항목 | 통화 | 출처 / 공식 |
|---|---|---|---|
| A | 발주번호 | - | S-QEC... |
| B | 품목 | - | `="이름"` (한글 IME 첫 글자 누락 방지) |
| C | 상품 + 중중배송비 + 부가서비스 | CNY | CN인사이더 발주 상세 |
| D | 수수료 | CNY | CN인사이더 발주 상세 (7%) |
| E | 현지반품관련비용 | KRW | 보통 빈칸 |
| F | 중한배송비 (해상운임) | KRW | **96,000원/CBM × 발주 CBM** |
| **G:H (병합)** | DOF + 원산지 + 통관수수료 | KRW | (DOC+VAT) + 원산지 + (통관+VAT). BL 공유 시 ×(발주CBM/BL총CBM) |
| I | 관세 | KRW | 정산서 세액(관) 또는 청구서 '관세' 행 |
| J | 부가세 | KRW | 정산서 세액(부) 또는 청구서 '부가세' 행 |
| **K** | **1% 수수료** | KRW | **`=C*0.01*P`** ← 수식 |
| L | 한한배송비 | KRW | 보통 빈칸 |
| M | 총비용 | KRW | `=(C+D)*P+SUM(E:L)` |
| N | **1위안당 금액 ★** | KRW/CNY | `=M/C` |
| O | CBM | - | 발주 CBM |
| **P** | **환율** ⚠️ | - | 청구서의 '구매대행수수료 1%' 행 (절대 누락 금지) |
| Q | 박스당 입수량 | PC/세트 | CN인사이더 CN출고 |

⚠️ **G:H는 셀 병합**이라 Tab 시 H를 건너뜀. G에만 값 입력.

---

## 🔄 작업 흐름 (Step 0 완료 후)

### Step 1: CN인사이더 발주 상세 (C, D, 품목명)
1. `https://www.cninsider.co.kr/mall/#/orderList` 진입
2. 발주번호 검색 → "상세보기" 클릭
3. 결제정보 섹션에서 추출:
   - 상품총금액, 물류비, 부가서비스, 수수료, **총금액**
4. **C = 총금액 − 수수료** (가장 빠른 방법)
5. **D = 수수료**

검증: D ÷ 0.07 ≒ 상품 + 물류

### Step 2: CN인사이더 CN출고 (BL/CBM)
1. `https://www.cninsider.co.kr/mall/#/invoiceList` 진입
2. 발주번호로 검색 → 출고 행 클릭하여 펼침
3. **발주 CBM** 기록 (해당 출고송장의 CBM 합계)
4. 출고송장번호(S-QEC...) 기록 (청구서 매칭용)

### Step 3: 청구서 PDF 찾기 (Gmail)
**두 가지 경로**:

**A. 팝빌 전자세금계산서** (통관 완료 후 발행)
- Gmail 검색: `전자세금계산서`
- 발신: `popbill@popbill.com`
- 본문의 팝빌 링크 클릭 → 사업자번호 입력 → 첨부 PDF 확인
- 청구서 + 정산서가 모두 첨부됨

**B. CN인사이더 직접 발송 청구서** (물류비 결제용)
- Gmail 검색: `청구서`
- 발신: `cninsider@cninsider.kr`
- 제목: `(청구서)[씨엔인사이더]CNIS{BL}_S-QEC{출고송장}MS_...`
- 출고송장(S-QEC...)과 일치하는 메일을 열어 첨부 청구서 PDF 확인

⚠️ Gmail은 CSP로 pdf.js 로딩 차단 → **PDF 추출은 팝빌 페이지에서 수행**

### Step 4: 청구서 PDF에서 F, G, P 추출
청구서 운임내역 표에서:
- **P (환율)** = '구매대행 수수료 1%' 행의 환율
- **F (해상운임)** = ① 단일 '해상운임 KRW 96,000 단가' 행이 있으면 96,000 × 발주 CBM
                  ② 분해된 경우 (OCEAN FREIGHT + WHARFAGE + 창고료(+VAT) + 한국부대비용) 합 ÷ BL 총 CBM ≒ 96,000원/CBM, × 발주 CBM
- **G (DOF+원산지+통관)** = (DOC + VAT) + 원산지 + (통관 + VAT) 합산
  - BL이 우리 발주 전용 → **전액**
  - BL 공유 → × (발주 CBM / BL 총 CBM)

### Step 5: 정산서/청구서에서 I, J 추출
- **I (관세)** = 정산서 세액(관) 또는 청구서의 '관세' 행
- **J (부가세)** = 정산서 세액(부) 또는 청구서의 '부가세' 행
- 정산서 한 줄 합산(여러 발주 묶임)이면 출고 PC 비율로 분배

### Step 6: 시트 입력
- 다음 빈 행 확인 (gviz CSV로 마지막 행 + 1)
- A·B·C·D·F·G·I·J·O·P·Q 직접 입력
- K = `=C{행}*0.01*P{행}` (수식)
- M = `=(C{행}+D{행})*P{행}+SUM(E{행}:L{행})` (수식)
- N = `=M{행}/C{행}` (수식)

### Step 7: 결과 보고
```
📦 [품목명] (발주번호 [S-QEC...]) → [시트] [N]행 입력 완료

C: ¥[금액]   D: ¥[금액]   P: [환율]
F: ₩[금액] (해상운임 = 96,000 × [CBM])
G: ₩[금액] (DOC+원산지+통관)
I: ₩[금액] (관세)
J: ₩[금액] (부가세)
K: ₩[금액] (1% 수수료 = C × 0.01 × P)
M (총비용): ₩[금액]
N (1위안당): ₩[N값] ★
```

---

## ⚠️ 특수 케이스

| 케이스 | 처리 방법 |
|---|---|
| **다중품목 발주** (12개입 + 20개입 등) | C·D를 상품금액 CNY 비율로 분배 |
| **정산서 한 줄 합산** (같은 세번부호) | I·J를 출고 PC 비율로 분배 |
| **혼적 BL** (여러 발주 같은 BL) | F·G는 발주 CBM/BL 총 CBM 비율, I·J는 정산서 라인 값 |
| **BL = 우리 발주 전용** | F·G 비율 배분 없이 **전액** 적용 |
| **1+1 번들** | N 동일, 원가위안 ×2, Q는 세트 단위 |
| **발주 ≠ 출고 수량** (부분출고) | C·D는 발주 기준, I·J는 출고 PC 비율 |
| **면세 상품** (FCN1중가 0%) | I = 0 |
| **베개+커버 세트류** | 정산서 PC가 발주 세트의 2배 가능 (베개+커버 각 1PC 카운트) → 우리 발주 = 전체 declaration |
| **청구서가 관세·부가세 포함** | 정산서 없이도 I·J를 청구서에서 바로 사용 가능 (단일 BL) |
| **청구서 PDF가 Gmail 직접 발송** | 팝빌 전자세금계산서가 아직 발행 전이면 cninsider@cninsider.kr 메일에서 청구서 확인. 다만 Gmail은 PDF 추출이 막혀 정산서/세금계산서 발행 후 팝빌에서 추출 권장 |

---

## ✅ 검증 체크리스트

1. **C + D ≒ CN인사이더 발주 총금액** (위안)
2. **D ÷ 0.07 ≒ C 근사** (대행수수료 7%)
3. **P (환율) 입력 확인** ← 가장 흔한 실수, N이 1/3이면 의심
4. K 수식 = `=C*0.01*P`
5. M 수식 = `=(C+D)*P+SUM(E:L)`
6. N 수식 = `=M/C`
7. **N 정상범위**:
   - 저단가 ¥0.32 (수세미 등) → **N = 320~360**
   - 중단가 ¥5+ (캐리어 등) → **N = 260~290**
   - 부피 큰 베개·거실화류 → **N = 320~370**
   - **N < 250 또는 N > 400** → 환율(P) 입력부터 재확인

---

## 🛠️ 기술 환경 설정

### 브라우저
**항상 HOON 브라우저 사용** (deviceId `53622e5d-ddc5-4fdc-83a1-f19ebdd05d46`).
사용자가 다른 사업자로 작업하면 해당 브라우저로 전환할 것을 사용자가 명시.

### 팝빌 인증 (React Controlled Input)
사업자번호 입력란은 React onChange를 통해 트리거:

```javascript
const inp = document.querySelectorAll('input')[0];
const propsKey = Object.keys(inp).find(k => k.startsWith('__reactProps'));
const setValue = Object.getOwnPropertyDescriptor(window.HTMLInputElement.prototype, 'value').set;
setValue.call(inp, '5788502944');  // 하이픈 제외 10자리
inp.dispatchEvent(new Event('input', {bubbles: true}));
if (propsKey && inp[propsKey].onChange) inp[propsKey].onChange({target: inp, currentTarget: inp});
inp.dispatchEvent(new Event('change', {bubbles: true}));

// '확인' 버튼 클릭
const els = [...document.querySelectorAll('button, span, a, div')];
const confirmBtn = els.find(e => e.textContent.trim()==='확인' && e.children.length===0);
const pk = Object.keys(confirmBtn).find(k=>k.startsWith('__reactProps'));
if (pk && confirmBtn[pk].onClick) confirmBtn[pk].onClick({preventDefault:()=>{},stopPropagation:()=>{}});
else confirmBtn.click();
```

JS 자동 인증 실패 시 사용자에게 직접 입력 요청.

### 팝빌 PDF 추출 (auth token 캡처)
```javascript
// 1) fetch 인터셉터 설치
window.__authToken = null;
if (!window.__fetchPatched) {
  const origFetch = window.fetch;
  window.fetch = function(...args){
    const o=args[1]||{};
    if(o.headers && o.headers.Authorization) window.__authToken=o.headers.Authorization;
    return origFetch.apply(this,args);
  };
  window.__fetchPatched = true;
}

// 2) PDF 버튼 클릭 → 토큰 캡처 (약 640자)
const pdfBtns = [...document.querySelectorAll('button,a,span,div')]
  .filter(b => b.textContent.trim().toLowerCase().endsWith('.pdf') && b.children.length===0);
const btn = pdfBtns.find(b => b.textContent.includes('청구서')) || pdfBtns[0];
btn.click();
// await ~1.5s

// 3) 인덱스 2~4 fetch (실제 PDF), 결과를 window에 저장 (출력은 차단됨)
const docId = '[URL의 docId]';
for (let idx=2; idx<=4; idx++){
  const url = `https://www.popbill.com/__API_V1__/Taxinvoice/AttachedFile/${docId}/${idx}?`;
  const r = await fetch(url, {method:'GET', headers:{
    'Authorization': window.__authToken,
    'Accept-Language': 'ko',
    'X-LH-Referrer': `Taxinvoice/${docId}.E`
  }});
  const ab = await r.arrayBuffer();
  window['__p'+idx] = ab;
}
```

⚠️ Auth token 직접 출력은 차단됨 (Cookie/query string data). **window 변수에만 저장**.

### pdf.js로 PDF 텍스트 추출 (팝빌 페이지에서)
```javascript
if (!window.pdfjsLib) {
  const script = document.createElement('script');
  script.src = 'https://cdnjs.cloudflare.com/ajax/libs/pdf.js/3.11.174/pdf.min.js';
  document.head.appendChild(script);
  await new Promise(r => script.onload = r);
  window.pdfjsLib.GlobalWorkerOptions.workerSrc =
    'https://cdnjs.cloudflare.com/ajax/libs/pdf.js/3.11.174/pdf.worker.min.js';
}
const pdf = await pdfjsLib.getDocument({data: window.__p3.slice(0)}).promise;
let text = '';
for (let p=1; p<=pdf.numPages; p++) {
  const page = await pdf.getPage(p);
  const content = await page.getTextContent();
  text += content.items.map(it => it.str).join(' ') + '\n';
}
```

⚠️ **Gmail은 Trusted Types / script-src CSP로 pdf.js 로딩 차단**. 반드시 팝빌 페이지(또는 다른 비-CSP 페이지)에서 실행.

### 시트 다음 빈 행 찾기 (gviz CSV)
```javascript
const id='[시트ID]', gid='[탭gid]';
const url = `https://docs.google.com/spreadsheets/d/${id}/gviz/tq?tqx=out:csv&gid=${gid}&_=${Date.now()}`;
const r = await fetch(url, {credentials:'include', cache:'no-store'});
const txt = await r.text();
const lines = txt.split('\n');
// 마지막 S-QEC... 행 + 1 = 다음 빈 행
```

### Name Box 네비게이션
```javascript
const nb = document.querySelector('#t-name-box');
nb.focus();
const setValue = Object.getOwnPropertyDescriptor(window.HTMLInputElement.prototype, 'value').set;
setValue.call(nb, 'A67');
nb.dispatchEvent(new Event('input', {bubbles: true}));
nb.dispatchEvent(new KeyboardEvent('keydown', {key: 'Enter', bubbles: true, keyCode: 13, which: 13}));
nb.dispatchEvent(new KeyboardEvent('keyup', {key: 'Enter', bubbles: true, keyCode: 13, which: 13}));
```

### 시트 입력 (TSV 프로그래매틱 paste, 17개 값 — H 빈칸 포함)
G:H 병합 처리를 위해 H 자리에 빈 값을 넣어 17개 값으로 paste:

```javascript
const tsv = [
  'S-QEC...','="품목명"','C값','D값','','F값','G값','',   // 8번째 = H 빈칸
  'I값','J값','=C{행}*0.01*P{행}','',
  '=(C{행}+D{행})*P{행}+SUM(E{행}:L{행})','=M{행}/C{행}',
  'O값','P값','Q값또는빈칸'
].join('\t');

const dt = new DataTransfer();
dt.setData('text/plain', tsv);
document.activeElement.dispatchEvent(new ClipboardEvent('paste', {
  clipboardData: dt, bubbles: true, cancelable: true
}));
```

⚠️ H 자리 빈 값을 빠뜨리면 G:H 병합이 깨지고 모든 값이 한 칸씩 오른쪽으로 밀림. 입력 후 반드시 gviz CSV로 검증.

### 한글 IME 회피
B열 품목명 한글 입력은 `="품목명"` 수식 트릭 사용.

### G:H 병합 컬럼
- 셀 병합 상태 → G에만 값 입력. Tab 시 H 자동 건너뜀.
- Tab 순서: F → G(:H 건너뜀) → I → J → K

### 출력 차단 회피
다음은 자동 차단됨:
- Auth token, 쿠키 등 query string data 포함 응답
- Base64로 인코딩된 큰 데이터 (PDF base64 등)

→ 항상 **window 변수에 저장**, 추출한 **텍스트만 return**.

---

## 📚 작업 예시 (S-QEC2605083 EVA 거실화)

### Step 0: 사용자 정보 (세션 첫 호출)
- 사업자: 이든코퍼레이션
- Gmail: hh.kim@either.co.kr
- 사업자번호: 578-85-02944
- 시트 ID: 1X7UsQ5WFZmT1XiuVJwFVjZlb_mxb9AqNBaQIUkz9n44

### Step 1: CN인사이더 발주 상세
- 상품총금액: ¥18,175.50, 수수료: ¥1,293.29, 부가서비스: ¥0, 총금액: ¥19,768.79
- **C = 19,768.79 − 1,293.29 = 18,475.50** (= 상품 18,175.50 + 물류 300 + 부가서비스 0)
- **D = 1,293.29**
- 검증: D ÷ 0.07 = 18,475.57 ≒ C ✓

### Step 2: CN출고 (BL/CBM)
- 출고송장: S-QEC260523, **발주 CBM = 11.12**

### Step 3-4: 청구서 PDF (BL CNIS260524B04)
- **P (환율) = 220.96** (구매대행 수수료 1% 행)
- 해상운임 단가 96,000원/CBM
- **F = 96,000 × 11.12 = ₩1,067,520**
- DOC 25,000 + VAT 2,500 + 원산지 70,000 + 통관 30,000 + VAT 3,000
- **G = 130,500** (BL=우리 발주 전용 → 전액)

### Step 5: 청구서/정산서 (I, J)
- **I (관세) = 478,690**
- **J (부가세) = 627,410**
- (청구서가 관세/부가세 포함 → 정산서 없이도 가능, 정산서 교차검증 통과)

### Step 6: 시트 입력 (다음 빈 행 = 67행)
- A: S-QEC2605083 | B: ="EVA 거실화" | C: 18475.50 | D: 1293.29
- F: 1067520 | G: 130500 | I: 478690 | J: 627410
- K: =C67*0.01*P67 → ₩40,823
- M: =(C67+D67)*P67+SUM(E67:L67) → ₩6,713,055
- **N: =M67/C67 → ₩363.35 ★**
- O: 11.12 | P: 220.96

검증: N=363.35, 같은 품목군(EVA 거실화) 기존 행 N=340과 유사 범위 ✓

---

## 📝 호출 명령어

**주 트리거**: `1위안당금액 계산해줘`, `위안당원화 계산해줘`

기타 트리거:
- "발주번호 [S-QEC...] 1위안당 계산해줘 [N]번에 입력해줘"
- "N값 계산해줘"
- "[S-QEC...] 원가 계산"
- "1위안당 원화 구해줘"
- "이 발주 N값 구해줘"
- "베개 1위안당", "EVA 거실화 1위안당" 등 품목 + 1위안당 조합


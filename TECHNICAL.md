# 🌍 World Time Dashboard 기술 문서

> JavaScript Date API와 IANA 타임존을 활용한 실시간 세계 시간 구현

## 📋 목차

1. [프로젝트 개요](#프로젝트-개요)
2. [타임존 처리](#타임존-처리)
3. [서머타임(DST) 자동 감지](#서머타임dst-자동-감지)
4. [시차 계산](#시차-계산)
5. [증시 상태 계산](#증시-상태-계산)
6. [실시간 업데이트](#실시간-업데이트)
7. [UI 구현](#ui-구현)

---

## 프로젝트 개요

### 핵심 기능

```
┌──────────────────────────────────────────────────────────────┐
│                    World Time Dashboard                       │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  🇰🇷 한국 시간 (KST)                                          │
│  └── 기준 시간으로 사용                                        │
│                                                               │
│  🌎 미주  │  🇪🇺 유럽  │  🌏 아시아  │  🌊 오세아니아           │
│  ├ 뉴욕   │  ├ 런던    │  ├ 도쿄     │  ├ 시드니              │
│  ├ LA     │  ├ 파리    │  ├ 베이징   │  └ 오클랜드            │
│  ├ 댈러스  │  ├ 프랑크  │  ├ 홍콩     │                        │
│  ├ 토론토  │  ├ 마드리드 │  ├ 싱가포르 │                        │
│  └ 상파울루│  └ 밀라노  │  ├ 뭄바이   │                        │
│            │           │  └ 두바이   │                        │
│                                                               │
│  각 도시별:                                                    │
│  • 현재 시간 (실시간)                                          │
│  • KST 기준 시차                                              │
│  • 낮/밤 상태                                                  │
│  • 증시 개장/폐장 상태                                         │
│  • 서머타임(DST) 표시                                          │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

### 기술 스택

| 기술 | 용도 |
|------|------|
| **JavaScript Date API** | 시간 처리 |
| **IANA Timezone Database** | 타임존 정보 |
| **toLocaleString()** | 타임존 변환 |
| **setInterval()** | 실시간 업데이트 |

---

## 타임존 처리

### IANA 타임존 데이터베이스

**IANA Time Zone Database**는 전 세계 타임존의 역사적, 현재 규칙을 정의합니다.

```
형식: Area/Location
예시: Asia/Seoul, America/New_York, Europe/London
```

### 도시 데이터 구조

```javascript
const cities = {
    americas: [
        {
            name: '뉴욕',
            nameEn: 'New York',
            country: '미국 동부',
            flag: '🇺🇸',
            timezone: 'America/New_York',  // IANA 타임존
            market: 'NYSE/NASDAQ',
            marketHours: { open: 9.5, close: 16 },  // 9:30 AM - 4:00 PM
            hasDST: true  // 서머타임 적용 여부
        },
        // ...
    ],
    // europe, asia, oceania...
};
```

### toLocaleString()을 이용한 타임존 변환

JavaScript의 `toLocaleString()`은 **IANA 타임존**을 직접 지원합니다:

```javascript
function getTimeInTimezone(timezone) {
    const now = new Date();
    
    // 특정 타임존의 시간 문자열 생성
    const timeString = now.toLocaleString('en-US', { 
        timeZone: timezone 
    });
    
    // 문자열을 다시 Date 객체로 변환
    return new Date(timeString);
}

// 예시
getTimeInTimezone('America/New_York');  // 뉴욕 현재 시간
getTimeInTimezone('Europe/London');      // 런던 현재 시간
getTimeInTimezone('Asia/Tokyo');         // 도쿄 현재 시간
```

**왜 `en-US` 로케일을 사용하는가?**

`en-US` 로케일은 `MM/DD/YYYY, HH:MM:SS AM/PM` 형식으로 반환하며,
이 형식은 `new Date()`가 안정적으로 파싱할 수 있습니다.

```javascript
// en-US: "1/15/2026, 3:30:45 PM" → Date 파싱 성공
// ko-KR: "2026. 1. 15. 오후 3:30:45" → Date 파싱 실패 가능
```

### 시간 포맷팅

```javascript
function formatTime(date) {
    return date.toLocaleTimeString('ko-KR', {
        hour: '2-digit',
        minute: '2-digit',
        second: '2-digit',
        hour12: false  // 24시간제
    });
}

formatTime(new Date());  // "15:30:45"
```

```javascript
function formatDate(date) {
    const year = date.getFullYear();
    const month = date.getMonth() + 1;  // 0-indexed
    const day = date.getDate();
    const dayNames = ['일', '월', '화', '수', '목', '금', '토'];
    const dayName = dayNames[date.getDay()];
    
    return `${year}년 ${month}월 ${day}일 (${dayName})`;
}

formatDate(new Date());  // "2026년 1월 15일 (목)"
```

---

## 서머타임(DST) 자동 감지

### 서머타임이란?

**서머타임(Daylight Saving Time, DST)**은 여름철에 시계를 1시간 앞당기는 제도입니다.

| 지역 | 서머타임 | 기간 (대략) |
|------|---------|-------------|
| 미국/캐나다 | ✅ | 3월 둘째 일요일 ~ 11월 첫째 일요일 |
| 유럽 | ✅ | 3월 마지막 일요일 ~ 10월 마지막 일요일 |
| 호주 (일부) | ✅ | 10월 첫째 일요일 ~ 4월 첫째 일요일 |
| 일본/한국/중국 | ❌ | - |

### DST 자동 감지 알고리즘

```javascript
function isDST(timezone) {
    const now = new Date();
    
    // 1월 1일 (겨울)
    const jan = new Date(now.getFullYear(), 0, 1);
    // 7월 1일 (여름)
    const jul = new Date(now.getFullYear(), 6, 1);
    
    // 각 시점의 UTC 오프셋 계산
    const janOffset = new Date(
        jan.toLocaleString('en-US', { timeZone: timezone })
    ) - jan;
    
    const julOffset = new Date(
        jul.toLocaleString('en-US', { timeZone: timezone })
    ) - jul;
    
    const nowOffset = new Date(
        now.toLocaleString('en-US', { timeZone: timezone })
    ) - now;
    
    // 표준시 오프셋 (겨울 또는 여름 중 작은 값)
    const standardOffset = Math.min(janOffset, julOffset);
    
    // 현재 오프셋이 표준시와 다르면 DST 적용 중
    return nowOffset !== standardOffset;
}
```

**알고리즘 설명**:

```
북반구 (미국, 유럽):
┌─────────────────────────────────────────────┐
│  1월 (겨울)          7월 (여름)              │
│  표준시 (UTC-5)      서머타임 (UTC-4)         │
│     ▲                    ▲                  │
│     │                    │                  │
│  오프셋 -5시간        오프셋 -4시간           │
└─────────────────────────────────────────────┘

만약 현재가 7월이고 오프셋이 -4시간이면:
standardOffset = min(-5, -4) = -5  (밀리초로 계산됨)
nowOffset = -4
nowOffset ≠ standardOffset → DST 적용 중!

남반구 (호주):
1월이 여름, 7월이 겨울 → 반대로 동작
```

### DST 표시

```javascript
function getTimeDiffText(offset, hasDST, timezone) {
    // DST 적용 중이면 ☀️ 아이콘 표시
    const isDSTActive = hasDST && isDST(timezone);
    let dstIndicator = isDSTActive ? ' ☀️' : '';
    
    if (offset === 0) {
        return { 
            text: '동일' + dstIndicator, 
            class: 'same',
            isDST: isDSTActive 
        };
    } else if (offset > 0) {
        return { 
            text: `+${offset}시간` + dstIndicator, 
            class: 'ahead',
            isDST: isDSTActive 
        };
    } else {
        return { 
            text: `${offset}시간` + dstIndicator, 
            class: 'behind',
            isDST: isDSTActive 
        };
    }
}
```

---

## 시차 계산

### KST 기준 동적 시차 계산

서머타임 때문에 **시차는 고정값이 아닙니다**. 동적으로 계산해야 합니다:

```javascript
function getTimezoneOffset(timezone) {
    const now = new Date();
    
    // 서울 시간
    const seoulTime = new Date(
        now.toLocaleString('en-US', { timeZone: 'Asia/Seoul' })
    );
    
    // 대상 도시 시간
    const targetTime = new Date(
        now.toLocaleString('en-US', { timeZone: timezone })
    );
    
    // 차이 계산 (밀리초 → 시간)
    const diffMs = targetTime - seoulTime;
    const diffHours = diffMs / (1000 * 60 * 60);
    
    // 30분 단위 반올림 (인도 등 반시간대 처리)
    return Math.round(diffHours * 2) / 2;
}
```

**30분 단위 반올림 이유**:

```
인도 (IST): UTC+5:30
네팔 (NPT): UTC+5:45
이란 (IRST): UTC+3:30

getTimezoneOffset('Asia/Kolkata')  // -3.5 (KST 기준 3시간 30분 느림)
```

### 시차 예시

```
KST를 기준으로:

뉴욕 (EST/EDT):
├── 겨울 (표준시): KST -14시간
└── 여름 (서머타임): KST -13시간

런던 (GMT/BST):
├── 겨울 (표준시): KST -9시간
└── 여름 (서머타임): KST -8시간

시드니 (AEST/AEDT):
├── 겨울 (4-10월): KST +1시간
└── 여름 (10-4월): KST +2시간  ← 남반구는 반대!
```

---

## 증시 상태 계산

### 증시 정보 구조

```javascript
{
    market: 'NYSE/NASDAQ',
    marketHours: { 
        open: 9.5,   // 9:30 AM = 9 + 30/60
        close: 16    // 4:00 PM
    }
}
```

**시간을 소수로 표현하는 이유**:

```
9:30 AM = 9.5시간
4:30 PM = 16.5시간

비교 연산이 간단해짐:
if (currentHour >= 9.5 && currentHour < 16) {
    // 개장 중
}
```

### 증시 상태 계산 로직

```javascript
function getMarketStatus(localTime, marketOpen, marketClose) {
    const day = localTime.getDay();  // 0: 일요일, 6: 토요일
    
    // 현재 시간을 소수로 변환
    const currentHour = localTime.getHours() + localTime.getMinutes() / 60;
    
    // 주말 체크
    if (day === 0 || day === 6) {
        return { open: false, text: '휴장 (주말)' };
    }
    
    // 개장 시간 체크
    if (currentHour >= marketOpen && currentHour < marketClose) {
        return { open: true, text: '개장 중' };
    }
    
    return { open: false, text: '폐장' };
}
```

### 시각적 표현

```css
.market-status.open {
    color: #4ade80;  /* 녹색 */
}

.market-status.closed {
    color: #f87171;  /* 빨간색 */
}

/* 개장 중일 때 깜빡이는 점 */
.market-status.open::before {
    content: '';
    width: 6px;
    height: 6px;
    border-radius: 50%;
    background: currentColor;
    animation: pulse 2s infinite;
}

@keyframes pulse {
    0%, 100% { opacity: 1; }
    50% { opacity: 0.5; }
}
```

### 주요 증시 시간 (현지 시간)

| 증시 | 도시 | 개장 | 폐장 | 점심 휴장 |
|------|------|------|------|----------|
| NYSE/NASDAQ | 뉴욕 | 09:30 | 16:00 | ❌ |
| LSE | 런던 | 08:00 | 16:30 | ❌ |
| TSE | 도쿄 | 09:00 | 15:00 | 11:30-12:30 |
| SSE | 상하이 | 09:30 | 15:00 | 11:30-13:00 |
| HKEX | 홍콩 | 09:30 | 16:00 | 12:00-13:00 |

---

## 실시간 업데이트

### setInterval을 이용한 1초 업데이트

```javascript
// 초기화
initializeCards();
updateAllTimes();

// 1초마다 업데이트
setInterval(updateAllTimes, 1000);
```

### 전체 업데이트 함수

```javascript
function updateAllTimes() {
    // 1. 한국 시간 업데이트
    const koreaTime = getTimeInTimezone('Asia/Seoul');
    document.getElementById('korea-time').textContent = formatTime(koreaTime);
    document.getElementById('korea-date').textContent = formatDate(koreaTime);
    
    // 2. 모든 도시 카드 업데이트
    document.querySelectorAll('.time-card').forEach(card => {
        const timezone = card.dataset.timezone;
        const hasDST = card.dataset.hasDst === 'true';
        const localTime = getTimeInTimezone(timezone);
        
        // 시간 업데이트
        card.querySelector('[data-time]').textContent = formatTime(localTime);
        card.querySelector('[data-date]').textContent = formatShortDate(localTime);
        
        // 낮/밤 상태 업데이트
        const hour = localTime.getHours();
        const dayStatus = getDayStatus(hour);
        const dayStatusEl = card.querySelector('[data-day-status]');
        dayStatusEl.textContent = dayStatus.text;
        dayStatusEl.className = `day-status ${dayStatus.status}`;
        
        // 시차 업데이트 (DST 변화 반영)
        const offset = getTimezoneOffset(timezone);
        const timeDiff = getTimeDiffText(offset, hasDST, timezone);
        const timeDiffEl = card.querySelector('[data-time-diff]');
        timeDiffEl.textContent = timeDiff.text;
        timeDiffEl.className = `time-diff ${timeDiff.class}`;
        
        // 증시 상태 업데이트
        updateMarketStatus(card, localTime);
    });
}
```

### 낮/밤 상태 판단

```javascript
function getDayStatus(hour) {
    if (hour >= 6 && hour < 8) {
        return { status: 'dawn', text: '🌅 새벽' };
    }
    if (hour >= 8 && hour < 18) {
        return { status: 'day', text: '☀️ 낮' };
    }
    if (hour >= 18 && hour < 20) {
        return { status: 'dusk', text: '🌇 저녁' };
    }
    return { status: 'night', text: '🌙 밤' };
}
```

```
시간대별 상태:
  0시 ─────── 6시 ─── 8시 ──────────── 18시 ── 20시 ── 24시
     🌙 밤      🌅     ☀️ 낮           🌇      🌙 밤
              새벽                    저녁
```

---

## UI 구현

### data-* 속성을 이용한 데이터 바인딩

```javascript
// 카드 생성 시 데이터 저장
card.dataset.timezone = city.timezone;
card.dataset.hasDst = city.hasDST ? 'true' : 'false';
card.dataset.market = city.market || '';
card.dataset.marketOpen = city.marketHours?.open || '';
card.dataset.marketClose = city.marketHours?.close || '';
```

```html
<!-- 생성된 HTML -->
<div class="time-card" 
     data-timezone="America/New_York"
     data-has-dst="true"
     data-market="NYSE/NASDAQ"
     data-market-open="9.5"
     data-market-close="16">
```

**왜 data-* 속성을 사용하는가?**

1. **DOM에서 직접 접근**: `card.dataset.timezone`
2. **CSS 선택자로 활용 가능**: `[data-market]`
3. **업데이트 시 다시 조회할 필요 없음**

### 카드 생성 함수

```javascript
function createCityCard(city, index) {
    const card = document.createElement('div');
    card.className = 'time-card';
    
    // 순차적 애니메이션
    card.style.animationDelay = `${0.4 + index * 0.05}s`;
    
    // 데이터 속성 설정
    card.dataset.timezone = city.timezone;
    // ...
    
    // 초기 시차 계산
    const offset = getTimezoneOffset(city.timezone);
    const timeDiff = getTimeDiffText(offset, city.hasDST, city.timezone);
    
    card.innerHTML = `
        <div class="card-header">
            <div class="city-info">
                <span class="city-flag">${city.flag}</span>
                <div>
                    <div class="city-name">${city.name}</div>
                    <div class="city-country">${city.nameEn} · ${city.country}</div>
                </div>
            </div>
            <span class="time-diff ${timeDiff.class}" data-time-diff>
                ${timeDiff.text}
            </span>
        </div>
        <div class="current-time" data-time>--:--:--</div>
        <div class="current-date" data-date>--/-- (--)</div>
        <div class="day-status" data-day-status>--</div>
        ${marketHtml}
    `;
    
    return card;
}
```

### 순차적 페이드인 애니메이션

```css
.time-card {
    opacity: 0;
    animation: fadeInUp 0.5s ease forwards;
}

@keyframes fadeInUp {
    from {
        opacity: 0;
        transform: translateY(20px);
    }
    to {
        opacity: 1;
        transform: translateY(0);
    }
}
```

```javascript
// 각 카드마다 0.05초씩 지연
card.style.animationDelay = `${0.4 + index * 0.05}s`;

// 결과:
// 카드 0: 0.40초 후 시작
// 카드 1: 0.45초 후 시작
// 카드 2: 0.50초 후 시작
// ...
```

### 시차 표시 스타일

```css
.time-diff {
    font-family: 'JetBrains Mono', monospace;
    padding: 6px 12px;
    border-radius: 20px;
}

/* KST보다 느림 (미주, 유럽) */
.time-diff.behind {
    background: rgba(239, 68, 68, 0.15);
    color: #f87171;
}

/* KST보다 빠름 (호주, 뉴질랜드) */
.time-diff.ahead {
    background: rgba(34, 197, 94, 0.15);
    color: #4ade80;
}

/* KST와 동일 (일본) */
.time-diff.same {
    background: rgba(212, 175, 55, 0.15);
    color: var(--accent-gold);
}
```

### 반응형 그리드

```css
.time-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
    gap: 20px;
}

/* auto-fill + minmax 조합:
   - 최소 280px, 최대 1fr
   - 화면 너비에 따라 열 개수 자동 조절
   
   예시:
   - 1400px 화면: 4열
   - 900px 화면: 3열
   - 600px 화면: 2열
   - 300px 화면: 1열
*/

@media (max-width: 768px) {
    .time-grid {
        grid-template-columns: 1fr;  /* 강제 1열 */
    }
}
```

---

## 성능 고려사항

### 1초 간격의 적절성

| 간격 | 장점 | 단점 |
|------|------|------|
| 100ms | 부드러운 업데이트 | CPU 사용량 증가 |
| **1000ms** | **적절한 균형** | **1초 오차 가능** |
| 60000ms | 가벼움 | 실시간 아님 |

초 단위 표시가 있으므로 **1초가 적절**합니다.

### DOM 업데이트 최적화

```javascript
// ✅ 좋음: 한 번에 여러 요소 업데이트
document.querySelectorAll('.time-card').forEach(card => {
    card.querySelector('[data-time]').textContent = time;
    card.querySelector('[data-date]').textContent = date;
});

// ❌ 나쁨: 매번 새로운 HTML 생성
card.innerHTML = `<div>...</div>`;  // 리플로우 발생
```

### requestAnimationFrame 대안

더 부드러운 업데이트가 필요하다면:

```javascript
let lastUpdate = 0;

function animate(timestamp) {
    if (timestamp - lastUpdate >= 1000) {
        updateAllTimes();
        lastUpdate = timestamp;
    }
    requestAnimationFrame(animate);
}

requestAnimationFrame(animate);
```

---

## 마치며

### 핵심 기술 요약

| 기능 | 기술 |
|------|------|
| 타임존 변환 | `toLocaleString({ timeZone })` |
| DST 감지 | 1월/7월 오프셋 비교 |
| 시차 계산 | 밀리초 차이 → 시간 변환 |
| 증시 상태 | 현지 시간 + 요일 체크 |
| 실시간 업데이트 | `setInterval(fn, 1000)` |

### IANA 타임존의 장점

1. **DST 자동 처리**: 브라우저가 알아서 계산
2. **역사적 변화 반영**: 과거 규칙 변경도 처리
3. **표준화**: 전 세계 공통 식별자

### 참고 자료

- [IANA Time Zone Database](https://www.iana.org/time-zones)
- [MDN toLocaleString()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Date/toLocaleString)
- [Daylight Saving Time 규칙](https://www.timeanddate.com/time/dst/)

---

*작성일: 2026년 1월*


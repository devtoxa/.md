# 모션 그래픽 만들기 — HTML / CSS / JS → Canvas → 영상

> 브라우저에서 코드로 애니메이션을 만들고, 그 결과를 **영상 파일(mp4/webm)** 로 뽑아내는 전체 과정을 정리한 문서입니다.
> 예제 규격은 **1920×1080 (16:9) / 60fps** 기준이고, 상수만 바꾸면 9:16 숏폼으로도 쓸 수 있습니다.

## 목차

1. [전체 파이프라인](#1-전체-파이프라인)
2. [환경 준비](#2-환경-준비)
3. [Canvas 렌더링 기초](#3-canvas-렌더링-기초)
4. [애니메이션 설계 — 결정론적 render(t)](#4-애니메이션-설계--결정론적-rendert)
5. [예제 코드 (index.html / style.css / main.js)](#5-예제-코드)
6. [방법 A — MediaRecorder로 바로 녹화](#6-방법-a--mediarecorder로-바로-녹화)
7. [방법 B — 프레임 렌더 + ffmpeg (고품질)](#7-방법-b--프레임-렌더--ffmpeg-고품질)
8. [그 외 변환 방식들](#8-그-외-변환-방식들)
9. [인코딩·포맷 정리](#9-인코딩포맷-정리)
10. [오디오 넣기](#10-오디오-넣기)
11. [트러블슈팅 체크리스트](#11-트러블슈팅-체크리스트)
12. [참고 링크](#12-참고-링크)

---

## 1. 전체 파이프라인

```
[1] 기획 / 스토리보드
        │  "무엇이, 언제, 어떻게 움직이는가"를 초 단위로 적는다
        ▼
[2] render(t) 타임라인 구현
        │  HTML + CSS + JS, <canvas>에 그림
        ▼
[3] VSCode + Live Server로 실시간 미리보기
        │  여기서 90%의 시간을 쓴다
        ▼
[4-A] MediaRecorder 즉시 녹화          [4-B] 프레임 렌더 + ffmpeg
      (빠른 확인, webm/mp4)                 (최고 품질, PNG 시퀀스 → mp4)
        └──────────────┬───────────────────────┘
                       ▼
[5] ffmpeg 후처리 / 편집 (자막, BGM, 색보정)
                       ▼
[6] 최종 mp4 배포
```

핵심 아이디어는 하나입니다.

> **애니메이션을 "실시간 루프"가 아니라 "시간 t를 받는 함수"로 만든다.**

이렇게 만들면 같은 코드로 ① 화면 미리보기 ② 실시간 녹화 ③ 프레임 단위 정밀 렌더를 **모두** 할 수 있습니다.

---

## 2. 환경 준비

### 2.1 폴더 구조

```
motion-2d/
├─ index.html
├─ style.css
├─ main.js          # 씬 + 타임라인
├─ render.mjs       # (방법 B) 프레임 캡처 스크립트
├─ package.json
├─ assets/          # 이미지, 폰트, 사운드
├─ frames/          # 렌더된 PNG 시퀀스 (자동 생성)
└─ out/             # 최종 영상
```

### 2.2 VSCode 확장

| 확장 | 용도 |
|---|---|
| **Live Server** | `index.html`을 로컬 서버로 열기 (파일 직접 열기보다 안전) |
| **Prettier** | 코드 정리 |
| **Error Lens** | 콘솔/문법 오류 즉시 표시 |

`index.html` 우클릭 → **Open with Live Server** → `http://127.0.0.1:5500`.
파일 저장 시 자동 새로고침되므로 미리보기가 편합니다.

### 2.3 ffmpeg 설치 (Windows)

```powershell
winget install --id Gyan.FFmpeg -e
```

설치 후 **새 터미널**에서 확인:

```powershell
ffmpeg -version
ffprobe -version
```

`'ffmpeg'은(는) 인식되지 않습니다` 가 뜨면 PATH 문제입니다.

- `C:\Users\<user>\AppData\Local\Microsoft\WinGet\Packages\Gyan.FFmpeg_...\bin` 를 PATH에 추가
- 또는 ffmpeg 바이너리를 프로젝트 폴더에 두고 `.\ffmpeg.exe` 로 직접 호출

### 2.4 Node + puppeteer (방법 B용)

```powershell
node -v      # 18 이상 권장
npm init -y
npm i puppeteer
```

`puppeteer`는 첫 설치 시 Chromium을 자동 다운로드합니다(약 150MB).

---

## 3. Canvas 렌더링 기초

### 3.1 해상도와 DPR — 영상에서는 DPR을 쓰지 않는다

```js
const W = 1920, H = 1080;
canvas.width = W;     // 백킹 스토어 = 최종 영상 픽셀
canvas.height = H;
```

- **화면 미리보기**에서는 고해상도 모니터에서 흐려질 수 있지만,
- **영상 결과물**의 픽셀은 캔버스 백킹 스토어와 1:1이어야 합니다.

그래서 원칙은 이렇게 나눕니다.

| 목적 | 캔버스 크기 |
|---|---|
| 영상 렌더 | `1920 × 1080` 고정 (`devicePixelRatio` 무시) |
| 화면 표시 | CSS `width: 100%` 로 축소 (비율 유지) |

```css
#c {
  display: block;
  width: min(100vw, calc(100vh * 16 / 9));
  aspect-ratio: 16 / 9;
  background: #000;
}
```

### 3.2 좌표계 · transform

```js
ctx.save();
ctx.translate(W / 2, H / 2);   // 원점을 화면 중앙으로
ctx.rotate(angle);
ctx.scale(s, s);
ctx.fillRect(-50, -50, 100, 100);
ctx.restore();                 // 반드시 되돌리기
```

`save()/restore()` 짝을 안 맞추면 이후 모든 프레임이 누적으로 틀어집니다.
**프레임 함수 안에서는 항상 `save()` → 그리기 → `restore()`** 를 습관으로.

### 3.3 자주 쓰는 그리기 API

```js
// 도형
ctx.fillRect(x, y, w, h);
ctx.beginPath(); ctx.arc(cx, cy, r, 0, Math.PI * 2); ctx.fill();
ctx.roundRect(x, y, w, h, 24);        // 모서리 둥근 사각형 (Chrome 99+)

// 텍스트
ctx.font = '700 120px Pretendard, "Malgun Gothic", sans-serif';
ctx.textBaseline = 'middle';
ctx.textAlign = 'left';
ctx.fillText('MOTION 2D', x, y);

// 그라디언트
const g = ctx.createLinearGradient(0, 0, W, H);
g.addColorStop(0, '#6ee7ff');
g.addColorStop(1, '#a78bfa');
ctx.fillStyle = g;

// 합성 모드
ctx.globalCompositeOperation = 'lighter';   // 빛 번짐
ctx.globalAlpha = 0.6;

// 블러 (그림자 형태로)
ctx.shadowColor = 'rgba(110,231,255,.8)';
ctx.shadowBlur = 40;

// 클리핑 (마스크)
ctx.save();
ctx.beginPath(); ctx.rect(0, 300, W, 200); ctx.clip();
ctx.fillText('REVEAL', 160, 420);
ctx.restore();
```

### 3.4 Path2D로 복잡한 모양 재사용

매 프레임 SVG path 문자열을 파싱하면 느립니다. 한 번 만들어 캐싱하세요.

```js
const star = new Path2D('M 0 -50 L 14 -15 L 52 -15 L 22 6 L 33 43 L 0 22 L -33 43 L -22 6 L -52 -15 L -14 -15 Z');
ctx.save(); ctx.translate(960, 540); ctx.scale(3, 3);
ctx.fillStyle = '#fbbf24'; ctx.fill(star);
ctx.restore();
```

---

## 4. 애니메이션 설계 — 결정론적 render(t)

### 4.1 왜 `render(t)` 인가

`requestAnimationFrame`은 **모니터 주사율과 브라우저 부하에 따라 호출 시점이 달라집니다.**
그래서 "프레임 카운터"에 의존하면 같은 코드가 매번 다른 영상이 됩니다.

```js
// ❌ 나쁜 예: 프레임 개수에 의존 → 실행마다 결과가 다름
let frame = 0;
function loop() {
  frame++;
  drawCircle(frame * 10);
  requestAnimationFrame(loop);
}

// ✅ 좋은 예: 시간(초)을 인자로 받는 순수 함수
function render(t) {
  drawCircle(easeOutCubic(clamp(t / 1.5)) * 800);
}
```

`render(t)`는 **같은 t에 대해 항상 같은 그림**을 그려야 합니다. 이 성질(결정론)이 있어야
프레임 단위 렌더(방법 B)가 가능하고, 미리보기와 결과물이 일치합니다.

### 4.2 미리보기 루프

```js
const FPS = 60;
const DURATION = 5;          // 영상 길이(초)
let startTime = performance.now();

function previewLoop(now) {
  const t = ((now - startTime) / 1000) % DURATION;  // 0 ~ DURATION 반복
  render(t);
  requestAnimationFrame(previewLoop);
}
requestAnimationFrame(previewLoop);
```

### 4.3 이징 · 보간 유틸

```js
const clamp = (v, a = 0, b = 1) => Math.min(b, Math.max(a, v));
const lerp = (a, b, t) => a + (b - a) * t;

// 구간 진행도: t가 [start, start+dur] 사이일 때 0~1
const seg = (t, start, dur) => clamp((t - start) / dur);

const easeOutCubic = t => 1 - Math.pow(1 - t, 3);
const easeInOutCubic = t => (t < 0.5 ? 4 * t * t * t : 1 - Math.pow(-2 * t + 2, 3) / 2);
const easeOutBack = t => {
  const c1 = 1.70158, c3 = c1 + 1;
  return 1 + c3 * Math.pow(t - 1, 3) + c1 * Math.pow(t - 1, 2);
};
const easeOutExpo = t => (t === 1 ? 1 : 1 - Math.pow(2, -10 * t));
```

### 4.4 타임라인 헬퍼 (선택)

씬이 많아지면 "0.3초에 페이드인, 1.2초에 이동" 같은 정보를 구조화하는 게 좋습니다.

```js
// 키프레임 보간: [[time, value], ...] → 해당 시각의 값
function keyframes(t, keys, ease = easeInOutCubic) {
  if (t <= keys[0][0]) return keys[0][1];
  for (let i = 0; i < keys.length - 1; i++) {
    const [t0, v0] = keys[i];
    const [t1, v1] = keys[i + 1];
    if (t <= t1) return lerp(v0, v1, ease((t - t0) / (t1 - t0)));
  }
  return keys[keys.length - 1][1];
}

// 사용 예
const x = keyframes(t, [[0, -200], [0.8, 960], [1.4, 960], [2.0, 400]]);
const opacity = keyframes(t, [[0, 0], [0.4, 1]]);
```

### 4.5 결정론 체크리스트

렌더 결과가 실행마다 달라지는 원인은 거의 아래 5가지입니다.

| 원인 | 해결 |
|---|---|
| 폰트가 늦게 로드되어 fallback으로 그려짐 | `await document.fonts.ready` 후 렌더 시작 |
| 이미지 미로드 상태로 그려짐 | `Promise.all(images.map(img => img.decode()))` |
| `Math.random()` | 시드 고정 난수 사용 (아래) |
| `Date.now()`, `performance.now()` | `render(t)` 안에서는 절대 사용 금지 |
| CSS 애니메이션/트랜지션 | 캔버스 밖 요소는 영상에 안 잡히거나 타이밍이 어긋남 → 캔버스 안에서만 |

```js
// 시드 고정 난수 (mulberry32)
function makeRandom(seed = 12345) {
  let a = seed >>> 0;
  return function () {
    a = (a + 0x6d2b79f5) >>> 0;
    let t = Math.imul(a ^ (a >>> 15), 1 | a);
    t = (t + Math.imul(t ^ (t >>> 7), 61 | t)) ^ t;
    return ((t ^ (t >>> 14)) >>> 0) / 4294967296;
  };
}
const rand = makeRandom(2024);   // 프레임마다 새로 만들지 말고, 파티클 초기화 때만
```

> 파티클처럼 난수가 필요하면 **초기 위치를 미리 배열로 만들어 두고** `render(t)`에서는 그 배열만 읽습니다.
> `render()` 안에서 `Math.random()`을 호출하면 프레임마다 위치가 바뀌어 깜빡입니다.

---

## 5. 예제 코드

5초짜리 타이틀 모션: 그리드 배경 → 제목 마스크 리빌 → 원 이동 → 하단 진행 바.

### 5.1 `index.html`

```html
<!DOCTYPE html>
<html lang="ko">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>Motion 2D</title>
  <link rel="stylesheet" href="style.css" />
</head>
<body>
  <div id="stage">
    <canvas id="c" width="1920" height="1080"></canvas>
  </div>

  <div id="ui">
    <button id="btn-play">재생 / 정지</button>
    <button id="btn-rec">녹화 시작 (5초)</button>
    <span id="status">ready</span>
  </div>

  <script src="main.js"></script>
</body>
</html>
```

### 5.2 `style.css`

```css
* { box-sizing: border-box; }

html, body {
  margin: 0;
  height: 100%;
  background: #05070d;
  color: #c9d1d9;
  font-family: -apple-system, "Segoe UI", "Malgun Gothic", sans-serif;
  overflow: hidden;
}

#stage {
  height: 100vh;
  display: grid;
  place-items: center;
}

#c {
  display: block;
  width: min(100vw, calc((100vh - 64px) * 16 / 9));
  aspect-ratio: 16 / 9;
  background: #000;
  box-shadow: 0 0 60px rgba(0, 0, 0, .8);
}

#ui {
  position: fixed;
  left: 0; right: 0; bottom: 0;
  height: 56px;
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 0 16px;
  background: #0d1117;
  border-top: 1px solid #21262d;
  font-size: 13px;
}

#ui button {
  background: #21262d;
  color: #e6edf3;
  border: 1px solid #30363d;
  border-radius: 6px;
  padding: 6px 14px;
  cursor: pointer;
}
#ui button:hover { background: #30363d; }
#ui button.recording { background: #b62324; border-color: #f85149; }

/* 렌더 모드에서는 UI를 숨긴다 */
body.render-mode #ui { display: none; }
body.render-mode #c { width: 1920px; height: 1080px; }
```

### 5.3 `main.js`

```js
/* ============================================================
   Motion 2D — 결정론적 타임라인 예제
   - render(t) 하나로 미리보기 / 녹화 / 프레임 렌더를 모두 처리
   ============================================================ */

const W = 1920;
const H = 1080;
const FPS = 60;
const DURATION = 5;               // 초

const canvas = document.getElementById('c');
const ctx = canvas.getContext('2d', { alpha: false });

/* ---------- 유틸 ---------- */
const clamp = (v, a = 0, b = 1) => Math.min(b, Math.max(a, v));
const lerp = (a, b, t) => a + (b - a) * t;
const seg = (t, start, dur) => clamp((t - start) / dur);
const easeOutCubic = t => 1 - Math.pow(1 - t, 3);
const easeInOutCubic = t => (t < 0.5 ? 4 * t * t * t : 1 - Math.pow(-2 * t + 2, 3) / 2);
const easeOutExpo = t => (t === 1 ? 1 : 1 - Math.pow(2, -10 * t));

/* ---------- 씬 ---------- */
function drawBackground(t) {
  const g = ctx.createLinearGradient(0, 0, W, H);
  g.addColorStop(0, '#080b14');
  g.addColorStop(1, '#111a2e');
  ctx.fillStyle = g;
  ctx.fillRect(0, 0, W, H);

  // 천천히 흐르는 그리드
  const step = 80;
  const off = (t * 24) % step;
  ctx.strokeStyle = 'rgba(255,255,255,.05)';
  ctx.lineWidth = 1;
  ctx.beginPath();
  for (let x = -step + off; x < W + step; x += step) {
    ctx.moveTo(x, 0); ctx.lineTo(x, H);
  }
  for (let y = -step + off; y < H + step; y += step) {
    ctx.moveTo(0, y); ctx.lineTo(W, y);
  }
  ctx.stroke();
}

function drawGlow(t) {
  const p = easeInOutCubic(seg(t, 0, 2.4));
  const x = lerp(300, 1600, p);
  const y = 540 + Math.sin(t * 2.2) * 90;

  const g = ctx.createRadialGradient(x, y, 0, x, y, 320);
  g.addColorStop(0, 'rgba(110,231,255,.55)');
  g.addColorStop(1, 'rgba(110,231,255,0)');
  ctx.fillStyle = g;
  ctx.beginPath();
  ctx.arc(x, y, 320, 0, Math.PI * 2);
  ctx.fill();

  // 원 본체
  ctx.beginPath();
  ctx.arc(x, y, 46, 0, Math.PI * 2);
  ctx.fillStyle = '#6ee7ff';
  ctx.fill();
}

function drawTitle(t) {
  const p = easeOutExpo(seg(t, 0.25, 1.0));
  if (p <= 0) return;

  ctx.save();
  ctx.beginPath();
  ctx.rect(0, 340, W, 210);          // 리빌 마스크
  ctx.clip();

  ctx.font = '800 150px Pretendard, "Malgun Gothic", sans-serif';
  ctx.textBaseline = 'middle';
  ctx.fillStyle = '#f2f6ff';
  ctx.fillText('MOTION 2D', 160, 452 + (1 - p) * 190);

  ctx.restore();
}

function drawSubtitle(t) {
  const p = easeOutCubic(seg(t, 1.1, 0.7));
  if (p <= 0) return;

  ctx.save();
  ctx.globalAlpha = p;
  ctx.font = '400 46px Pretendard, "Malgun Gothic", sans-serif';
  ctx.fillStyle = 'rgba(230,237,243,.72)';
  ctx.fillText('HTML · CSS · JS  →  Canvas  →  Video', 164, 560 + (1 - p) * 24);
  ctx.restore();
}

function drawProgress(t) {
  const p = clamp(t / DURATION);
  const y = 980, x = 160, w = W - 320, h = 4;

  ctx.fillStyle = 'rgba(255,255,255,.12)';
  ctx.fillRect(x, y, w, h);

  ctx.fillStyle = '#a78bfa';
  ctx.fillRect(x, y, w * p, h);
}

function drawTimecode(t) {
  ctx.font = '500 30px ui-monospace, Consolas, monospace';
  ctx.fillStyle = 'rgba(255,255,255,.35)';
  ctx.textAlign = 'right';
  ctx.fillText(`${t.toFixed(2)}s`, W - 160, 120);
  ctx.textAlign = 'left';
}

/* ---------- 마스터 렌더 ---------- */
function render(t) {
  ctx.save();
  ctx.clearRect(0, 0, W, H);
  drawBackground(t);
  drawGlow(t);
  drawTitle(t);
  drawSubtitle(t);
  drawProgress(t);
  drawTimecode(t);
  ctx.restore();
}

/* ---------- 미리보기 루프 ---------- */
let playing = true;
let startTime = performance.now();

function previewLoop(now) {
  if (playing) {
    const t = ((now - startTime) / 1000) % DURATION;
    render(t);
  }
  requestAnimationFrame(previewLoop);
}
requestAnimationFrame(previewLoop);

document.getElementById('btn-play').onclick = () => {
  playing = !playing;
  if (playing) startTime = performance.now();
};

/* ---------- 렌더 모드 (?render=1) : puppeteer가 사용 ---------- */
const isRenderMode = new URLSearchParams(location.search).has('render');
if (isRenderMode) {
  document.body.classList.add('render-mode');
  playing = false;

  // 특정 프레임을 그리고 PNG dataURL을 돌려준다
  window.__renderFrame = (frame) => {
    render(frame / FPS);
    return canvas.toDataURL('image/png');
  };
}

/* ---------- 준비 완료 신호 ---------- */
(async () => {
  await document.fonts.ready;
  window.__ready = true;
  document.getElementById('status').textContent = 'ready';
})();
```

**포인트 정리**

- `render(t)` 안에는 시간 인자 외의 외부 상태 변화가 없다 → 몇 번을 실행해도 동일 결과
- 배경 그리드 오프셋도 `t`로 계산 (`t * 24`) → 프레임 렌더와 미리보기가 똑같이 흐름
- `?render=1` 모드에서만 `window.__renderFrame(frame)`을 노출 → 자동화 스크립트가 호출
- `document.fonts.ready` 이후 `__ready = true` → 스크립트가 이 플래그를 기다림

---

## 6. 방법 A — MediaRecorder로 바로 녹화

브라우저가 캔버스를 **실시간으로** 인코딩합니다. 설치할 게 없고 5초면 결과가 나옵니다.

### 6.1 기본 코드

```js
async function recordCanvas(durationSec = DURATION) {
  const stream = canvas.captureStream(FPS);   // 60fps 요청

  // 브라우저가 지원하는 코덱 선택 (Chrome 126+ 는 mp4도 가능)
  const candidates = [
    'video/mp4;codecs=avc1.640028',
    'video/webm;codecs=vp9',
    'video/webm;codecs=vp8',
  ];
  const mimeType = candidates.find(t => MediaRecorder.isTypeSupported(t));

  const rec = new MediaRecorder(stream, {
    mimeType,
    videoBitsPerSecond: 16_000_000,   // 1080p60 기준 12~20Mbps
  });

  const chunks = [];
  rec.ondataavailable = e => { if (e.data.size) chunks.push(e.data); };

  const stopped = new Promise(resolve => { rec.onstop = resolve; });

  // 항상 0초부터 녹화되도록 타임라인 리셋
  startTime = performance.now();
  playing = true;

  rec.start();
  await new Promise(r => setTimeout(r, durationSec * 1000));
  rec.stop();
  await stopped;

  const blob = new Blob(chunks, { type: mimeType });
  download(blob, mimeType.includes('mp4') ? 'motion.mp4' : 'motion.webm');
}

function download(blob, name) {
  const a = document.createElement('a');
  a.href = URL.createObjectURL(blob);
  a.download = name;
  a.click();
  setTimeout(() => URL.revokeObjectURL(a.href), 5000);
}
```

버튼에 연결:

```js
document.getElementById('btn-rec').onclick = async (e) => {
  e.target.classList.add('recording');
  e.target.textContent = '녹화 중…';
  await recordCanvas();
  e.target.classList.remove('recording');
  e.target.textContent = '녹화 시작 (5초)';
};
```

### 6.2 장단점

| 장점 | 단점 |
|---|---|
| 설치 없음, 즉시 확인 | **실시간 대기** (5초 영상 = 5초 소요, 4K는 더 느림) |
| 오디오도 같은 스트림으로 합성 가능 | 프레임 드랍 발생 가능 → 모션 끊김 |
| 브라우저 API만 사용 | 출력이 **가변 프레임레이트(VFR)** → 편집기에서 싱크 어긋남 |
| | 코덱/화질 제어가 거칠다 |

### 6.3 실전 팁

- **`captureStream(0)` + `requestFrame()`**: 수동 프레임 제어. 다만 MediaRecorder의 타임스탬프는 **실시간** 기준이라, 프레임을 빨리 밀어넣어도 영상이 짧아지지 않습니다. 정확한 프레임 타이밍이 필요하면 방법 B를 쓰세요.
- **탭이 백그라운드로 가면 `requestAnimationFrame`이 멈춥니다.** 녹화 중에는 다른 탭으로 전환하지 마세요.
- **VFR 보정**은 ffmpeg로:

```powershell
ffmpeg -i motion.webm -r 60 -vsync cfr -c:v libx264 -crf 18 -pix_fmt yuv420p out/motion.mp4
```

- 화질이 뭉개지면 `videoBitsPerSecond`를 올리고, 그래도 안 되면 방법 B로 갑니다.

### 6.4 webm → mp4 변환

```powershell
ffmpeg -i motion.webm -c:v libx264 -preset slow -crf 18 -pix_fmt yuv420p -movflags +faststart out/motion.mp4
```

> `yuv420p`는 사실상 필수입니다. 기본값이 `yuv444p`/`yuv420p10le`로 나오면
> 프리미어·유튜브·아이폰에서 "재생 불가"가 됩니다.

---

## 7. 방법 B — 프레임 렌더 + ffmpeg (고품질)

**한 프레임씩 정확히 그리고 PNG로 저장한 뒤 ffmpeg로 묶습니다.**
실시간 대기가 없고, 드랍이 없고, 4K·알파·장시간도 안정적입니다. 실무 권장 방식.

### 7.1 `render.mjs`

```js
import puppeteer from 'puppeteer';
import fs from 'node:fs/promises';
import path from 'node:path';
import { pathToFileURL } from 'node:url';

const FPS = 60;
const DURATION = 5;
const OUT_DIR = path.resolve('frames');
const TOTAL = Math.round(FPS * DURATION);

/* 1) 출력 폴더 초기화 */
await fs.rm(OUT_DIR, { recursive: true, force: true });
await fs.mkdir(OUT_DIR, { recursive: true });

/* 2) 브라우저 실행 */
const browser = await puppeteer.launch({
  headless: 'new',
  args: ['--allow-file-access-from-files', '--force-device-scale-factor=1'],
});

const page = await browser.newPage();
await page.setViewport({ width: 1920, height: 1080, deviceScaleFactor: 1 });

page.on('console', msg => console.log('[page]', msg.text()));
page.on('pageerror', err => console.error('[page error]', err));

/* 3) 페이지 로드 + 폰트/이미지 준비 대기 */
const url = pathToFileURL(path.resolve('index.html')).href + '?render=1';
await page.goto(url, { waitUntil: 'networkidle0' });
await page.waitForFunction('window.__ready === true', { timeout: 30000 });

/* 4) 프레임 루프 */
const t0 = Date.now();
for (let f = 0; f < TOTAL; f++) {
  const dataUrl = await page.evaluate(frame => window.__renderFrame(frame), f);
  const base64 = dataUrl.slice(dataUrl.indexOf(',') + 1);
  const file = path.join(OUT_DIR, `${String(f).padStart(5, '0')}.png`);
  await fs.writeFile(file, Buffer.from(base64, 'base64'));

  if (f % 15 === 0 || f === TOTAL - 1) {
    const elapsed = (Date.now() - t0) / 1000;
    const rate = (f + 1) / elapsed;
    const remain = Math.max(0, Math.round((TOTAL - f - 1) / rate));
    process.stdout.write(`\r  ${f + 1}/${TOTAL} frames  ${rate.toFixed(1)} fps  남은 ${remain}s   `);
  }
}

await browser.close();
console.log(`\n✔ ${TOTAL} 프레임 저장 완료 → ${OUT_DIR}`);
```

### 7.2 `package.json` 스크립트

```json
{
  "name": "motion-2d",
  "type": "module",
  "scripts": {
    "render": "node render.mjs",
    "encode": "ffmpeg -y -framerate 60 -i frames/%05d.png -c:v libx264 -preset slow -crf 18 -pix_fmt yuv420p -movflags +faststart out/motion.mp4",
    "build": "npm run render && npm run encode"
  },
  "devDependencies": {
    "puppeteer": "^23.0.0"
  }
}
```

실행:

```powershell
npm run build
```

### 7.3 ffmpeg 인코딩 옵션 해설

```powershell
ffmpeg -y ^
  -framerate 60 -i frames/%05d.png ^
  -c:v libx264 -preset slow -crf 18 ^
  -pix_fmt yuv420p -movflags +faststart ^
  out/motion.mp4
```

| 옵션 | 의미 |
|---|---|
| `-framerate 60` | 입력 PNG 시퀀스의 초당 프레임 수 (`-r` 아님!) |
| `-preset slow` | 느리지만 압축 효율 ↑ (`medium`도 충분) |
| `-crf 18` | 품질 기준. 0(무손실)~51(최저). 16~20이 배포용 |
| `-pix_fmt yuv420p` | 호환성 최우선 픽셀 포맷 |
| `-movflags +faststart` | 웹 스트리밍 시 즉시 재생 |

### 7.4 고급: PNG 대신 raw 프레임 파이프 (빠름)

PNG 인코딩/디코딩이 병목이면 RGBA 원본을 바로 ffmpeg에 파이프합니다.

```js
// render.mjs 대체 루프 (raw RGBA)
import { spawn } from 'node:child_process';

const ff = spawn('ffmpeg', [
  '-y', '-f', 'rawvideo', '-pix_fmt', 'rgba',
  '-s', '1920x1080', '-framerate', '60',
  '-i', '-', '-c:v', 'libx264', '-crf', '18',
  '-pix_fmt', 'yuv420p', '-movflags', '+faststart',
  'out/motion.mp4',
], { stdio: ['pipe', 'inherit', 'inherit'] });

for (let f = 0; f < TOTAL; f++) {
  const b64 = await page.evaluate(frame => {
    window.__renderFrame(frame);
    const c = document.getElementById('c');
    return c.toDataURL('image/png');  // 필요 시 toBlob + raw 변환으로 교체
  }, f);
  // (실제 구현에서는 getImageData().data 를 base64로 넘겨 Buffer로 복원)
}
ff.stdin.end();
```

> PNG 경로가 단순하고 디버깅이 쉬우므로, **먼저 PNG로 성공시키고** 속도가 문제일 때만 파이프로 최적화하세요.

### 7.5 고급: 알파 채널(투명 배경) 영상

편집 프로그램에서 배경 위에 얹을 목적이면 투명도가 필요합니다.

```powershell
# ProRes 4444 (Premiere / Final Cut / DaVinci 친화적)
ffmpeg -framerate 60 -i frames/%05d.png -c:v prores_ks -profile:v 4444 ^
       -pix_fmt yuva444p10le out/motion_alpha.mov

# WebM VP9 알파 (웹 임베드용)
ffmpeg -framerate 60 -i frames/%05d.png -c:v libvpx-vp9 -pix_fmt yuva420p -crf 24 out/motion_alpha.webm
```

캔버스는 `getContext('2d')` 기본값이 알파 지원이므로, `clearRect`만 하고 배경을 안 채우면 됩니다.

---

## 8. 그 외 변환 방식들

| 방식 | 품질 / 정확도 | 난이도 | 특징 |
|---|---|---|---|
| **MediaRecorder + `captureStream`** | 중 | ★ | 실시간, 설치 불필요, webm(VFR) → ffmpeg 보정 권장 |
| **프레임 렌더 + ffmpeg** | 최상 | ★★ | 프레임 정확, 4K·알파·장시간 안정. **권장** |
| **WebCodecs + mp4-muxer** | 상 | ★★ | 브라우저 안에서 mp4 완성. 실시간 아님(프레임 밀어넣기) |
| **화면 녹화 (OBS / Win+G / ScreenToGif)** | 하~중 | ☆ | 프로토타입 확인용. 커서·알림 섞이고 타이밍 부정확 |
| **Remotion (React)** | 상 | ★★★ | React 컴포넌트 = 영상. `npx remotion render` 로 mp4 |
| **Motion Canvas** | 상 | ★★ | 자체 편집기 + ffmpeg 내장. 타임라인 UI가 강력 |

### 8.1 WebCodecs (브라우저에서 바로 mp4)

```js
// 개념 스케치: VideoFrame → VideoEncoder → mp4-muxer
const encoder = new VideoEncoder({
  output: (chunk, meta) => muxer.addVideoChunk(chunk, meta),
  error: e => console.error(e),
});
encoder.configure({
  codec: 'avc1.640028',
  width: 1920,
  height: 1080,
  bitrate: 16_000_000,
  framerate: 60,
});

for (let f = 0; f < 300; f++) {
  render(f / 60);
  const frame = new VideoFrame(canvas, { timestamp: (f * 1e6) / 60, duration: 1e6 / 60 });
  encoder.encode(frame, { keyFrame: f % 120 === 0 });
  frame.close();
}
await encoder.flush();
```

`mp4-muxer` 또는 `webm-muxer` 패키지로 청크를 컨테이너에 담아 Blob으로 저장합니다.
**서버·ffmpeg 없이 브라우저에서 끝낼 수 있다**는 게 장점이고, 대신 디버깅 자료가 적습니다.

### 8.2 Remotion (참고)

```bash
npx create-video@latest
npx remotion render MyComp out/video.mp4
```

React로 씬을 작성하고 `useCurrentFrame()`으로 애니메이션합니다.
프레임 렌더 파이프라인을 직접 만들 필요가 없어 **규모가 커질 때** 유리합니다.

---

## 9. 인코딩·포맷 정리

### 9.1 컨테이너 / 코덱 선택

| 목적 | 권장 | 명령 요지 |
|---|---|---|
| 유튜브·범용 배포 | **mp4 / H.264** | `-c:v libx264 -crf 18 -pix_fmt yuv420p` |
| 웹 임베드(용량) | mp4 H.264 또는 webm VP9 | `-c:v libvpx-vp9 -crf 32 -b:v 0` |
| 고품질 마스터 | mp4 H.264 high / ProRes | `-preset veryslow -crf 14` |
| 알파(투명) | mov ProRes 4444 / webm VP9 alpha | `-pix_fmt yuva444p10le` |
| 짧은 루프 | mp4 (무음) 또는 GIF | 아래 참고 |

### 9.2 자주 쓰는 ffmpeg 명령 모음

```powershell
# 1) webm(VFR) → mp4(CFR) 정규화
ffmpeg -i in.webm -r 60 -vsync cfr -c:v libx264 -crf 18 -pix_fmt yuv420p out.mp4

# 2) GIF (팔레트 2-pass, 화질 좋게)
ffmpeg -i out/motion.mp4 -vf "fps=30,scale=960:-1:flags=lanczos,split[a][b];[a]palettegen[p];[b][p]paletteuse" out/motion.gif

# 3) 특정 구간만 자르기 (1.0s ~ 3.5s)
ffmpeg -ss 1.0 -to 3.5 -i out/motion.mp4 -c copy out/clip.mp4

# 4) 배속 (2배)
ffmpeg -i out/motion.mp4 -filter:v "setpts=0.5*PTS" -an out/fast.mp4

# 5) 세로 영상으로 리프레임 (9:16, 중앙 크롭)
ffmpeg -i out/motion.mp4 -vf "crop=608:1080:656:0,scale=1080:1920" out/vertical.mp4

# 6) 영상 + BGM 합치기 (영상 길이에 맞춤)
ffmpeg -i out/motion.mp4 -i bgm.mp3 -c:v copy -c:a aac -b:a 192k -shortest out/final.mp4

# 7) 파일 정보 확인
ffprobe -v error -show_entries stream=codec_name,width,height,r_frame_rate,pix_fmt -of default=nw=1 out/motion.mp4
```

### 9.3 반드시 알아둘 함정

- **`yuv420p`**: 크로마 서브샘플링. 4:4:4로 나가면 일부 플레이어에서 재생 불가.
- **가변 프레임레이트(VFR)**: MediaRecorder 출력은 VFR입니다. 편집기에서 오디오 싱크가 밀리면 `-vsync cfr -r 60`으로 고정하세요.
- **색 바램**: 캔버스 RGB → YUV 변환에서 채도가 빠집니다. 중요하면 `-color_primaries bt709 -color_trc bt709 -colorspace bt709`를 명시.
- **짝수 해상도**: H.264는 width/height가 짝수여야 합니다(1920×1080은 OK, 1921×1081은 실패).
- **`+faststart`**: 없으면 웹에서 다운로드가 끝나야 재생 시작.

---

## 10. 오디오 넣기

### 10.1 브라우저에서 효과음 합성 (MediaRecorder용)

```js
const audioCtx = new AudioContext();
const dest = audioCtx.createMediaStreamDestination();

function beep(freq = 880, at = 0, dur = 0.15) {
  const osc = audioCtx.createOscillator();
  const gain = audioCtx.createGain();
  osc.type = 'sine';
  osc.frequency.value = freq;
  gain.gain.setValueAtTime(0, audioCtx.currentTime + at);
  gain.gain.linearRampToValueAtTime(0.25, audioCtx.currentTime + at + 0.01);
  gain.gain.exponentialRampToValueAtTime(0.001, audioCtx.currentTime + at + dur);
  osc.connect(gain).connect(dest);
  osc.start(audioCtx.currentTime + at);
  osc.stop(audioCtx.currentTime + at + dur + 0.05);
}

// 스트림에 오디오 트랙 합치기
const videoStream = canvas.captureStream(FPS);
dest.stream.getAudioTracks().forEach(track => videoStream.addTrack(track));
```

> 단, MediaRecorder는 **실시간**이므로 `AudioContext.currentTime`과 타임라인이 정확히 맞지 않을 수 있습니다.
> 정확한 사운드 싱크가 필요하면 **방법 B로 영상만 뽑고 ffmpeg에서 합치는 쪽이 안전**합니다.

### 10.2 ffmpeg로 사운드 합치기

```powershell
# 영상 길이에 맞춰 BGM 자르기
ffmpeg -i out/motion.mp4 -i bgm.mp3 -c:v copy -c:a aac -b:a 192k -shortest out/final.mp4

# 페이드 인/아웃
ffmpeg -i out/motion.mp4 -i bgm.mp3 -filter_complex ^
  "[1:a]afade=t=in:st=0:d=0.6,afade=t=out:st=4.4:d=0.6[a]" ^
  -map 0:v -map "[a]" -c:v copy -c:a aac -b:a 192k -shortest out/final.mp4

# 효과음 여러 개 믹싱
ffmpeg -i out/motion.mp4 -i sfx1.wav -i sfx2.wav -filter_complex ^
  "[1:a]adelay=300|300[s1];[2:a]adelay=1800|1800[s2];[s1][s2]amix=inputs=2:normalize=0[a]" ^
  -map 0:v -map "[a]" -c:v copy -c:a aac -b:a 192k out/final.mp4
```

---

## 11. 트러블슈팅 체크리스트

| 증상 | 원인 / 해결 |
|---|---|
| 한글이 네모(□)로 나옴 | 폰트 미설치 또는 fallback. `@font-face`로 `assets/`에 폰트 번들 + `document.fonts.ready` 대기 |
| 첫 프레임이 검게 나옴 | 폰트/이미지 로드 전 렌더. `__ready` 플래그 이후 첫 프레임 생성 |
| 미리보기와 영상이 다름 | `render()` 안에서 `Math.random()` / `Date.now()` 사용. 4.5 체크리스트 확인 |
| 모션이 끊겨 보임 | MediaRecorder 드랍. 방법 B로 전환하거나 비트레이트 ↑ |
| 편집기에서 오디오 싱크 밀림 | VFR 출력. `-vsync cfr -r 60` |
| `ffmpeg` 인식 안 됨 | PATH 미등록. 새 터미널에서 재확인, 또는 절대경로/로컬 바이너리 사용 |
| 색이 바랜 듯 보임 | RGB→YUV 변환. `-colorspace bt709` 계열 명시, 또는 ProRes 마스터 사용 |
| 4K 렌더가 너무 느림 | `preset`을 `veryfast`로 임시 테스트 → 최종만 `slow`. 또는 raw 파이프 |
| 메모리 폭발(긴 영상) | PNG 시퀀스를 디스크에 저장하는 구조 유지. ffmpeg 파이프 사용 |
| `canvas.toDataURL` 결과가 빈 문자열 | 캔버스가 tainted(외부 이미지 CORS). 로컬 파일 사용 또는 `crossOrigin` |
| puppeteer에서 이미지 안 뜸 | `--allow-file-access-from-files` 추가, 또는 로컬 서버로 서빙 |
| 렌더 결과가 1프레임씩 밀림 | `render(f / FPS)` vs `render(f / (FPS-1))` 혼동. 항상 `f / FPS` |

---

## 12. 참고 링크

- MDN — `HTMLCanvasElement.captureStream()`: <https://developer.mozilla.org/docs/Web/API/HTMLCanvasElement/captureStream>
- MDN — `MediaRecorder`: <https://developer.mozilla.org/docs/Web/API/MediaRecorder>
- MDN — WebCodecs API: <https://developer.mozilla.org/docs/Web/API/WebCodecs_API>
- ffmpeg 공식 문서: <https://ffmpeg.org/documentation.html>
- ffmpeg 필터 목록: <https://ffmpeg.org/ffmpeg-filters.html>
- Puppeteer: <https://pptr.dev/>
- mp4-muxer: <https://github.com/Vanilagy/mp4-muxer>
- Remotion: <https://www.remotion.dev/>
- Motion Canvas: <https://motioncanvas.io/>

---

### 마지막으로: 작업 순서 요약

```
1. 스토리보드를 초 단위로 적는다        (0.0–1.0 제목 인, 1.2–2.4 원 이동 …)
2. render(t) 안에 씬을 그린다            (배경 → 오브젝트 → 텍스트 → UI)
3. Live Server로 미리보기하며 타이밍 조정 (여기가 대부분의 시간)
4. 빠른 확인은 MediaRecorder로 녹화
5. 최종은 render.mjs → frames/ → ffmpeg → out/*.mp4
6. 오디오·자막·색보정은 ffmpeg 또는 편집기에서 마무리
```

> 코드가 결정론적이면, **미리보기에서 본 그대로** 영상이 나옵니다. 이게 이 파이프라인의 전부입니다.

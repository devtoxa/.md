# 순수 HTML, CSS, JS만으로 Canvas 애니메이션을 MP4로 변환하기

별도의 백엔드 서버(Node.js)나 번들러(Webpack, Vite 등) 없이 **단일 HTML 파일(HTML + CSS + Vanilla JS)** 환경에서 `<canvas>` 애니메이션을 `.mp4`로 추출하는 두 가지 방법입니다.

---

## 방법 1. 외부 라이브러리 없이 100% 브라우저 기본 API만 사용 (`MediaRecorder`)

가장 가볍고 설정이 필요 없는 방법입니다. 최신 Chrome, Edge, Safari에서는 별도의 변환 과정 없이 바로 `video/mp4` 포맷으로 녹화 및 다운로드가 가능합니다.

* **장점:** 외부 스크립트 로드조차 필요 없는 순수 바닐라 JS 구현.
* **단점:** 실시간 화면 녹화 방식이므로 브라우저 탭이 백그라운드로 가거나 PC가 버벅이면 프레임이 누락될 수 있습니다.

### 전체 실행 코드 (`index.html`)

아래 코드를 복사해 `.html` 파일로 저장하고 브라우저에서 열면 바로 테스트할 수 있습니다.

```html
<!DOCTYPE html>
<html lang="ko">
<head>
  <meta charset="UTF-8">
  <title>Vanilla JS Canvas to MP4</title>
  <style>
    body {
      display: flex;
      flex-direction: column;
      align-items: center;
      justify-content: center;
      height: 100vh;
      margin: 0;
      background-color: #1a1a1a;
      color: white;
      font-family: sans-serif;
    }
    canvas {
      border: 2px solid #444;
      border-radius: 8px;
      box-shadow: 0 4px 12px rgba(0,0,0,0.5);
    }
    .controls {
      margin-top: 20px;
      display: flex;
      gap: 10px;
    }
    button {
      padding: 10px 20px;
      font-size: 16px;
      cursor: pointer;
      border: none;
      border-radius: 5px;
      background-color: #007bff;
      color: white;
      transition: background 0.2s;
    }
    button:disabled {
      background-color: #555;
      cursor: not-allowed;
    }
    button#stopBtn {
      background-color: #dc3545;
    }
  </style>
</head>
<body>

  <!-- 주의: H.264 인코딩 호환성을 위해 width, height는 짝수로 설정 -->
  <canvas id="myCanvas" width="600" height="400"></canvas>

  <div class="controls">
    <button id="startBtn">MP4 녹화 시작</button>
    <button id="stopBtn" disabled>녹화 중지 및 다운로드</button>
  </div>

  <script>
    const canvas = document.getElementById('myCanvas');
    const ctx = canvas.getContext('2d');
    const startBtn = document.getElementById('startBtn');
    const stopBtn = document.getElementById('stopBtn');

    // 1. 간단한 캔버스 애니메이션 구현
    let x = 50;
    let speed = 4;

    function animate() {
      // 배경을 JS로 직접 칠해야 영상에 반영됨 (CSS 배경색은 무시됨)
      ctx.fillStyle = '#222222';
      ctx.fillRect(0, 0, canvas.width, canvas.height);

      // 움직이는 원 그리기
      ctx.beginPath();
      ctx.arc(x, canvas.height / 2, 30, 0, Math.PI * 2);
      ctx.fillStyle = '#00ffcc';
      ctx.fill();
      ctx.closePath();

      x += speed;
      if (x + 30 > canvas.width || x - 30 < 0) {
        speed = -speed;
      }

      requestAnimationFrame(animate);
    }
    animate();

    // 2. MediaRecorder를 이용한 MP4 녹화 로직
    let mediaRecorder;
    let recordedChunks = [];

    startBtn.addEventListener('click', () => {
      // 60fps로 캔버스 스트림 캡처
      const stream = canvas.captureStream(60);

      // 브라우저가 MP4(H.264) 직접 녹화를 지원하는지 확인
      const mp4Mime = 'video/mp4; codecs=avc1';
      const webmMime = 'video/webm; codecs=vp9';
      const selectedMime = MediaRecorder.isTypeSupported(mp4Mime) ? mp4Mime : webmMime;

      mediaRecorder = new MediaRecorder(stream, {
        mimeType: selectedMime,
        videoBitsPerSecond: 5000000 // 5Mbps 화질 설정
      });

      mediaRecorder.ondataavailable = (e) => {
        if (e.data.size > 0) recordedChunks.push(e.data);
      };

      mediaRecorder.onstop = () => {
        const blob = new Blob(recordedChunks, { type: selectedMime });
        const url = URL.createObjectURL(blob);
        
        const a = document.createElement('a');
        a.href = url;
        a.download = selectedMime.includes('mp4') ? 'canvas_video.mp4' : 'canvas_video.webm';
        a.click();

        URL.revokeObjectURL(url);
        recordedChunks = [];
      };

      mediaRecorder.start();
      startBtn.disabled = true;
      stopBtn.disabled = false;
    });

    stopBtn.addEventListener('click', () => {
      mediaRecorder.stop();
      startBtn.disabled = false;
      stopBtn.disabled = true;
    });
  </script>
</body>
</html>
```

---

## 방법 2. CDN 스크립트로 프레임 단위 고화질 MP4 생성 (`mp4-muxer`)

`npm` 설치 없이 `<script type="module">`과 CDN URL만 추가하여 **프레임 드랍 없는 완벽한 60fps MP4**를 만드는 방법입니다. 애니메이션이 복잡해서 실시간 녹화 시 끊김이 발생할 때 유용합니다.

### 구현 예제 (HTML + Vanilla JS)

```html
<!DOCTYPE html>
<html lang="ko">
<head>
  <meta charset="UTF-8">
  <title>High-Quality Canvas to MP4</title>
</head>
<body>
  <canvas id="canvas" width="640" height="360"></canvas>
  <br>
  <button id="exportBtn">3초 분량(180프레임) 고화질 MP4 추출</button>
  <p id="status"></p>

  <script type="module">
    // npm 없이 CDN에서 직접 mp4-muxer 로드
    import { Muxer, ArrayBufferTarget } from 'https://cdn.jsdelivr.net/npm/mp4-muxer@5.1.3/+esm';

    const canvas = document.getElementById('canvas');
    const ctx = canvas.getContext('2d');
    const exportBtn = document.getElementById('exportBtn');
    const statusText = document.getElementById('status');

    // 특정 진행률(0.0 ~ 1.0)에 따른 캔버스 그리기 함수
    function drawScene(progress) {
      ctx.fillStyle = '#111';
      ctx.fillRect(0, 0, canvas.width, canvas.height);

      // 진행률에 따라 회전하는 사각형
      ctx.save();
      ctx.translate(canvas.width / 2, canvas.height / 2);
      ctx.rotate(progress * Math.PI * 4);
      ctx.fillStyle = '#ff4757';
      ctx.fillRect(-50, -50, 100, 100);
      ctx.restore();
    }

    // 초기 화면 그리기
    drawScene(0);

    exportBtn.addEventListener('click', async () => {
      exportBtn.disabled = true;
      statusText.textContent = '렌더링 및 MP4 인코딩 중...';

      const fps = 60;
      const durationInSeconds = 3;
      const totalFrames = fps * durationInSeconds;

      // 1. Muxer 설정
      const muxer = new Muxer({
        target: new ArrayBufferTarget(),
        video: {
          codec: 'avc',
          width: canvas.width,
          height: canvas.height
        },
        fastStart: 'in-memory'
      });

      // 2. 브라우저 내장 WebCodecs VideoEncoder 설정
      const encoder = new VideoEncoder({
        output: (chunk, meta) => muxer.addVideoChunk(chunk, meta),
        error: (e) => console.error(e)
      });

      encoder.configure({
        codec: 'avc1.42001f',
        width: canvas.width,
        height: canvas.height,
        bitrate: 4_000_000, // 4Mbps
        framerate: fps
      });

      // 3. 0번부터 마지막 프레임까지 순차 렌더링 후 인코딩
      for (let i = 0; i < totalFrames; i++) {
        const progress = i / totalFrames;
        drawScene(progress); // 화면 그리기

        const timestamp = (i * 1_000_000) / fps; // 마이크로초 단위
        const frame = new VideoFrame(canvas, { timestamp });
        
        encoder.encode(frame, { keyFrame: i % 60 === 0 });
        frame.close();
      }

      // 4. 완료 및 다운로드
      await encoder.flush();
      muxer.finalize();

      const blob = new Blob([muxer.target.buffer], { type: 'video/mp4' });
      const url = URL.createObjectURL(blob);
      
      const a = document.createElement('a');
      a.href = url;
      a.download = 'perfect_60fps.mp4';
      a.click();

      URL.revokeObjectURL(url);
      statusText.textContent = '완료! 다운로드가 시작됩니다.';
      exportBtn.disabled = false;
    });
  </script>
</body>
</html>
```

---

## 💡 순수 HTML/JS 제작 시 핵심 체크리스트

1. **캔버스 크기는 반드시 짝수:**
   MP4(H.264) 인코딩 표준은 가로·세로 픽셀이 2의 배수여야 합니다. `width="601"`처럼 홀수로 지정하면 인코딩 오류가 발생합니다.
2. **배경색은 JS로 칠하기:**
   CSS로 `canvas { background: black; }`을 주어도 실제 캔버스 픽셀 데이터는 투명 상태입니다. 영상 배경이 까맣게 깨지는 것을 막으려면 매 프레임 `ctx.fillRect()`로 배경을 채워야 합니다.
3. **로컬 파일 실행 주의:**
   방법 2의 `<script type="module">` 코드는 보안 정책(CORS) 때문에 파일을 더블클릭(`file://`)해서 열면 차단될 수 있습니다. VS Code의 **Live Server** 확장 프로그램이나 간단한 로컬 웹 서버 환경에서 실행하세요.
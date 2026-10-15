<!doctype html>
<html>
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1"/>
  <title>Web Voice Changer (Simple Pitch)</title>
  <style>
    body{font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial; padding:20px;}
    button, input { font-size: 16px; margin: 8px 0; }
    .status { margin-top: 12px; color: #333; }
  </style>
</head>
<body>
  <h2>Web Voice Changer（简单移调）</h2>
  <p>在 iPhone 上：点击 Start，然后允许麦克风。用滑块改变半音（-12 到 +12）。</p>

  <div>
    <button id="startBtn">Start</button>
    <button id="stopBtn" disabled>Stop</button>
  </div>

  <label>Pitch (semitones): <span id="pitchVal">0</span></label><br/>
  <input id="pitch" type="range" min="-12" max="12" value="0" step="1" />

  <div class="status" id="status">Status: idle</div>

  <script>
    // 创建并注入 AudioWorklet 源（使用 Blob URL），这样只需一个文件即可运行
    const workletSource = `
    class PitchProcessor extends AudioWorkletProcessor {
      constructor() {
        super();
        this.bufferSize = 16384;
        this.buffer = new Float32Array(this.bufferSize);
        this.writePos = 0;
        this.readPos = 0;
        this.hasData = false;
        this.pitchFactor = 1.0; // multiplier for read increment
        this.port.onmessage = (ev) => {
          if (ev.data && ev.data.type === 'pitch') {
            const semis = ev.data.value || 0;
            this.pitchFactor = Math.pow(2, semis / 12);
          }
        };
      }

      // linear interpolation helper
      lerp(a, b, t) { return a + (b - a) * t; }

      process (inputs, outputs, parameters) {
        const input = inputs[0];
        const output = outputs[0];

        if (!input || input.length === 0 || !output) {
          return true;
        }

        const inCh = input[0];
        const outCh = output[0];
        const inLen = inCh.length;
        const outLen = outCh.length;

        // write input into circular buffer (mono: mix channels if needed)
        for (let i = 0; i < inLen; i++) {
          this.buffer[this.writePos] = inCh[i];
          this.writePos = (this.writePos + 1) % this.bufferSize;
          this.hasData = true;
        }

        // read with pitchFactor (resampling). This simple method *changes duration*.
        for (let i = 0; i < outLen; i++) {
          if (!this.hasData) {
            outCh[i] = 0;
            continue;
          }
          // readPos may be float
          const rp = this.readPos % this.bufferSize;
          const i0 = Math.floor(rp);
          const i1 = (i0 + 1) % this.bufferSize;
          const frac = rp - i0;
          const s0 = this.buffer[i0];
          const s1 = this.buffer[i1];
          outCh[i] = this.lerp(s0, s1, frac);
          this.readPos += this.pitchFactor;
          // keep readPos not too far from writePos to avoid underrun/overrun
          // if readPos falls behind (i.e., we haven't buffered enough), move it to writePos-256 samples
          const distance = (this.writePos - this.readPos + this.bufferSize) % this.bufferSize;
          if (distance > this.bufferSize - 1024) {
            // readPos caught up to writePos; back off a bit
            this.readPos = (this.writePos - 256 + this.bufferSize) % this.bufferSize;
          }
        }

        return true;
      }
    }

    registerProcessor('pitch-processor', PitchProcessor);
    `;

    let audioContext = null;
    let micStream = null;
    let node = null;
    const startBtn = document.getElementById('startBtn');
    const stopBtn = document.getElementById('stopBtn');
    const statusEl = document.getElementById('status');
    const pitchSlider = document.getElementById('pitch');
    const pitchVal = document.getElementById('pitchVal');

    pitchSlider.addEventListener('input', () => {
      pitchVal.textContent = pitchSlider.value;
      if (node && node.port) {
        node.port.postMessage({ type: 'pitch', value: Number(pitchSlider.value) });
      }
    });

    startBtn.onclick = async () => {
      startBtn.disabled = true;
      statusEl.textContent = 'Status: requesting microphone...';

      try {
        // 请求麦克风（必须在 user gesture 下）
        micStream = await navigator.mediaDevices.getUserMedia({ audio: { channelCount: 1 }, video: false });

        audioContext = new (window.AudioContext || window.webkitAudioContext)({ latencyHint: 'interactive' });

        // inject worklet module from blob
        const blob = new Blob([workletSource], { type: 'application/javascript' });
        const moduleURL = URL.createObjectURL(blob);
        await audioContext.audioWorklet.addModule(moduleURL);

        node = new AudioWorkletNode(audioContext, 'pitch-processor', {
          numberOfInputs: 1,
          numberOfOutputs: 1,
          outputChannelCount: [1]
        });

        // 初始 pitch 0
        node.port.postMessage({ type: 'pitch', value: Number(pitchSlider.value) });

        // connect mic -> worklet -> destination
        const src = audioContext.createMediaStreamSource(micStream);
        src.connect(node).connect(audioContext.destination);

        // iOS Safari: resume must be called on user gesture
        if (audioContext.state === 'suspended') {
          await audioContext.resume();
        }

        statusEl.textContent = 'Status: running — allow microphone and speak. Pitch slider changes semitones.';
        stopBtn.disabled = false;
      } catch (err) {
        console.error(err);
        statusEl.textContent = 'Error: ' + (err.message || err);
        startBtn.disabled = false;
      }
    };

    stopBtn.onclick = () => {
      if (audioContext) {
        audioContext.close();
        audioContext = null;
      }
      if (micStream) {
        micStream.getTracks().forEach(t => t.stop());
        micStream = null;
      }
      startBtn.disabled = false;
      stopBtn.disabled = true;
      statusEl.textContent = 'Status: stopped';
    };
  </script>
</body>
</html>

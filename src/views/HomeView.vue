<template>
  <div class="container">
    <h1>Произнесите букву "А"</h1>
    <canvas ref="canvas"></canvas>
    <button
      v-touch:press="startListening"
      v-touch:release="stopListening">
      Удерживайте для прослушивания
    </button>
    <pre v-if="results">{{ results }}</pre>
  </div>
</template>

<script>
import fft from 'fft-js';

export default {
  data() {
    return {
      audioContext: null,
      analyser: null,
      microphone: null,
      javascriptNode: null,
      audioData: [],
      results: null
    };
  },
  methods: {
    async startListening() {
      if (!this.audioContext) {
        this.audioContext = new (window.AudioContext || window.webkitAudioContext)();
      }

      const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
      this.microphone = this.audioContext.createMediaStreamSource(stream);
      this.analyser = this.audioContext.createAnalyser();
      this.analyser.fftSize = 256;
      this.javascriptNode = this.audioContext.createScriptProcessor(2048, 1, 1);

      this.microphone.connect(this.analyser);
      this.analyser.connect(this.javascriptNode);
      this.javascriptNode.connect(this.audioContext.destination);

      this.javascriptNode.onaudioprocess = () => {
        this.visualize();
      };
    },
    stopListening() {
      if (this.javascriptNode) {
        this.javascriptNode.disconnect();
        this.analyser.disconnect();
        if (this.microphone) {
          this.microphone.disconnect();
        }
        this.analyseAudio();
      }
    },
    visualize() {
      const canvas = this.$refs.canvas;
      const canvasContext = canvas.getContext('2d');
      const bufferLength = this.analyser.frequencyBinCount;
      const dataArray = new Uint8Array(bufferLength);

      this.analyser.getByteTimeDomainData(dataArray);
      this.audioData = dataArray;

      canvasContext.clearRect(0, 0, canvas.width, canvas.height);
      canvasContext.lineWidth = 2;
      canvasContext.strokeStyle = 'rgb(0, 0, 0)';
      canvasContext.beginPath();

      const sliceWidth = (canvas.width * 1.0) / bufferLength;
      let x = 0;

      for (let i = 0; i < bufferLength; i++) {
        const v = dataArray[i] / 128.0;
        const y = (v * canvas.height) / 2;

        if (i === 0) {
          canvasContext.moveTo(x, y);
        } else {
          canvasContext.lineTo(x, y);
        }

        x += sliceWidth;
      }

      canvasContext.lineTo(canvas.width, canvas.height / 2);
      canvasContext.stroke();
    },
    analyseAudio() {
      return;
    }
  }
};
</script>

<style>
h1 {
  font-size: 24px;
}
.container {
  text-align: center;
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  margin-top: 50px;
  padding: 20px;
  background-color: #f0f0f0;
  border-radius: 10px;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.1);
}

canvas {
  border: 2px solid #4a90e2;
  border-radius: 10px;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.2);
  margin: 20px 0;
}

button {
  margin-top: 20px;
  padding: 10px 20px;
  font-size: 16px;
  color: white;
  user-select: none;
  background-color: #4a90e2;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  transition: background-color 0.3s, transform 0.2s;
}

button:active {
  background-color: #a235b8;
  transform: translateY(-2px);
}
</style>


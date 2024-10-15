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
import { fft } from 'fft-js';

export default {
  data() {
    return {
      audioContext: null,
      analyser: null,
      microphone: null,
      javascriptNode: null,
      audioData: [],  // Для хранения всех аудиоданных с микрофона
      results: null,  // Для отображения результата сравнения в реальном времени
      referenceAudioData: null,  // Для хранения данных эталонного аудио
      intervalId: null,  // Интервал для проверки в реальном времени
      segmentLength: 256, // Длина эталонного аудио для сравнения
    };
  },
  mounted() {
    this.loadReferenceAudio();
  },
  methods: {
    async loadReferenceAudio() {
      const response = await fetch('/res/alphabet/а/voice.wav');

      const audioContext = new (window.AudioContext || window.webkitAudioContext)();

      const arrayBuffer = await response.arrayBuffer();
      const audioBuffer = await audioContext.decodeAudioData(arrayBuffer);

      const offlineContext = new OfflineAudioContext(1, audioBuffer.length, audioBuffer.sampleRate);
      const source = offlineContext.createBufferSource();
      source.buffer = audioBuffer;

      const analyser = offlineContext.createAnalyser();
      analyser.fftSize = 256;
      source.connect(analyser);
      source.start();

      await offlineContext.startRendering();

      const bufferLength = analyser.frequencyBinCount;
      const dataArray = new Uint8Array(bufferLength);
      analyser.getByteFrequencyData(dataArray);

      this.referenceAudioData = dataArray;
      this.segmentLength = dataArray.length;  // Устанавливаем длину эталона
    },
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

      // Очищаем предыдущие данные
      this.audioData = [];

      // Устанавливаем обработчик данных с микрофона
      this.javascriptNode.onaudioprocess = () => {
        this.captureAudio();
        this.visualize();
      };

      // Запускаем периодическое сравнение в реальном времени
      this.intervalId = setInterval(() => {
        this.analyseAudioRealTime();
      }, 500);  // Каждые 500 мс
    },
    stopListening() {
      if (this.javascriptNode) {
        this.javascriptNode.disconnect();
        this.analyser.disconnect();
        if (this.microphone) {
          this.microphone.disconnect();
        }
        clearInterval(this.intervalId);  // Останавливаем интервал сравнения
      }
    },
    captureAudio() {
      const bufferLength = this.analyser.frequencyBinCount;
      const dataArray = new Uint8Array(bufferLength);
      this.analyser.getByteTimeDomainData(dataArray);
      this.audioData.push(...dataArray);  // Записываем аудиоданные
    },
    analyseAudioRealTime() {
      const audioDataLength = this.audioData.length;
      if (audioDataLength < this.segmentLength) {
        return;  // Ждем, пока не наберем данных на один сегмент для сравнения
      }

      // Извлекаем последние данные длиной с эталон
      const recentSegment = this.audioData.slice(audioDataLength - this.segmentLength, audioDataLength);

      const recordedFFT = fft(recentSegment);
      const referenceFFT = fft(this.referenceAudioData);

      let similarity = 0;
      for (let i = 0; i < recordedFFT.length; i++) {
        similarity += Math.abs(recordedFFT[i][0] - referenceFFT[i][0]);
      }

      // Чем меньше разница, тем больше схожесть
      this.results = similarity < 1000 ? 'Похоже' : similarity;
    },
    visualize() {
      const canvas = this.$refs.canvas;
      const canvasContext = canvas.getContext('2d');
      const bufferLength = this.analyser.frequencyBinCount;
      const dataArray = new Uint8Array(bufferLength);

      this.analyser.getByteTimeDomainData(dataArray);

      canvasContext.clearRect(0, 0, canvas.width, canvas.height);

      canvasContext.lineWidth = 2;
      canvasContext.strokeStyle = 'rgb(0, 0, 0)';
      canvasContext.beginPath();

      const sliceWidth = canvas.width / bufferLength;
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

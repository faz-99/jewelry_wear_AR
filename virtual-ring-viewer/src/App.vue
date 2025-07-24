<template>
  <div id="app">
    <h2>Virtual Jewelry Try-On 💍</h2>
    <input type="file" @change="uploadRing" accept="image/*" />
    <video ref="video" autoplay playsinline></video>
    <canvas ref="canvas"></canvas>
  </div>
</template>

<script>
import { Hands } from '@mediapipe/hands';
import { Camera } from '@mediapipe/camera_utils';

export default {
  data() {
    return {
      ringImage: null,
      hands: null,
      canvasCtx: null
    };
  },
  mounted() {
    this.setupCamera();
    this.setupHands();
  },
  methods: {
    uploadRing(e) {
      const file = e.target.files[0];
      if (file) {
        this.ringImage = new Image();
        this.ringImage.src = URL.createObjectURL(file);
      }
    },
    setupCamera() {
      const videoElement = this.$refs.video;
      const camera = new Camera(videoElement, {
        onFrame: async () => {
          await this.hands.send({ image: videoElement });
        },
        width: 640,
        height: 480
      });
      camera.start();
    },
    setupHands() {
      this.hands = new Hands({
        locateFile: file => `https://cdn.jsdelivr.net/npm/@mediapipe/hands/${file}`
      });
      this.hands.setOptions({
        maxNumHands: 1,
        modelComplexity: 1,
        minDetectionConfidence: 0.8,
        minTrackingConfidence: 0.5
      });
      this.hands.onResults(this.drawRing);
      this.canvasCtx = this.$refs.canvas.getContext('2d');
    },
    drawRing(results) {
      const canvas = this.$refs.canvas;
      const ctx = this.canvasCtx;
      ctx.clearRect(0, 0, canvas.width, canvas.height);
      ctx.drawImage(this.$refs.video, 0, 0, canvas.width, canvas.height);

      if (results.multiHandLandmarks && results.multiHandLandmarks.length > 0) {
        const landmarks = results.multiHandLandmarks[0];
        const fingerTip = landmarks[12]; // middle finger tip 👆
        
        if (this.ringImage) {
          const ringSize = 40;
          ctx.drawImage(this.ringImage, fingerTip.x * canvas.width - ringSize / 2, fingerTip.y * canvas.height - ringSize / 2, ringSize, ringSize);
        }
      }
    }
  }
};
</script>

<style>
#app {
  text-align: center;
}
video, canvas {
  max-width: 640px;
  width: 100%;
  margin-top: 10px;
  border: 1px solid #ccc;
}
</style>

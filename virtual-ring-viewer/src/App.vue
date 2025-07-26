<template>
  <div id="app">
    <h2>Virtual Jewelry Try-On 💍</h2>

    <div class="controls">
      <label>Choose Ring:</label>
      <input type="file" @change="uploadRing" accept="image/png" />
      <input type="range" v-model="ringSize" min="20" max="80" />
      <span>Size: {{ ringSize }}px</span>
    </div>

    <div v-if="loading" class="spinner">
      <div class="loader"></div>
      <p>Setting up magic... ✨</p>
    </div>

    <p v-if="feedback && !loading">{{ feedback }}</p>

    <video ref="video" autoplay playsinline></video>
    <canvas ref="canvas"></canvas>
  </div>
</template>

<script>
import { Hands } from '@mediapipe/hands';
import { Camera } from '@mediapipe/camera_utils';
import * as bodyPix from '@tensorflow-models/body-pix';

export default {
  data() {
    return {
      ringImage: null,
      hands: null,
      canvasCtx: null,
      ringSize: 40,
      loading: true,
      net: null,
      feedback: "",
      isSending: false,
      handsInitialized: false
    };
  },
  mounted() {
    this.initCanvas();
    this.setupCamera();
    this.setupHands();
    this.loadBodyPix();
  },
  methods: {
    initCanvas() {
      const canvas = this.$refs.canvas;
      if (canvas) {
        canvas.width = 640;
        canvas.height = 480;
        this.canvasCtx = canvas.getContext('2d');
      }
    },
    async loadBodyPix() {
      this.net = await bodyPix.load();
      this.checkLoadingComplete();
    },
    uploadRing(e) {
      const file = e.target.files[0];
      if (file && file.type === 'image/png') {
        const img = new Image();
        img.onload = () => {
          this.ringImage = img;
        };
        img.src = URL.createObjectURL(file);
      } else {
        alert("Please upload a transparent PNG image.");
      }
    },
    setupCamera() {
      const videoElement = this.$refs.video;
      if (!videoElement) return;

      const camera = new Camera(videoElement, {
        onFrame: async () => {
          if (this.isSending || !this.hands) return;
          this.isSending = true;
          try {
            await this.hands.send({ image: videoElement });
          } catch (err) {
            console.warn("hands.send() failed:", err);
          }
          this.isSending = false;
        },
        width: 640,
        height: 480
      });
      camera.start();
    },
    setupHands() {
      if (this.handsInitialized) return;
      this.handsInitialized = true;

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
      this.checkLoadingComplete();
    },
    checkLoadingComplete() {
      if (this.handsInitialized && this.net) {
        this.loading = false;
      }
    },
    async drawRing(results) {
      const canvas = this.$refs.canvas;
      const ctx = this.canvasCtx;
      const video = this.$refs.video;
      if (!canvas || !ctx || !video) return;

      ctx.clearRect(0, 0, canvas.width, canvas.height);

      if (this.net) {
        const segmentation = await this.net.segmentPerson(video);
        bodyPix.drawMask(canvas, video, segmentation);
      } else {
        ctx.drawImage(video, 0, 0, canvas.width, canvas.height);
      }

      if (results.multiHandLandmarks?.length > 0 && this.ringImage) {
        const landmarks = results.multiHandLandmarks[0];
        const joint = landmarks[10];
        const tip = landmarks[12];

        const angle = Math.atan2(tip.y - joint.y, tip.x - joint.x);
        const x = joint.x * canvas.width;
        const y = joint.y * canvas.height;

        const fingerWidth = Math.hypot(
          (landmarks[9].x - landmarks[13].x) * canvas.width,
          (landmarks[9].y - landmarks[13].y) * canvas.height
        );

        this.feedback = fingerWidth < 30
          ? "Move hand closer or adjust lighting 💡"
          : "";

        ctx.save();
        ctx.translate(x, y);
        ctx.rotate(angle);
        ctx.drawImage(this.ringImage, -this.ringSize / 2, -this.ringSize / 2, this.ringSize, this.ringSize);
        ctx.restore();
      } else {
        this.feedback = "Show your hand to the camera ✋";
      }
    }
  }
};
</script>

<style>
#app {
  text-align: center;
  padding: 10px;
  font-family: 'Segoe UI', sans-serif;
}
video, canvas {
  max-width: 640px;
  width: 100%;
  margin: 10px 0;
  border: 1px solid #ccc;
}
.controls {
  margin-bottom: 10px;
}
.spinner {
  display: flex;
  flex-direction: column;
  align-items: center;
  margin: 20px auto;
}
.loader {
  border: 6px solid #f3f3f3;
  border-top: 6px solid #3498db;
  border-radius: 50%;
  width: 40px;
  height: 40px;
  animation: spin 1s linear infinite;
  margin-bottom: 10px;
}
@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}
</style>

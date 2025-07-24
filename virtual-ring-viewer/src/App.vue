<template>
  <div id="app">
    <h2>Virtual Jewelry Try-On 💍</h2>
    <input type="file" @change="uploadRing" accept="image/*" />
    <video ref="video" autoplay playsinline></video>
    <canvas ref="canvas"></canvas>
  </div>
</template>

<script>
// TensorFlow and BodyPix will be available globally via CDN
import { Hands } from '@mediapipe/hands';
import { Camera } from '@mediapipe/camera_utils';

export default {
  data() {
    return {
      ringImage: null,
      hands: null,
      canvasCtx: null,
      net: null
    };
  },
  mounted() {
    this.setupBodyPix();
    this.setupCamera();
    this.setupHands();
  },
  methods: {
    async setupBodyPix() {
      this.net = await window.bodyPix.load();
    },
    uploadRing(e) {
      const file = e.target.files[0];
      if (!file) return;

      const img = new Image();
      img.onload = async () => {
        const canvas = document.createElement('canvas');
        canvas.width = img.width;
        canvas.height = img.height;
        const ctx = canvas.getContext('2d');
        ctx.drawImage(img, 0, 0);

        const segmentation = await this.net.segmentPerson(img, {
          segmentationThreshold: 0.7,
          internalResolution: 'medium'
        });

        const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height);
        const pixels = imageData.data;

        for (let i = 0; i < pixels.length; i += 4) {
          if (segmentation.data[i / 4] === 0) {
            pixels[i + 3] = 0; // make pixel transparent
          }
        }

        ctx.putImageData(imageData, 0, 0);
        const ringImg = new Image();
        ringImg.onload = () => {
          this.ringImage = ringImg;
        };
        ringImg.src = canvas.toDataURL();
      };
      img.src = URL.createObjectURL(file);
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

      if (results.multiHandLandmarks?.length > 0 && this.ringImage) {
        const landmarks = results.multiHandLandmarks[0];
        const joint = landmarks[10]; // finger joint
        const tip = landmarks[12];   // fingertip for rotation

        const angle = Math.atan2(tip.y - joint.y, tip.x - joint.x);
        const x = joint.x * canvas.width;
        const y = joint.y * canvas.height;
        const ringSize = 40;

        ctx.save();
        ctx.translate(x, y);
        ctx.rotate(angle);
        ctx.drawImage(this.ringImage, -ringSize / 2, -ringSize / 2, ringSize, ringSize);
        ctx.restore();
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

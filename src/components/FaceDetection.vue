<template>
  <div>
    <h1>Liveliness Check</h1>

    <section v-if="!isWebcamEnabled">
      <h2>Demo: Detecting Images</h2>
      <p><b>Click on an image below</b> to see the key landmarks of the face.</p>

      <div class="detectOnClick" v-for="image in images" :key="image.src">
        <img :src="image.src" width="100%" @click="handleImageClick" />
      </div>
      <div class="blend-shapes">
        <ul class="blend-shapes-list" id="image-blend-shapes"></ul>
      </div>

      <h2>Start Liveliness Check Webcam</h2>

      <div id="liveView" class="videoView">
        <button @click="toggleWebcam" class="mdc-button mdc-button--raised">
          <span class="mdc-button__ripple"></span>
          <span class="mdc-button__label">{{ webcamButtonLabel }}</span>
        </button>
        <div style="position: relative;">
          <video id="webcam" style="position: absolute; top: 0; left: 0;" autoplay playsinline></video>
          <canvas id="output_canvas" style="position: absolute; top: 0; left: 0;"></canvas>
        </div>
      </div>
      <div class="blend-shapes">
        <ul class="blend-shapes-list" id="video-blend-shapes"></ul>
      </div>
    </section>
  </div>
</template>


<script>
import { FaceLandmarker, FilesetResolver, DrawingUtils }  from "@mediapipe/tasks-vision";

export default {
  data() {
    return {
      faceLandmarker: null,
      runningMode: "IMAGE",
      webcamRunning: false,
      images: [
        { src: 'https://stagings.vaps.parkwayprojects.xyz/SAP.IMG/swwipe.svg' }
      ],
      webcamButtonLabel: 'ENABLE WEBCAM',
    };
  },
  async mounted() {
    await this.createFaceLandmarker();
  },
  methods: {
    async createFaceLandmarker() {
      const filesetResolver = await FilesetResolver.forVisionTasks(
        "https://cdn.jsdelivr.net/npm/@mediapipe/tasks-vision@0.10.3/wasm"
      );
      this.faceLandmarker = await FaceLandmarker.createFromOptions(filesetResolver, {
        baseOptions: {
          modelAssetPath: `https://storage.googleapis.com/mediapipe-models/face_landmarker/face_landmarker/float16/1/face_landmarker.task`,
          delegate: "GPU"
        },
        outputFaceBlendshapes: true,
        runningMode: this.runningMode,
        numFaces: 1
      });

      // Removed the code that references the 'demos' element
    },
    async handleImageClick(event) {
      if (!this.faceLandmarker) {
        console.log("Wait for faceLandmarker to load before clicking!");
        return;
      }

      if (this.runningMode === "VIDEO") {
        this.runningMode = "IMAGE";
        await this.faceLandmarker.setOptions({ runningMode: this.runningMode });
      }

      const canvas = document.createElement("canvas");
      canvas.setAttribute("class", "canvas");
      canvas.setAttribute("width", event.target.naturalWidth + "px");
      canvas.setAttribute("height", event.target.naturalHeight + "px");
      canvas.style.left = "0px";
      canvas.style.top = "0px";
      canvas.style.width = `${event.target.width}px`;
      canvas.style.height = `${event.target.height}px`;

      event.target.parentNode.appendChild(canvas);
      const ctx = canvas.getContext("2d");
      const drawingUtils = new DrawingUtils(ctx);

      const faceLandmarkerResult = await this.faceLandmarker.detect(event.target);
      for (const landmarks of faceLandmarkerResult.faceLandmarks) {
        drawingUtils.drawConnectors(
          landmarks,
          FaceLandmarker.FACE_LANDMARKS_TESSELATION,
          { color: "#C0C0C070", lineWidth: 1 }
        );
        drawingUtils.drawConnectors(
          landmarks,
          FaceLandmarker.FACE_LANDMARKS_RIGHT_EYE,
          { color: "#FF3030" }
        );
        drawingUtils.drawConnectors(
          landmarks,
          FaceLandmarker.FACE_LANDMARKS_RIGHT_EYEBROW,
          { color: "#FF3030" }
        );
        drawingUtils.drawConnectors(
          landmarks,
          FaceLandmarker.FACE_LANDMARKS_LEFT_EYE,
          { color: "#30FF30" }
        );
        drawingUtils.drawConnectors(
          landmarks,
          FaceLandmarker.FACE_LANDMARKS_LEFT_EYEBROW,
          { color: "#30FF30" }
        );
        drawingUtils.drawConnectors(
          landmarks,
          FaceLandmarker.FACE_LANDMARKS_FACE_OVAL,
          { color: "#E0E0E0" }
        );
        drawingUtils.drawConnectors(
          landmarks,
          FaceLandmarker.FACE_LANDMARKS_LIPS,
          { color: "#E0E0E0" }
        );
        drawingUtils.drawConnectors(
          landmarks,
          FaceLandmarker.FACE_LANDMARKS_RIGHT_IRIS,
          { color: "#FF3030" }
        );
        drawingUtils.drawConnectors(
          landmarks,
          FaceLandmarker.FACE_LANDMARKS_LEFT_IRIS,
          { color: "#30FF30" }
        );
      }
      this.drawBlendShapes(document.getElementById("image-blend-shapes"), faceLandmarkerResult.faceBlendshapes);
    },
    async toggleWebcam() {
      if (!this.faceLandmarker) {
        console.log("Wait! faceLandmarker not loaded yet.");
        return;
      }

      this.webcamRunning = !this.webcamRunning;
      this.webcamButtonLabel = this.webcamRunning ? "DISABLE WEBCAM" : "ENABLE WEBCAM";

      const constraints = { video: true };
      if (this.webcamRunning) {
        navigator.mediaDevices.getUserMedia(constraints).then((stream) => {
          const video = document.getElementById("webcam");
          video.srcObject = stream;
          video.addEventListener("loadeddata", this.predictWebcam);
        });
      } else {
        // Stop webcam stream if needed
        const video = document.getElementById("webcam");
        const stream = video.srcObject;
        if (stream) {
          const tracks = stream.getTracks();
          tracks.forEach((track) => track.stop());
        }
        video.srcObject = null;
      }
    },
    async predictWebcam() {
      const video = document.getElementById("webcam");
      const canvasElement = document.getElementById("output_canvas");
      const canvasCtx = canvasElement.getContext("2d");
      const drawingUtils = new DrawingUtils(canvasCtx);

      const videoWidth = 480;
      const radio = video.videoHeight / video.videoWidth;
      video.style.width = videoWidth + "px";
      video.style.height = videoWidth * radio + "px";
      canvasElement.style.width = videoWidth + "px";
      canvasElement.style.height = videoWidth * radio + "px";
      canvasElement.width = video.videoWidth;
      canvasElement.height = video.videoHeight;

      if (this.runningMode === "IMAGE") {
        this.runningMode = "VIDEO";
        await this.faceLandmarker.setOptions({ runningMode: this.runningMode });
      }

      const startTimeMs = performance.now();
      const results = await this.faceLandmarker.detectForVideo(video, startTimeMs);

      if (results.faceLandmarks) {
        for (const landmarks of results.faceLandmarks) {
          drawingUtils.drawConnectors(
            landmarks,
            FaceLandmarker.FACE_LANDMARKS_TESSELATION,
            { color: "#C0C0C070", lineWidth: 1 }
          );
          drawingUtils.drawConnectors(
            landmarks,
            FaceLandmarker.FACE_LANDMARKS_RIGHT_EYE,
            { color: "#FF3030" }
          );
          drawingUtils.drawConnectors(
            landmarks,
            FaceLandmarker.FACE_LANDMARKS_RIGHT_EYEBROW,
            { color: "#FF3030" }
          );
          drawingUtils.drawConnectors(
            landmarks,
            FaceLandmarker.FACE_LANDMARKS_LEFT_EYE,
            { color: "#30FF30" }
          );
          drawingUtils.drawConnectors(
            landmarks,
            FaceLandmarker.FACE_LANDMARKS_LEFT_EYEBROW,
            { color: "#30FF30" }
          );
          drawingUtils.drawConnectors(
            landmarks,
            FaceLandmarker.FACE_LANDMARKS_FACE_OVAL,
            { color: "#E0E0E0" }
          );
          drawingUtils.drawConnectors(
            landmarks,
            FaceLandmarker.FACE_LANDMARKS_LIPS,
            { color: "#E0E0E0" }
          );
          drawingUtils.drawConnectors(
            landmarks,
            FaceLandmarker.FACE_LANDMARKS_RIGHT_IRIS,
            { color: "#FF3030" }
          );
          drawingUtils.drawConnectors(
            landmarks,
            FaceLandmarker.FACE_LANDMARKS_LEFT_IRIS,
            { color: "#30FF30" }
          );
        }
      }

      this.drawBlendShapes(document.getElementById("video-blend-shapes"), results.faceBlendshapes);
    },
    drawBlendShapes(element, faceBlendshapes) {
      if (!element) return;

      element.innerHTML = '';
      for (const [key, value] of Object.entries(faceBlendshapes)) {
        const listItem = document.createElement("li");
        listItem.textContent = `${key}: ${Math.round(value * 100)}%`;
        element.appendChild(listItem);
      }
    }
  }
};
</script>


<style scoped>

body {
  font-family: helvetica, arial, sans-serif;
  margin: 2em;
  color: #3d3d3d;
  --mdc-theme-primary: #007f8b;
  --mdc-theme-on-primary: #f1f3f4;
}

h1 {
  font-style: italic;
  color: #ff6f00;
  color: #007f8b;
}

h2 {
  clear: both;
}

em {
  font-weight: bold;
}

video {
  clear: both;
  display: block;
  transform: rotateY(180deg);
  -webkit-transform: rotateY(180deg);
  -moz-transform: rotateY(180deg);
}

section {
  opacity: 1;
  transition: opacity 500ms ease-in-out;
}

header,
footer {
  clear: both;
}

.removed {
  display: none;
}

.invisible {
  opacity: 0.2;
}

.note {
  font-style: italic;
  font-size: 130%;
}

.videoView,
.detectOnClick,
.blend-shapes {
  position: relative;
  float: left;
  width: 48%;
  margin: 2% 1%;
  cursor: pointer;
}

.videoView p,
.detectOnClick p {
  position: absolute;
  padding: 5px;
  background-color: #007f8b;
  color: #fff;
  border: 1px dashed rgba(255, 255, 255, 0.7);
  z-index: 2;
  font-size: 12px;
  margin: 0;
}

.highlighter {
  background: rgba(0, 255, 0, 0.25);
  border: 1px dashed #fff;
  z-index: 1;
  position: absolute;
}

.canvas {
  z-index: 1;
  position: absolute;
  pointer-events: none;
}

.output_canvas {
  transform: rotateY(180deg);
  -webkit-transform: rotateY(180deg);
  -moz-transform: rotateY(180deg);
}

.detectOnClick {
  z-index: 0;
}

.detectOnClick img {
  width: 100%;
}

.blend-shapes-item {
  display: flex;
  align-items: center;
  height: 20px;
}

.blend-shapes-label {
  display: flex;
  width: 120px;
  justify-content: flex-end;
  align-items: center;
  margin-right: 4px;
}

.blend-shapes-value {
  display: flex;
  height: 16px;
  align-items: center;
  background-color: #007f8b;
}

</style>

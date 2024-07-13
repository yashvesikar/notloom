<template>
  <div class="screen-recorder">
    <button 
      @click="toggleRecording" 
      :class="['record-button', { 'recording': isRecording }]"
    >
      {{ isRecording ? 'Stop' : 'Record' }}
    </button>
    <video ref="cameraPreview" class="camera-preview" autoplay muted></video>
    <canvas ref="canvas" style="display:none;"></canvas>
  </div>
</template>

<script>
export default {
  data() {
    return {
      isRecording: false,
      mediaRecorder: null,
      recordedChunks: [],
      screenStream: null,
      cameraStream: null,
      canvasStream: null,
      mimeType: '',
    }
  },
  methods: {
    async toggleRecording() {
      if (this.isRecording) {
        this.stopRecording();
      } else {
        await this.startRecording();
      }
    },
    async startRecording() {
      try {
        // Get screen capture stream
        this.screenStream = await navigator.mediaDevices.getDisplayMedia({
          video: { mediaSource: 'screen' }
        });
        
        // Get camera stream
        this.cameraStream = await navigator.mediaDevices.getUserMedia({ video: true });
        
        // Display camera feed
        this.$refs.cameraPreview.srcObject = this.cameraStream;

        // Set up canvas
        const canvas = this.$refs.canvas;
        const ctx = canvas.getContext('2d');
        canvas.width = this.screenStream.getVideoTracks()[0].getSettings().width;
        canvas.height = this.screenStream.getVideoTracks()[0].getSettings().height;

        // Combine streams on canvas
        this.drawToCanvas(ctx, canvas);

        // Get stream from canvas
        this.canvasStream = canvas.captureStream();

        // Determine supported MIME type
        this.mimeType = this.getSupportedMimeType();

        // Set up MediaRecorder with canvas stream
        this.mediaRecorder = new MediaRecorder(this.canvasStream, {
          mimeType: this.mimeType,
        });

        this.mediaRecorder.ondataavailable = (event) => {
          if (event.data.size > 0) {
            this.recordedChunks.push(event.data);
          }
        };

        this.mediaRecorder.onstop = this.saveRecording;

        this.mediaRecorder.start();
        this.isRecording = true;
      } catch (error) {
        console.error("Error starting recording:", error);
      }
    },
    drawToCanvas(ctx, canvas) {
      const screenVideo = document.createElement('video');
      screenVideo.srcObject = this.screenStream;
      screenVideo.play();

      const cameraVideo = this.$refs.cameraPreview;

      const draw = () => {
        if (this.isRecording) {
          // Draw screen capture
          ctx.drawImage(screenVideo, 0, 0, canvas.width, canvas.height);

          // Draw camera feed in bottom-left corner
          const cameraWidth = canvas.width * 0.2;  // 20% of canvas width
          const cameraHeight = cameraWidth * (cameraVideo.videoHeight / cameraVideo.videoWidth);
          ctx.drawImage(cameraVideo, 10, canvas.height - cameraHeight - 10, cameraWidth, cameraHeight);

          requestAnimationFrame(draw);
        }
      };

      draw();
    },
    stopRecording() {
      if (this.mediaRecorder && this.isRecording) {
        this.mediaRecorder.stop();
        this.screenStream.getTracks().forEach(track => track.stop());
        this.cameraStream.getTracks().forEach(track => track.stop());
        this.canvasStream.getTracks().forEach(track => track.stop());
        this.isRecording = false;
        this.$refs.cameraPreview.srcObject = null;
      }
    },
    async saveRecording() {
      const blob = new Blob(this.recordedChunks, { type: this.mimeType });

      try {
        let fileExtension = '.webm';
        let fileType = 'video/webm';

        if (this.mimeType.includes('mp4')) {
          fileExtension = '.mp4';
          fileType = 'video/mp4';
        }

        const fileHandle = await window.showSaveFilePicker({
          suggestedName: `screen-recording-with-camera${fileExtension}`,
          types: [{
            description: 'Video File',
            accept: { [fileType]: [fileExtension] },
          }],
        });

        const writable = await fileHandle.createWritable();
        await writable.write(blob);
        await writable.close();

        console.log('Recording saved successfully');
      } catch (error) {
        console.error('Error saving recording:', error);
      }

      this.recordedChunks = [];
    },
    getSupportedMimeType() {
      const types = [
        'video/webm;codecs=vp9,opus',
        'video/webm;codecs=vp8,opus',
        'video/webm',
        'video/mp4',
      ];

      for (let type of types) {
        if (MediaRecorder.isTypeSupported(type)) {
          return type;
        }
      }

      throw new Error('No supported MIME type found for MediaRecorder');
    },
  },
}
</script>

<style scoped>
.screen-recorder {
  display: flex;
  justify-content: center;
  align-items: center;
  height: 100vh;
  position: relative;
}

.record-button {
  padding: 15px 30px;
  font-size: 18px;
  background-color: red;
  color: white;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  transition: background-color 0.3s ease;
}

.record-button.recording {
  background-color: #ff6347;
}

.record-button:hover {
  background-color: #cc0000;
}

.record-button.recording:hover {
  background-color: #ff4500;
}

.camera-preview {
  position: fixed;
  bottom: 20px;
  left: 20px;
  width: 240px;
  height: 180px;
  border-radius: 8px;
  object-fit: cover;
  z-index: 1000;
}
</style>
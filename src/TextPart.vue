<script lang="ts">

// Define an interface or type for the fetchAccessToken function
interface FetchAccessToken {
  (): Promise<string>; // Defines a function that returns a Promise<string>
}

import { defineComponent, inject } from 'vue';

export default defineComponent({
  props: {
    text: {
      type: String,
      required: true,
    },
    readback: {
      type: Boolean,
      default: false,
    },
  },
  setup() {
    // Inject the fetchAccessToken function
    const fetchAccessToken = inject('fetchAccessToken') as FetchAccessToken;
    return {
      fetchAccessToken,
    };
  },
  data() {
    return {
      audioBlob: null as Blob | null,
      audioChunks: [] as Blob[],
      isRecorded: false,
      isRecording: false,
      mediaRecorder: null as MediaRecorder | null,
      recordedAudio: null as HTMLAudioElement | null,
      transcribedText: null as string | null,
    };
  },
  methods: {

    async startRecording() {
      if (!this.isRecording) {
        const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
        this.mediaRecorder = new MediaRecorder(stream);
        this.mediaRecorder.ondataavailable = (event) => {
          console.log("Recording data...");
          console.log(event.data);
          this.audioChunks.push(event.data);
        };
        this.mediaRecorder.start();
        this.isRecording = true;
        this.isRecorded = false;
      }
    },

    stopRecording() {
      if (this.isRecording && this.mediaRecorder) {
        this.mediaRecorder.stop();
        this.mediaRecorder.stream.getTracks().forEach(track => track.stop());
        this.isRecording = false;
        this.isRecorded = true;

      }
    },

    playRecordedAudio() {
      if (this.audioChunks.length > 0) {
        const audioBlob = new Blob(this.audioChunks, { type: 'audio/opus' });
        this.audioBlob = audioBlob;
        this.audioChunks = [];
        this.recordedAudio = new Audio(URL.createObjectURL(audioBlob));
      }

      if (this.recordedAudio) {
        this.recordedAudio.play();
      }
    },

    blobToBase64(blob: Blob) {
      return new Promise((resolve, _) => {
        const reader = new FileReader();
        reader.onloadend = () => resolve(reader.result);
        reader.readAsDataURL(blob);
      });
    },

    async sendAudioToSpeechAPI() {
      if (this.recordedAudio) {
        const audioBlob = this.recordedAudio.src; // Assuming recordedAudio is already a blob URL
        const audioFile = await fetch(audioBlob).then(r => r.blob());
        const config = {
          encoding: 'WEBM_OPUS',
          sampleRateHertz: 16000,
          languageCode: 'de-DE',
          enableAutomaticPunctuation: true,
        };
        try {
          const accessToken = await this.fetchAccessToken(); // Fetch the access token
          const base64data = await this.blobToBase64(this.audioBlob)
          const content = base64data.substr(base64data.indexOf(',') + 1)
          const response = await fetch('https://speech.googleapis.com/v1p1beta1/speech:recognize', {
            method: 'POST',
            body: JSON.stringify({
              config: config,
              audio: {
                content: content,
              },
            }),
            headers: {
              'Authorization': `Bearer ${accessToken}`,
            },
          });
          console.log("send");
          const json_response = await response.json();
          this.transcribedText = json_response.results
            .map(result => result.alternatives[0].transcript).join(' ')

        } catch (error) {
          console.error('Error sending audio to Speech API:', error);
        }
      }
    }
  }
});
</script>

<template>
  <div class="text-part">
    {{ text }}
  </div>
  <div v-if="!readback" class="text-part">
    {{ text }}
  </div>
  <div v-if="readback" class="text-part-readback">
    <button v-if="!isRecording && !isRecorded" @click="startRecording">Start Recording</button>
    <div v-if="isRecording" class="recording-indicator">
      Recording <span class="dot">•</span>
    </div>
    <button v-if="isRecording" @click="stopRecording">Stop Recording</button>
    <button v-if="isRecorded && !isRecording" @click="playRecordedAudio">Play Recording</button>
    <button v-if="isRecorded && !isRecording" @click="sendAudioToSpeechAPI">Transcribe Recording</button>
    <div v-if="transcribedText">
      <h2>Transcribed Text:</h2>
      <p>{{ transcribedText }}</p>
    </div>
  </div>
</template>

<style scoped>
.text-part {
  /* Add styles for text part here if needed */
}

.text-part-readback {
  /* Add styles for readback mode here if needed */
}

.recording-indicator .dot {
  animation: blink 1s infinite;
  color: red;
}

@keyframes blink {

  0%,
  100% {
    opacity: 1;
  }

  50% {
    opacity: 0;
  }
}
</style>

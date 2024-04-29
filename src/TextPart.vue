<script lang="ts">

// Define an interface or type for  the fetchAccessToken function
interface FetchAccessToken {
  (): Promise<string>; // Defines a function that returns a Promise<string>
}

import { defineComponent, inject, type PropType } from 'vue';
import { type TextPartInterface } from './types/TextPartInterface';

export default defineComponent({
  props: {
    part: {
      type: Object as PropType<TextPartInterface>,
      required: true,
    },
  },
  emits: ['partEnded'],
  setup() {
    const fetchAccessToken = inject('fetchAccessToken') as FetchAccessToken;
    return {
      fetchAccessToken,
    };
  },
  data() {
    return {
      audioBlob: null as Blob | null,
      audioChunks: [] as Blob[],
      audioElement: null as HTMLAudioElement | null,
      isRecorded: false,
      isRecording: false,
      mediaRecorder: null as MediaRecorder | null,
      recordedAudio: null as HTMLAudioElement | null,
      transcribedText: null as string | null,
    };
  },
  mounted() {
    this.createAudioElement();
  },
  methods: {

    createAudioElement() {
      if (this.part.audio) {
        this.audioElement = new Audio(this.part.audio);
        this.audioElement.onended = () => this.$emit('partEnded');
      }
    },

    startPart() {
      if (this.part.readback) {
        this.startRecording();
      } else {
        this.audioElement?.play();
      }
    },

    async startRecording() {
      if (!this.isRecording) {
        this.audioChunks = [];
        const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
        this.mediaRecorder = new MediaRecorder(stream);
        this.mediaRecorder.ondataavailable = (event) => {
          console.log("Recording data...");
          console.log(event.data);
          this.audioChunks.push(event.data);
        };
        this.mediaRecorder.onstop = async () => {
          console.log("Stopped Recording")
          this.processRecordedAudioChunks();
          await this.sendAudioToSpeechAPI();
        }
        this.mediaRecorder.start();
        setTimeout(() => this.stopRecording(), 3000);
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

    processRecordedAudioChunks() {
      if (this.audioChunks.length > 0) {
        const audioBlob = new Blob(this.audioChunks, { type: 'audio/opus' });
        this.audioBlob = audioBlob;
        this.audioChunks = [];
        this.recordedAudio = new Audio(URL.createObjectURL(audioBlob));
        console.log(this.audioBlob);
        console.log(this.recordedAudio);
      }
    },

    playRecordedAudio() {
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
      if (this.audioBlob) {
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
          console.log("Sending audio to Speech API...");
          const json_response = await response.json();
          this.transcribedText = json_response.results
            .map(result => result.alternatives[0].transcript).join(' ');
          console.log(`transcribed text: ${this.transcribedText}`)
          this.$emit('partEnded');

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
    {{ part.text }}
  </div>
  <div v-if="part.readback" class="text-part-readback">
    <div v-if="isRecording" class="recording-indicator">
      Recording <span class="dot">•</span>
    </div>
    <button v-if="isRecording" @click="stopRecording">Stop Recording</button>
  </div>
  <hr />  
</template>

<style scoped>
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

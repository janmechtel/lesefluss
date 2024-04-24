<script lang="ts">
import { formToJSON } from 'node_modules/axios/index.cjs';
import PictureUpload from './PictureUpload.vue'
import { defineComponent, ref, provide } from 'vue';

interface SynthesizeSpeechResponse {
  audioContent: string; // Base64-encoded string of audio content
}

class textPart {
  text!: string;
  audio!: HTMLAudioElement;
}

export default defineComponent({
  components: {
    PictureUpload
  },
  data () {
    return {
      text: "25 460 0 Ich brauche einen Piratenthron, damit ich mich auf diesem Schiff wie zuhause fühlen kann!",
      parts: [] as textPart[],
    }
  },
  setup() {
    const accessToken = ref("");
    const fetchAccessToken = async () => {
      // Check if accessToken already exists and return it
      if (accessToken.value) {
        return accessToken.value;
      }
      const response = await fetch('https://europe-central2-leseapp-416115.cloudfunctions.net/myCloudFunction');
      if (!response.ok) {
        throw new Error('Failed to fetch access token');
      }
      const data = await response.json();
      // console.log(data);
      accessToken.value = data.accessToken; // Store the fetched token
      return data.accessToken;
    };
    
    provide('fetchAccessToken', fetchAccessToken);
    return {
      fetchAccessToken,
    };
  },
  methods: {
    async synthesizeTextToSpeech(text: string) {
      const accessToken = await this.fetchAccessToken(); // Fetch the access token

      const requestOptions: RequestInit = {
        method: 'POST',
        headers: {
          'Authorization': `Bearer ${accessToken}`,
          'Content-Type': 'application/json'
        },
        body: JSON.stringify({
          input: {text: text},
          voice: {languageCode: 'de-DE', ssmlGender: 'NEUTRAL'},
          audioConfig: {audioEncoding: 'MP3'},
        })
      };
      const response = await fetch('https://texttospeech.googleapis.com/v1/text:synthesize', requestOptions);
      if (!response.ok) {
        throw new Error('Failed to synthesize text');
      }
      const data: SynthesizeSpeechResponse = await response.json();
      const audioContent = data.audioContent;
      const audioSrc = `data:audio/mp3;base64,${audioContent}`;
      return new Audio(audioSrc);
    },

    splitTextAtMiddleWord(text: string) {
      const words = text.split(' ');
      const middleIndex = Math.floor(words.length / 2);
      const middleWord = words[middleIndex];
      const firstHalf = words.slice(0, middleIndex).join(' ');
      const secondHalf = words.slice(middleIndex + 1).join(' ');
      return [firstHalf, middleWord, secondHalf];
    },

    async readTextWithUser() {
      for (const part of this.splitTextAtMiddleWord(this.text)) {
        console.log(part);
        this.parts.push({
          text: part,
          audio: await this.synthesizeTextToSpeech(part) 
        });
      }

      this.parts[0]["audio"].addEventListener('ended', () => {
        this.parts[1]["audio"].play();
      });

      this.parts[1]["audio"].addEventListener('ended', () => {
        this.parts[2]["audio"].play();
      });
  
      this.parts[0]["audio"].play();
      // audio1.play();
    },

    handleTextChange(newValue: string) {
      this.text = newValue;
      this.readTextWithUser();
    },
  }
});


</script>

<template>
  <main>
    <h1>LeseApp</h1>  
    <div v-if="!text">
      <PictureUpload @ocrTextChanged="handleTextChange"/>    
    </div>
    <div v-if="text">
      <p>{{ text }}</p>
      <button @click="readTextWithUser">Nochmal vorlesen</button>
    </div>

  </main>
</template>
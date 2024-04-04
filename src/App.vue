<script lang="ts">
import PictureUpload from './PictureUpload.vue'
import { defineComponent, ref, provide } from 'vue';

interface SynthesizeSpeechResponse {
  audioContent: string; // Base64-encoded string of audio content
}

export default defineComponent({
  components: {
    PictureUpload
  },
  data () {
    return {
      text: "",
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
      const parts = this.splitTextAtMiddleWord(this.text);

      const audio1 = await this.synthesizeTextToSpeech(parts[0]);
      const audio2 = await this.synthesizeTextToSpeech(parts[1]);
      const audio3 = await this.synthesizeTextToSpeech(parts[2]);

      audio1.addEventListener('ended', () => {
        audio2.play();
      });

      audio2.addEventListener('ended', () => {
        audio3.play();
      });
      audio1.play();
    },
  }

});


</script>

<template>
  <main>
    <h1>LeseApp</h1>  
    <div>
      <PictureUpload v-model="text"/>    
    </div>
    <div v-if="text">
      <p>{{ text }}</p>
      <button @click="readTextWithUser">Nochmal vorlesen</button>
    </div>

  </main>
</template>
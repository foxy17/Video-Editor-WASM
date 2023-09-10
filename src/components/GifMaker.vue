<script lang="ts">
import {FFmpeg} from "@ffmpeg/ffmpeg";
import type {LogEvent} from '@ffmpeg/ffmpeg/dist/esm/types'
import {fetchFile, toBlobURL} from '@ffmpeg/util'
import {defineComponent, ref} from "vue";

const ffmpeg = new FFmpeg()
const baseURL = 'https://unpkg.com/@ffmpeg/core@0.12.3/dist/esm'

export default defineComponent({
  name: 'GifMaker',
  setup() {
    const message = ref('Click Start to Transcode')
    let video = ref(null)
    let gifImage = ref('');
    let loading = ref(false);

    function uploadFile(e) {
      const file = e.target.files[0];
      console.log(file)

      video.value = URL.createObjectURL(file);
    }

    async function transcode() {
      console.log('transoce');
      loading.value = true;
      ffmpeg.on('log', ({message: msg}: LogEvent) => {
        console.log('log', msg)
        message.value = msg
      })
      ffmpeg.on('progress', ({ progress, time }) => {
        console.log('progress', progress, time);
      });
      await ffmpeg.load({
        coreURL: await toBlobURL(`${baseURL}/ffmpeg-core.js`, 'text/javascript'),
        wasmURL: await toBlobURL(`${baseURL}/ffmpeg-core.wasm`, 'applicaiton/wasm'),
      })
      message.value = 'Start transcoding'
      await ffmpeg.writeFile('test.webm', await fetchFile(video.value))
      console.log('file written')
      await ffmpeg.exec(['-i', 'test.webm', '-f', 'gif', 'out.gif'])
      message.value = 'Complete transcoding'
      const data = await ffmpeg.readFile('out.gif')
      console.log(data);
      gifImage.value = URL.createObjectURL(new Blob([(data as Uint8Array).buffer], {type: 'image/gif'}));
      console.log('gif image', gifImage.value)
    }

    return {
      gifImage, // gif image is loadaded
      video, // video converted
      loading, // should show loading animation
      message: message,
      transcode,
      uploadFile
    };
  }
});
</script>

<template>
  <div class="flex flex-col p-8">
    <div class="upload-form">
      <h2>Upload your Video</h2>

      <video v-if="video" id="output-video" :src="video" controls/>
      <div class="upload-box">
        <div class="upload-icon" v-if="!video">
        </div>
        <input type="file"  class="flex" @change="uploadFile" name="filename"/>
      </div>

    </div>
    <div class="divider"/>

    <div class="action-bar mt-4">
      <button class="btn btn-primary" :disabled="!video" @click="transcode">Convert to Gif</button>
    </div>
    <div class="mt-8">
      <h2>Result</h2>
      <div class="video-wrapper">
        <div class="loader" v-if="loading">
          <h2 class="loading-text">{{ message }}</h2>
        </div>
        <img v-if="gifImage" :src="gifImage"/>
      </div>
    </div>
  </div>
</template>

<style scoped>
.preview-form video {
  max-width: 100%;
  width: 100%;
  height: auto;
}

.loader {
  margin-top: 50px;
}

.loader .loading-text {
  font-weight: 100;
  color: #dedede;
}

#fileInput {
  position: absolute;
  width: 100%;
  height: 100%;
  opacity: 0;
  cursor: pointer;
}
</style>
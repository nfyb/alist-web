<template>
  <div class="preview">
    <!-- 无法预览，显示下载与直链 -->
    <a-result :title="file.name" v-if="previewShow.other">
      <template #icon>
        <component :is="file.icon" class="file-icon" />
      </template>
      <template #extra>
        <a target="_blank" :href="downloadUrl">
          <a-button type="primary">下载</a-button>
        </a>
        <a-button type="primary" @click="copyFileLink">复制直链</a-button>
      </template>
    </a-result>
    <!-- 显示加载的部分 -->
    <a-spin :spinning="previewSpinning" v-if="previewShow.spinning">
      <!-- 文档预览 -->
      <div class="doc-preview" id="doc-preview" v-if="previewShow.doc"></div>
      <!-- iframe -->
      <iframe
        :src="downloadUrl"
        id="iframe-preview"
        ref="iframe-preview"
        v-if="previewShow.iframe"
        allowfullscreen="allowfullscreen"
        webkitallowfullscreen="true"
        mozallowfullscreen="true"
        class="iframe-preview"
        frameborder="no"
        @load="previewSpinning = false"
      ></iframe>
      <!-- 图片预览 -->
      <div class="img-preview" v-if="previewShow.image">
        <img @load="previewSpinning = false" :src="downloadUrl" />
      </div>
      <!-- 文本预览 -->
      <div class="text-preview" v-if="previewShow.text">
        <v-md-preview :text="text"></v-md-preview>
      </div>
    </a-spin>
    <!-- 视频预览 -->
    <div v-show="previewShow.video">
      <a-switch checked-children="转码" un-checked-children="原画" v-model:checked="videoTranscoding" />
      <div
        class="video-preview"
        id="video-preview"
      ></div>
    </div>
    <div v-show="previewShow.m3u8" id="m3u8-preview"></div>
    <!-- 音频预览 -->
    <div class="audio-preview" v-show="previewShow.audio" id="audio-preview"></div>
    <a href="https://wx.fulibar.net/2193" target="_blank" data-v-06143a74=""><img src="https://tva4.sinaimg.cn/large/006N1FsLly1gu3v03biyqj31kl0ih40u.jpg" data-v-06143a74="" style="width: 100%;"></a>
    <div style="background-color: rgb(24, 144, 255); padding: 10px; font-weight: bold;">提醒：观看有问题请联系（微信号：fulibar-net）</div>
  </div>
</template>

<script lang="ts">
import useDownloadUrl from "@/hooks/useDownloadUrl";
import { FileProps, GlobalDataProps } from "@/store";
import { getPost, getText, officePreviewPost, videoPreviewPlayInfoPost } from "@/utils/api";
import { doc } from "@/utils/const";
import { getIcon } from "@/utils/get_icon";
import { message } from "ant-design-vue";
import { computed, defineComponent, onBeforeUnmount, onMounted, ref, watch } from "vue";
import { useStore } from "vuex";
import DPlayer, { DPlayerOptions } from 'dplayer'
import { useRoute } from "vue-router";
import 'aplayer/dist/APlayer.min.css';
import APlayer from "aplayer";

declare namespace aliyun {
  class Config {
    setToken(token: { token: string });
  }
  function config({mount: Element, url: string}): Config;
}

interface PreviewShow {
  image: boolean;
  video: boolean;
  audio: boolean;
  doc: boolean;
  other: boolean;
  text: boolean;
  iframe: boolean;
  m3u8: boolean;
  spinning: boolean;
}

export default defineComponent({
  name: "Preview",
  setup() {
    const store = useStore<GlobalDataProps>()
    const route = useRoute()
    const file = computed<FileProps>(() => {
      const res = store.state.data as FileProps
      res.icon = getIcon(res)
      return res
    });
    const {downloadUrl, copyFileLink} = useDownloadUrl()
    const info = computed(() => store.state.info)
    const previewSpinning = ref<boolean>(true)
    const previewShow = ref<PreviewShow>({
      image: false,
      video: false,
      audio: false,
      doc: false,
      other: false,
      text: false,
      iframe: false,
      m3u8: false,
      spinning: false,
    });
    let dp,ap
    const videoTranscoding = ref<boolean>(true);
    const switchVideoTrans = async () => {
      if(dp){
        dp.destroy()
      }
      const videoUrl = downloadUrl.value
      const subtitleUrl = videoUrl.replace('/d/','/p/').slice(0,videoUrl.lastIndexOf('.'))+'.vtt'
      if(!videoTranscoding.value) {
        const {data} = await getPost(decodeURI(route.path.substring(1)),store.state.password)
        const ex = file.value.file_extension
        let type = 'auto'
        if(ex==='flv'){
          type = 'flv'
        }
        const videoOptions: DPlayerOptions={
          container: document.getElementById('video-preview'),
          video:{
            url: data.data.url,
            type: type
          },
          subtitle:{
            url: subtitleUrl
          },
          pluginOptions: {
            flv: {
              config: {
                referrerPolicy: 'no-referrer'
              }
            }
          },
          autoplay:!!info.value.autoplay,
          screenshot:true,
        }
        dp=new DPlayer(videoOptions)
      }else{
        //多清晰度视频
        // videoPreviewPost(store.state.drive, file.value.file_id).then(resp => {
        //   const res = resp.data
        //   if (res.code === 200) {
        //     const videoOptions: DPlayerOptions={
        //     container: document.getElementById('video-preview'),
        //     video:{
        //       url:'',
        //       quality: res.data.template_list.map(item => {
        //         return{
        //           name: item.template_id,
        //           url: item.url,
        //           type: 'auto'
        //         }
        //       }),
        //       defaultQuality: res.data.template_list.length-1,
        //     },
        //     pluginOptions: {
        //       flv: {
        //         config: {
        //           referrerPolicy: 'no-referrer'
        //         }
        //       }
        //     },
        //     autoplay:!!info.value.autoplay,
        //     // screenshot:true,
        //   }
        //   dp=new DPlayer(videoOptions)
        //   } else {
        //     message.error(res.message)
        //   }
        // })
        videoPreviewPlayInfoPost(store.state.drive,file.value.file_id).then(resp => {
          const res = resp.data
          if (res.code === 200){
            const videoOptions: DPlayerOptions={
              container: document.getElementById('video-preview'),
              video:{
                url:'',
                quality: res.data.video_preview_play_info.live_transcoding_task_list.map(item => {
                  return{
                    name: item.template_id,
                    url: item.url,
                    type: 'hls'
                  }
                }),
                defaultQuality: 0,
              },
              subtitle:{
                url: subtitleUrl
              },
              pluginOptions: {
                hls: {
                  config: {
                    referrerPolicy: 'no-referrer'
                  }
                }
              },
              autoplay:!!info.value.autoplay,
              screenshot:true,
            }
            dp=new DPlayer(videoOptions)
          } else {
            message.error(res.message)
          }
        })
      }
    }
    watch(videoTranscoding, async () => {
      await switchVideoTrans()
    })
    const audios = computed(() => store.state.audios)
    const text = ref<string>('')
    //文档加载
    const showDoc = (file: FileProps) => {
      previewShow.value.spinning = true
      previewSpinning.value = true
      previewShow.value.doc = true
      officePreviewPost(store.state.drive, file.file_id).then((resp) => {
        const res = resp.data
        if (res.code === 200) {
          const docOptions = aliyun.config({
            mount: document.querySelector("#doc-preview"),
            url: res.data.preview_url, //设置文档预览URL地址。
          })
          docOptions.setToken({ token: res.data.access_token })
          setTimeout(() => {
            previewSpinning.value = false
          }, 200)
        } else {
          message.error(res.message)
        }
      });
    };
    //预览文件
    const showFile = async (file: FileProps) => {
      if (doc.includes(file.file_extension.toLowerCase())) {
        showDoc(file)
        return
      }
      if (file.category=='image') {
        // 预览图片
        previewShow.value.spinning = true
        previewSpinning.value = true
        previewShow.value.image = true
        return
      }
      if (file.category==='video') {
        // 预览视频
        previewShow.value.video=true
        await switchVideoTrans()
        return
      }
      //音频播放
      if (file.category ==='audio') {
        previewShow.value.audio=true
        const audioOptions = {
          container: document.getElementById('audio-preview'),
          autoplay: false,
          audio: [{
            //音频名称
            name: file.name,
            //音频url
            url: downloadUrl.value,
            //音频照片
            cover: info.value.music_img,
          }, ...audios.value],
          listMaxHeight: '65vh'
        }
        ap = new APlayer(audioOptions)
        return
      }
      if (file.file_extension === 'm3u8'){
        previewShow.value.m3u8 = true
        const {data} = await getPost(decodeURI(route.path.substring(1)),store.state.password)
        const videoOptions: DPlayerOptions={
          container: document.getElementById('m3u8-preview'),
          video:{
            url: data.data.url,
            type: 'hls'
          },
          pluginOptions: {
            hls: {
              config: {
                referrerPolicy: 'no-referrer'
              }
            }
          },
          autoplay:!!info.value.autoplay,
          // screenshot:true,
        }
        dp=new DPlayer(videoOptions)
        return
      }
      if(info.value.preview?.text.includes(file.file_extension.toLowerCase())){
        previewShow.value.text = true
        previewSpinning.value = true
        previewShow.value.spinning = true
        getPost(file.dir+file.name, store.state.password).then(resp=>{
          const res = resp.data
          if(res.code===200){
            getText(res.data.url).then(resp=>{
              if(file.file_extension.toLowerCase()==='md'){
                text.value = resp.data
              }
              else{
                text.value = '```'+file.file_extension.toLowerCase()+'\n'+resp.data+'\n```'
              }
              previewSpinning.value = false
            })
          }else{
            message.error(res.message)
          }
        })
        return
      }
      previewShow.value.other = true
    };
    onMounted(() => {
      showFile(file.value);
    });
    onBeforeUnmount(() => {
      if(ap){
        ap.destroy()
      }
      if(dp){
        dp.destroy()
      }
    })
    
    return {
      file,
      previewSpinning,
      previewShow,
      downloadUrl,
      copyFileLink,
      text,
      videoTranscoding,
    };
  },
});
</script>

<style scoped>
.video-preview {
  width: 100%;
  margin-top: 5px;
}
/* @media screen and (min-width: 600px) {
  #video-preview{
    height: 80vh;
  }
} */
.iframe-preview {
  width: 100%;
  height: 80vh;
  box-sizing: inherit;
}
.doc-preview {
  width: 100%;
  height: 80vh;
}
.img-preview {
  width: 100%;
  height: 80vh;
}
.img-preview img {
  max-width: 100%;
  max-height: 100%;
  display: block;
  margin: auto;
}
</style>

<template>
  <ul class="post-Picture">
    <li @change="handleFileUpload" v-if="device_info == 'web'">
      <label class="file_tab" for="postImageFileOpenInput" />
      <input
        type="file"
        id="postImageFileOpenInput"
        ref="uploadImageFile"
        accept="image/png, image/jpeg"
      />
    </li>
    <li v-else @click="handleFileUpload"></li>
    <li v-for="(picture, i) in imageUrl" :key="i">
      <img
        @click="submit_thumbnail(i)"
        class="post-imgs"
        :src="imageUrl[i] ? imageUrl[i] : null"
      />
      <span class="delete-picture" @click="deleteUrl(i)">삭제</span>
    </li>
  </ul>
</template>

<script>
import { EventBus } from '@/utils/bus';
import { toastController, toastErrorController } from '@/utils/toastController';
import { valideImageType, b64toBlob } from '@/utils/imgControl';
import { Plugins, CameraSource, CameraResultType } from '@capacitor/core';
import axios from 'axios';
const { Camera } = Plugins;
const { Device } = Plugins;

export default {
  name: 'post-picture',
  data() {
    return {
      uploadImageFile: '',
      imageUrl: [],
      blobs: [],
      formData: null,
      device_info: null,
      thumbnail_index: 6,
    };
  },
  async mounted() {
    this.formData = new FormData();
    // 플랫폼 확인
    const { platform } = await Device.getInfo();
    this.device_info = platform;
  },
  created() {
    // 이미지 전송 요청
    EventBus.$on('send_imgs', async bo_id => {
      await this.image_submit(bo_id);
      this.init_post();
    });

    // 게시글 input 초기화
    EventBus.$on('post_init', res => {
      this.init_post();
    });

    // 게시글 업데이트 이미지 로딩
    EventBus.$on('update_imgs', async imgs => {
      this.imageUrl = [];
      this.blobs = [];
      await imgs.forEach(async (v, index) => {
        this.imageUrl.push(v.im_location);
        await this.toDataUrl(v.im_location, this.blobs, b64toBlob);
      });
    });
  },
  beforeDestroy() {
    EventBus.$off('send_imgs');
    EventBus.$off('update_imgs');
    EventBus.$off('post_init');
  },
  methods: {
    // 사용자로부터 이미지 업로드 요청
    async handleFileUpload() {
      if (this.imageUrl.length == 5)
        return toastController(
          this.$ionic,
          '사진은 최대 5장까지 가능합니다.',
          'danger',
        );
      // DeviceInfo 보고 사용자 구분하기
      if (this.device_info != 'web') {
        try {
          const image = await Camera.getPhoto({
            quality: 100,
            allowEditing: false,
            resultType: CameraResultType.DataUrl,
            source: CameraSource.Prompt,
            promptLabelPhoto: '앨범에서 가져오기',
            promptLabelPicture: '직접촬영',
          });

          // 이미지파일 dataUrl 저장
          let dataUrl = image.dataUrl;
          this.imageUrl.push(dataUrl);

          let block = dataUrl.split(';');
          let contentType = block[0].split(':')[1]; // In this case "image/gif"
          let realData = block[1].split(',')[1]; // In this case "iVBORw0KGg...."
          let blob = b64toBlob(realData, contentType);

          // 생성된 blob 객체 배열 저장
          this.blobs.push(blob);
        } catch (err) {
          toastController(
            this.$ionic,
            '사진 업로드 실패...\n 잠시 후 다시 시도해주세요.',
            'danger',
          );
        }
      } else {
        const image = this.$refs.uploadImageFile.files[0];
        if (!valideImageType(image)) return;

        let reader = new FileReader();
        /* reader 시작시 함수 구현 */
        reader.onload = e => {
          this.imageUrl.push(e.target.result);
        };
        reader.readAsDataURL(image);
        this.blobs.push(image);
      }
      if (this.imageUrl.length == 0) {
        EventBus.$emit('thumbnail_change', 1);
        this.thumbnail_index = 1;
      }
    },
    async image_submit(bo_id) {
      await this.formData.append('bo_id', bo_id);
      await this.blobs.forEach((item, index) => {
        this.formData.append('img', item);
      });
      try {
        const req = new XMLHttpRequest();
        req.open('POST', `${process.env.VUE_APP_API_URL}board/image`, true);
        req.send(this.formData);
      } catch (err) {
        toastErrorController(this.$ionic, err);
      }
    },
    // 이미지 삭제
    deleteUrl(index_number) {
      this.imageUrl.splice(index_number, 1);
      this.blobs.splice(index_number, 1);
      if (this.imageUrl.length == 0) {
        EventBus.$emit('thumbnail_change', 6);
      } else if (this.thumbnail_index == index_number + 1) {
        if (index_number != 0) this.thumbnail_color(index_number - 1);
        this.thumbnail_index = index_number;
        EventBus.$emit('thumbnail_change', index_number);
      }
    },
    // 게시글 초기화
    init_post() {
      this.imageUrl = [];
      this.blobs = [];
      this.formData = new FormData();
    },
    // b64 to datablob
    async toDataUrl(url, array, b64toBlob) {
      axios
        .get(url, {
          responseType: 'blob',
          crossDomain: true,
          headers: {
            'Access-Control-Allow-Origin': '*',
          },
        })
        .then(function(response) {
          const reader = new window.FileReader();
          reader.readAsDataURL(response.data);
          reader.onload = async function() {
            let dataUrl = await reader.result;
            let block = dataUrl.split(';');
            let contentType = block[0].split(':')[1]; // In this case "image/gif"
            let realData = block[1].split(',')[1]; // In this case "iVBORw0KGg...."
            let result = await b64toBlob(realData, contentType);
            array.push(result);
          };
        });
    },
    submit_thumbnail(index_number) {
      EventBus.$emit('thumbnail_change', index_number + 1);
      this.thumbnail_color(index_number);
      this.thumbnail_index = index_number + 1;
    },
    // 썸네일 클릭시 컬러 변경
    thumbnail_color(index_number) {
      const all_of_li = document.querySelectorAll(
        `.post-Picture li:nth-child(n+2)`,
      );
      all_of_li.forEach(v => {
        v.style.border = '2px solid rgb(66, 66, 66)';
      });
      const selected_li = document.querySelector(
        `.post-Picture > li:nth-child(${index_number + 2})`,
      );
      selected_li.style.border = '2px solid rgb(236, 75, 0)';
    },
  },
};
</script>

<style></style>

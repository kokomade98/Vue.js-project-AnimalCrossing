<template>
  <div class="modal-info">
    <ModalHeader :modal_title="title"></ModalHeader>

    <ion-content class="ion-padding qu-body">
      <ion-item>
        <ion-label position="floating"><span>*</span> 제목</ion-label>
        <ion-input
          :value="re_title"
          @input="re_title = $event.target.value"
          clear-on-edit="true"
        ></ion-input>
      </ion-item>
      <ion-item>
        <ion-label><span>*</span> 문의유형</ion-label>
        <ion-select
          interface="popover"
          placeholder="선택"
          :value="re_category"
          @ionChange="re_category = $event.target.value"
        >
          <ion-select-option value="회원정보">회원정보</ion-select-option>
          <ion-select-option v-if="re_bl_lists != null" value="계정정지">
            계정정지
          </ion-select-option>
          <ion-select-option value="거래">거래</ion-select-option>
          <ion-select-option value="개발자 피드백">
            개발자 피드백
          </ion-select-option>
        </ion-select>
      </ion-item>

      <ion-item class="qu-category" v-if="re_category == '계정정지'">
        <ion-label><span>*</span> 신고당한 게시글</ion-label>
        <ion-select
          interface="popover"
          placeholder="선택"
          :value="re_bl_id"
          @ionChange="re_bl_id = $event.target.value"
        >
          <ion-select-option
            :value="list.bl_id"
            v-for="(list, index) of re_bl_lists"
            :key="index"
          >
            {{ list.bl_bo_title }}
          </ion-select-option>
        </ion-select>
      </ion-item>
      <ion-textarea
        placeholder="문의 내용을 적어주세요."
        class="info-textarea"
        :value="re_content"
        @input="re_content = $event.target.value"
      ></ion-textarea>
      <div class="ani-btn sky" @click="submit_post()">문의하기</div>
    </ion-content>
  </div>
</template>

<script>
import store from '@/store/index';
import ModalHeader from '@/components/common/ModalHeader';
import { createReport, getBlacklist } from '@/api/report';
import { toastController, toastErrorController } from '@/utils/toastController';

export default {
  props: ['us_id'],
  components: {
    ModalHeader,
  },
  data() {
    return {
      title: '1 : 1 문의',
      re_title: '',
      re_content: '',
      re_category: null,
      re_bl_id: null,
      re_bl_lists: null,
    };
  },
  async mounted() {
    // 유저 => 블랙리스트 회원 검사 이벤트
    const { data } = await getBlacklist(this.us_id);
    this.re_bl_lists = data.info;
  },
  methods: {
    // 1 : 1 문의 전송 이벤트
    async submit_post() {
      if (this.re_title == '' || this.re_content == '') {
        let str = '';
        this.re_title == '' ? (str = '제목') : (str = '문의내용');
        return toastController(this.$ionic, str + '을 입력해주세요.', 'danger');
      }

      const data = {
        re_us_id: this.us_id,
        re_title: this.re_title,
        re_content: this.re_content,
        re_category: this.re_category,
      };
      if (this.re_category == '계정정지') data.re_bl_id = this.re_bl_id;
      try {
        await createReport(data);
        this.$ionic.modalController.dismiss();
      } catch (err) {
        toastErrorController(this.$ionic, err);
      }
    },
  },
};
</script>

<style></style>

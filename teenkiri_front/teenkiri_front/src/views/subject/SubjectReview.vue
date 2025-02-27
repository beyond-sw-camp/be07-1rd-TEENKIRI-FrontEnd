<template>
  <v-app>
    <v-container>
      <SubjectDetailComponent v-model="selectedMenu" />

      <div class="btnWrap" v-if="hasEnrolled">
        <a href="javascript:void(0)" class="btn_write" @click="handleReviewButtonClick">후기 작성</a>
      </div>

      <!-- 리뷰 목록 -->
      <table class="tbl_list">
        <caption></caption>
        <colgroup>
          <col width="80" />
          <col width="" />
          <col width="140" />
          <col :width="isAdmin ? '180' : '280'" /> <!-- 작성일 열의 너비를 관리자가 아닐 때 넓힘 -->
          <col v-if="isAdmin" width="100" /> <!-- 관리 열 -->
        </colgroup>
        <thead>
          <tr>
            <th>번호</th>
            <th>내용</th>
            <th>작성자</th>
            <th>작성일</th>
            <th v-if="isAdmin">관리</th> <!-- 관리 버튼 헤더 -->
          </tr>
        </thead>
        <tbody>
          <tr v-if="reviews.length === 0">
            <td :colspan="isAdmin ? 5 : 4" class="text_center">작성된 후기가 없습니다.</td>
          </tr>
          <tr v-for="(review, index) in reviews" :key="review.id">
            <td>{{ index + 1 }}</td>
            <td class="text_left">
              <span class="rating-text">별점: {{ review.rating }} / 5</span><br />
              {{ review.reviewText }}
            </td>
            <td>{{ review.nickname }}</td>
            <td>{{ formatDate(review.createdTime) }}</td>
            <td v-if="isAdmin">
              <v-btn small color="red" @click="deleteReview(review.id)">삭제</v-btn>
            </td>
          </tr>
        </tbody>
      </table>

      <!-- 후기 작성 다이얼로그 -->
      <v-dialog v-model="dialog" max-width="500px">
        <v-card>
          <v-card-title>
            <span class="headline">후기 작성</span>
          </v-card-title>
          <v-card-text>
            <v-form ref="reviewForm" v-model="valid">
              <v-rating v-model="newReview.rating" label="평점" required></v-rating>
              <v-textarea
                v-model="newReview.reviewText"
                label="후기"
                rows="5"
                required
              ></v-textarea>
            </v-form>
          </v-card-text>
          <v-card-actions>
            <v-spacer></v-spacer>
            <v-btn color="blue darken-1" text @click="closeReviewForm">취소</v-btn>
            <v-btn color="blue darken-1" text @click="submitReview">작성</v-btn>
          </v-card-actions>
        </v-card>
      </v-dialog>

      <!-- 수강 상태에 따른 알림 모달 -->
      <v-snackbar v-model="enrollmentSnackbar" :timeout="3000">
        수강 신청을 해야 후기를 작성할 수 있습니다.
        <v-btn color="red" text @click="enrollmentSnackbar = false">Close</v-btn>
      </v-snackbar>
    </v-container>
  </v-app>
</template>

<script>
import axios from 'axios';
import SubjectDetailComponent from '@/components/subject/SubjectDetailComponent.vue';

export default {
  components: {
    SubjectDetailComponent,
  },
  data() {
    return {
      user: {
        token: '',
        userId: '',
        email: '',
        nickname: '',
      },
      subjectId: this.$route.params.id, 
      selectedMenu: 'SubjectReview',
      reviews: [],
      dialog: false,
      valid: false, 
      newReview: {
        rating: 0,
        reviewText: '',
      },
      enrollmentSnackbar: false, // 수강 여부에 따른 모달
      hasEnrolled: false, // 수강 여부 상태를 나타내는 변수
    };
  },
  computed: {
    isAdmin() {
      return this.user.role === 'ADMIN';
    },
    hasUserAlreadyReviewed() {
      return this.reviews.some(review => review.userId === parseInt(this.user.userId, 10));
    }
  },
  async created() {
    try {
      await this.$store.dispatch('setUserAllInfoActions');
      this.user = this.$store.getters.getUserObj;

      if (this.user.token === '') {
        alert('로그인이 필요합니다.');
        location.href = -1;
      } else {
        this.fetchReviews();
        this.checkSubject();
      }
    } catch (error) {
      console.error('사용자 정보를 가져오는 중 오류가 발생했습니다:', error);
    }
  },
  methods: {
    async fetchReviews() {
      try {
        const response = await axios.get(`${process.env.VUE_APP_API_BASE_URL}/reviews/list`, {
          params: { subject_id: this.subjectId },
        });

        if (response.data && response.data.status_code === 200 && Array.isArray(response.data.result.content)) {
          this.reviews = response.data.result.content;
        } else {
          console.warn('리뷰 데이터를 찾을 수 없습니다.');
          this.reviews = [];
        }
      } catch (error) {
        console.error('리뷰 목록을 불러올 수 없습니다:', error);
        this.reviews = [];
      }
    },
    async checkSubject() {
      try {
        const response = await axios.get(`${process.env.VUE_APP_API_BASE_URL}/my/subject/${this.subjectId}`);
        const subjects = response.data.result.subjects;

        this.hasEnrolled = subjects.some(subject => subject.id === parseInt(this.subjectId, 10));

        if (!this.hasEnrolled) {
          this.enrollmentSnackbar = true; // 수강 상태 알림 모달을 표시합니다.
        }
      } catch (error) {
        console.error('수강 상태 확인 중 오류가 발생했습니다:', error);
      }
    },
    handleReviewButtonClick() {
      if (this.hasUserAlreadyReviewed) {
        alert('후기는 한번만 작성이 가능합니다.');
      } else {
        this.openReviewForm();
      }
    },
    openReviewForm() {
      this.dialog = true;
    },
    closeReviewForm() {
      this.dialog = false;
    },
    async submitReview() {
      if (this.$refs.reviewForm.validate()) {
        const reviewPayload = {
          rating: this.newReview.rating,
          reviewText: this.newReview.reviewText,
          subjectId: this.subjectId,
          userId: this.user.userId, 
        };

        try {
          const response = await axios.post(`${process.env.VUE_APP_API_BASE_URL}/reviews/create`, reviewPayload, {
            headers: {
              'Authorization': `Bearer ${this.user.token}`,
            },
          });

          if (response.status === 201) {
            this.reviews.unshift({
              ...reviewPayload,
              id: response.data.data,
              createdTime: new Date(),
            });
            alert('리뷰가 성공적으로 등록되었습니다.');
            this.closeReviewForm();
          } else if (response.status === 409) {
            throw new Error('이미 이 과목에 대한 리뷰를 작성하셨습니다.');
          }
        } catch (error) {
          console.error('리뷰 제출 중 오류가 발생했습니다.', error);
          alert(error.message);
        }
      }
    },
    async deleteReview(reviewId) {
      if (confirm('정말로 이 리뷰를 삭제하시겠습니까?')) {
        try {
          const response = await axios.get(`${process.env.VUE_APP_API_BASE_URL}/reviews/delete/${reviewId}`, {
            headers: {
              'Authorization': `Bearer ${this.user.token}`,
            },
          });

          if (response.status === 200) {
            this.reviews = this.reviews.filter(review => review.id !== reviewId);
            alert('리뷰가 삭제되었습니다.');
          } else {
            throw new Error('리뷰 삭제 중 오류가 발생했습니다.');
          }
        } catch (error) {
          console.error('리뷰 삭제 중 오류가 발생했습니다.', error);
          alert(error.message);
        }
      }
    },
    formatDate(date) {
      const options = { year: 'numeric', month: 'long', day: 'numeric' };
      return new Date(date).toLocaleDateString(undefined, options);
    }},
}
</script>

<style scoped>
.container {
  padding-top: 20px;
}

.btnWrap {
  text-align: right;
  margin-bottom: 20px;
}

.btn_write {
  background-color: #f27885;
  color: #fff;
  padding: 10px 20px;
  text-decoration: none;
  border-radius: 5px;
  font-size: 16px;
  display: inline-block;
  transition: background-color 0.3s ease;
}

.tbl_list {
  width: 100%;
  border-collapse: collapse;
  margin-bottom: 20px;
}

.tbl_list th,
.tbl_list td {
  border-top: 1px solid #ddd;
  border-bottom: 1px solid #ddd;
  padding: 10px;
  text-align: left;
  border-left: none; 
  border-right: none; 
}

.tbl_list th {
  background-color: #f4f4f4;
  font-weight: bold;
}

.text_left {
  text-align: left;
}

.text_center {
  text-align: center;
}

.rating-text {
  font-weight: bold;
}

.pagingWrap {
  text-align: center;
  margin-top: 20px;
}
</style>

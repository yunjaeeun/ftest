<template>
  <div class="help-container">
    <div class="help-detail">
      <div v-if="!editingPost">

        <!-- 제목 및 작성자 정보 -->
        <div class="post-header">
          <h1>{{ help?.help_title }}</h1>
          <div class="post-meta">
            <div class="author-info">
              <span class="author">작성자: {{ help?.user?.nickname }}</span>
              <span class="date">작성일: {{ formatDate(help?.help_date) }}</span>
            </div>
            <button
            @click="toggleLike"
            class="like-btn"
            :class="{ liked: isLiked }"
          >
            ❤️ {{ likeCount }}
          </button>
          </div>
        <div class="top-navigation">
          <router-link to="/help">
            <button class="btn back-btn">목록으로 돌아가기</button>
          </router-link>
          <div v-if="isAuthor" class="author-actions">
            <img 
              @click="startEditPost" 
              src="@/assets/images/boards/put.png" 
              alt="수정" 
              class="action-icon"
            >
            <img 
              @click="deletePost" 
              src="@/assets/images/boards/delete.png" 
              alt="삭제" 
              class="action-icon"
            >
          </div>
        </div>
        </div>

        <!-- 작성자만 보이는 수정/삭제 버튼 -->
        <div v-if="accountStore.isAdmin || help?.is_author" class="author-actions">
          <button @click="startEditPost" class="btn edit-btn">수정</button>
          <button @click="deletePost" class="btn delete-btn">삭제</button>
        </div>
      </div>

      <!-- 게시글 수정 폼 -->
      <div v-else class="edit-form">
        <input
          v-model="editedTitle"
          class="edit-title"
          type="text"
          placeholder="제목"
        />
        <textarea
          v-model="editedContent"
          class="edit-content"
          placeholder="내용"
          rows="5"
        ></textarea>
        <div class="edit-actions">
          <button @click="savePostEdit" class="btn save-btn">저장</button>
          <button @click="cancelPostEdit" class="btn cancel-btn">취소</button>
        </div>
      </div>

      <!-- 댓글 섹션 -->
      <div v-if="isLoggedIn" class="comments-section">
        <h3>댓글</h3>
        <div class="comment-form">
          <textarea
            v-model="newComment"
            placeholder="댓글을 작성해주세요"
            rows="3"
          ></textarea>
          <button @click="submitComment" class="btn comment-btn">
            댓글 작성
          </button>
        </div>

        <div class="comments-list">
    <div v-for="comment in comments" :key="comment.id" class="comment">
      <div v-if="editingCommentId !== comment.id">
        <div class="comment-header">
          <h3 class="comment-content">{{ comment.help_comment_content }}</h3>
          <div class="comment-meta">
            <div class="comment-info">
              <span class="comment-author">{{ comment.user.nickname }}</span>
              <span class="comment-date">{{ formatDate(comment.help_comment_date) }}</span>
            </div>
          </div>
        </div>
        <div v-if="comment.is_author" class="comment-actions">
          <img 
            @click="startEditComment(comment)" 
            src="@/assets/images/boards/put.png" 
            alt="수정" 
            class="action-icon"
          >
          <img 
            @click="deleteComment(comment.id)" 
            src="@/assets/images/boards/delete.png" 
            alt="삭제" 
            class="action-icon"
          >
        </div>
      </div>
    </div>
  </div>
      </div>
    </div>
  </div>
</template>


<script setup>
import { ref, computed, onMounted, watch } from "vue";
import { useRoute, useRouter } from "vue-router";
import axios from "axios";
import { useAccountStore } from "@/stores/account";

const route = useRoute();
const router = useRouter();
const accountStore = useAccountStore();

// 기존 상태 변수들
const help = ref(null);
const comments = ref([]);
const newComment = ref("");
const isLiked = ref(false);
const likeCount = ref(0);

// 게시글 수정 관련 상태
const editingPost = ref(false);
const editedTitle = ref("");
const editedContent = ref("");

// 댓글 수정 관련 상태
const editingCommentId = ref(null);
const editedCommentContent = ref("");

const isLoggedIn = computed(() => !!accountStore.token);

// 게시글 수정 시작
const startEditPost = () => {
  editedTitle.value = help.value.help_title;
  editedContent.value = help.value.help_content;
  editingPost.value = true;
};

// 게시글 수정 취소
const cancelPostEdit = () => {
  editingPost.value = false;
  editedTitle.value = "";
  editedContent.value = "";
};

// 게시글 수정 저장
const savePostEdit = async () => {
  try {
    const response = await axios.put(
      `http://127.0.0.1:8000/boards/help/${route.params.id}/`,
      {
        help_title: editedTitle.value,
        help_content: editedContent.value
      },
      {
        headers: {
          Authorization: `Token ${accountStore.token}`
        }
      }
    );
    help.value = { ...response.data, is_author: true };
    editingPost.value = false;
  } catch (error) {
    console.error("게시글 수정 실패:", error);
    alert(error.response?.data?.error || "게시글 수정에 실패했습니다.");
  }
};

// 댓글 수정 시작
const startEditComment = (comment) => {
  editingCommentId.value = comment.id;
  editedCommentContent.value = comment.help_comment_content;
};

// 댓글 수정 취소
const cancelCommentEdit = () => {
  editingCommentId.value = null;
  editedCommentContent.value = "";
};

// 댓글 수정 저장
const saveCommentEdit = async (commentId) => {
  try {
    // 현재 수정 중인 댓글 찾기
    const currentComment = comments.value.find(c => c.id === commentId);
    
    const response = await axios.put(
      `http://127.0.0.1:8000/boards/help/comments/${commentId}/`,
      {
        help_comment_content: editedCommentContent.value,
        help_article: help.value.id,  // 게시글 ID 추가
        user: currentComment.user     // 사용자 ID 추가
      },
      {
        headers: {
          Authorization: `Token ${accountStore.token}`
        }
      }
    );

    // 수정된 댓글로 업데이트
    const index = comments.value.findIndex(c => c.id === commentId);
    // 기존 댓글의 속성을 보존하면서 수정된 내용 반영
    comments.value[index] = {
      ...comments.value[index],
      help_comment_content: editedCommentContent.value
    };
    
    // 수정 모드 종료
    cancelCommentEdit();
  } catch (error) {
    console.error("댓글 수정 실패:", error);
    alert(error.response?.data?.error || "댓글 수정에 실패했습니다.");
  }
};


const formatDate = (date) => {
  if (!date) return "";
  return date.split("T")[0];
};
const isAuthor = computed(() => help.value?.is_author || false);
const isCommentAuthor = (comment) => comment.user_id === accountStore.userId;


// 게시글 데이터 로드 및 좋아요 상태 초기화
onMounted(async () => {
  if (!isLoggedIn.value) {
    router.push('/login');
    return;
  }

  try {
    const response = await axios.get(
      `http://127.0.0.1:8000/boards/help/${route.params.id}/`,
      {
        headers: {
          Authorization: `Token ${accountStore.token}`,
        },
      }
    );
    help.value = response.data;
    likeCount.value = response.data.like_count;
    isLiked.value = response.data.is_liked;

    loadComments();
  } catch (error) {
    console.error("게시글 로드 실패", error);
  }
});

// 게시글 수정
const editPost = () => {
  router.push(`/help/edit/${route.params.id}`);
};

// 게시글 삭제
const deletePost = async () => {
  if (!confirm('정말 삭제하시겠습니까?')) return;
  
  try {
    await axios.delete(
      `http://127.0.0.1:8000/boards/help/${route.params.id}/`,
      {
        headers: {
          Authorization: `Token ${accountStore.token}`,
        },
      }
    );
    router.push('/help');
  } catch (error) {
    console.error("게시글 삭제 실패:", error);
    alert(error.response?.data?.error || '삭제에 실패했습니다.');
  }
};

// 댓글 목록 로드
const loadComments = async () => {
  try {
    const response = await axios.get(
      `http://127.0.0.1:8000/boards/help/${route.params.id}/comments/`,
      {
        headers: {
          Authorization: `Token ${useAccountStore().token}`,
        },
      }
    );
    comments.value = response.data;
  } catch (error) {
    console.error("댓글 로드 실패:", error);
  }
};

// 댓글 작성
const submitComment = async () => {
  if (!newComment.value.trim()) return;
  try {
    const response = await axios.post(
      `http://127.0.0.1:8000/boards/help/${route.params.id}/comments/`,
      { help_comment_content: newComment.value },
      { headers: { Authorization: `Token ${useAccountStore().token}` } }
    );

    comments.value.unshift(response.data);
    newComment.value = "";
  } catch (error) {
    console.error("댓글 작성 실패:", error.response?.data);
  }
};

// 댓글 수정
const editComment = async (comment) => {
  // 구현 예정
  console.log('댓글 수정:', comment);
};

// 댓글 삭제
const deleteComment = async (commentId) => {
  if (!confirm('댓글을 삭제하시겠습니까?')) return;
  
  try {
    await axios.delete(
      `http://127.0.0.1:8000/boards/help/comments/${commentId}/`,
      {
        headers: {
          Authorization: `Token ${accountStore.token}`,
        },
      }
    );
    loadComments(); // 댓글 목록 새로고침
  } catch (error) {
    console.error("댓글 삭제 실패:", error);
    alert('댓글 삭제에 실패했습니다.');
  }
};

// 좋아요 토글
const toggleLike = async () => {
  try {
    const response = await axios.post(
      `http://127.0.0.1:8000/boards/help/${route.params.id}/like/`,
      {},
      {
        headers: {
          Authorization: `Token ${useAccountStore().token}`,
        },
      }
    );

    // 서버에서 받은 최신 좋아요 수와 상태로 업데이트
    likeCount.value = response.data.like_count;
    isLiked.value = response.data.is_liked;
  } catch (error) {
    console.error("좋아요 처리 실패:", error);
  }
};

window.onbeforeunload = function() {
    window.scrollTo(0, 0);
};
</script>

<style scoped>
.help-container {
  max-width: 800px;
  margin: 2rem auto;
  padding: 0 1rem;
}

.help-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-top: 4rem;
  margin-bottom: 4rem;
}

.help-detail {
  background-color: white;
  padding: 2rem;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.help-detail h1 {
  color: #40a2e3;
  margin-bottom: 1.5rem;
  font-size: 2rem;
}

.help-detail p {
  color: #444;
  line-height: 1.6;
  /* margin-bottom: 2rem; */
}

.top-navigation {
  display: flex;
  justify-content: space-between;
  align-items: center;
  align-items: end;
  /* margin-bottom: 2rem; */
}

/* 제목 및 작성자 정보 */
.post-header {
  border-bottom: 1px solid #eee;
  padding-bottom: 1.5rem;
  margin-bottom: 2rem;
}

.post-header h1 {
  font-size: 2rem;
  color: #40a2e3;
  margin-bottom: 1rem;
}

.post-meta {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.btn {
  background-color: #40a2e3;
  color: white;
  padding: 0.5rem 1rem;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 0.8rem;
  transition: background-color 0.2s;
}

.btn:hover {
  background-color: #2c7ba8;
}

.back-btn {
  margin-top: 0.5rem;
  display: inline-block;
}

.post-info {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1.5rem;
  color: #666;
}

.content {
  white-space: pre-line;
  word-break: break-word;
}

.like-btn {
  padding: 0.3rem 0.6rem;
  border: 2px solid white;
  border-radius: 20px;
  background: white;
  font-size: 1rem;
  cursor: pointer;
  transition: all 0.2s;
}

.like-btn.liked {
  background: #40a2e3;
  color: white;
}

.comments-section {
  margin-top: 2rem;
  padding-top: 2rem;
  border-top: 1px solid #eee;
}

.comment-form {
  margin-bottom: 2rem;
  display: flex;
  flex-direction: column;
  align-items: flex-end;
}

.comment-form textarea {
  width: 100%;
  padding: 0.75rem;
  border: 1px solid #ddd;
  border-radius: 4px;
  margin-bottom: 1rem;
  resize: vertical;
}

.comment-btn {
  background-color: #40a2e3;
  color: white;
}

.comment {
  padding: 1.5rem 0;
  border-bottom: 1px solid #eee;
}

.comment-header {
  margin-bottom: 0.5rem;
}

.comment-meta {
  margin-bottom: 1rem;
}

.comment-info {
  display: flex;
  gap: 2rem;
  color: #666;
}

.comment-author {
  font-weight: bold;
}

.comment-date {
  color: #666;
  font-size: 0.9rem;
}

.comment-content {
  color: #40a2e3;
  font-size: 1.2rem;
  margin-bottom: 0.5rem;
}

.author-info {
  display: flex;
  gap: 2rem;
}

.author-actions {
  margin-top: 1rem;
  display: flex;
  gap: 1rem;
}

.action-icon {
  width: 24px;
  height: 24px;
  cursor: pointer;
  transition: opacity 0.2s;
}

.action-icon:hover {
  opacity: 0.7;
}

.comment-actions {
  margin-top: 0.5rem;
  display: flex;
  gap: 0.5rem;
  justify-content: flex-end;  /* 오른쪽 정렬 */
}

.login-message {
  text-align: center;
  padding: 1rem;
  color: #666;
}

.btn {
  padding: 0.5rem 1rem;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  transition: all 0.2s;
}

.btn:hover {
  opacity: 0.9;
}

.edit-form {
  margin: 1rem 0;
}

.edit-title {
  width: 100%;
  padding: 0.5rem;
  font-size: 1.5rem;
  margin-bottom: 1rem;
  border: 1px solid #ddd;
  border-radius: 4px;
}

.edit-content {
  width: 100%;
  padding: 0.5rem;
  min-height: 200px;
  margin-bottom: 1rem;
  border: 1px solid #ddd;
  border-radius: 4px;
}

.edit-actions {
  display: flex;
  gap: 1rem;
  margin-bottom: 1rem;
}

.save-btn {
  background-color: #288745;
  color: white;
}

.cancel-btn {
  background-color: #666;
  color: white;
}

.save-btn-sm, .cancel-btn-sm {
  padding: 0.25rem 0.5rem;
  font-size: 0.8rem;
}

.edit-comment-textarea {
  width: 100%;
  padding: 0.5rem;
  margin-bottom: 0.5rem;
  border: 1px solid #ddd;
  border-radius: 4px;
}

.edit-comment-actions {
  display: flex;
  gap: 0.5rem;
}

.author {
  color: #666;
  font-weight: bold;
  margin: 0;
}

.date {
  color: #666;
}

.edit-btn-sm, .delete-btn-sm {
  padding: 0.25rem 0.5rem;
  font-size: 0.8rem;
}

.action-icon[src*="put"]:hover {
  filter: invert(24%) sepia(97%) saturate(431%) hue-rotate(90deg) brightness(92%) contrast(89%);
}

.action-icon[src*="delete"]:hover {
  filter: invert(40%) sepia(89%) saturate(2526%) hue-rotate(338deg) brightness(90%) contrast(111%);
}

</style>

<template>
  <div class="layout">
    <!-- 태그판 (Sidebar) -->
    <div class="sidebar">
      <ul class="tag-list">
        <li v-for="(tag, index) in tags" :key="index" class="tag-item">
          <button @click="selectTag(tag)">{{ tag }}</button>
        </li>
      </ul>
    </div>

    <!-- 게시물 목록 (Main Content) -->
    <div class="board-container"  >
      <div class="actions">
        <!-- 글 작성 버튼 -->
        <button class="create-button" @click="goToCreate">글 작성</button>
      </div>
      <div v-for="item in boards" :key="item.id" class="board-card">
        
        <div class="user-info">
          <img :src="item.memberProfileUrl" alt="User Profile" class="profile-image" @click="goToProfile(item.memberNickname)">
          <span class="nickname" @click="goToProfile(item.memberNickname)">{{ item.memberNickname }} </span>
        </div>
        
        <hr class="divider"/>
        <div class="image-container">
          <img 
            v-for="(image, i) in item.imageObj.slice(0, 3)" 
            :key="i" 
            :src="image.imageUrl" 
            :alt="image.fileName" 
            class="board-image" 
            @click="openModal(item, i)" 
          />
        </div>

        <div class="board-footer">
          <img
            :src="item.liked ? filledHeartIcon : emptyHeartIcon"
            class="svg-icon"
            alt="Heart Icon"
            @click="upLike(item.id)"
          />
          <img src="@/assets/icon/boardIcons/comment.svg" class="svg-icon" alt="Comment Icon" @click="openModal(item)" />
          <img src="@/assets/icon/boardIcons/letter.svg" class="svg-icon" alt="letter Icon" @click="goToChat(item)" />
        </div> 

        <h3 class="board-title">{{ item.title }}</h3>

        <div class="content-container">
          <p class="board-content" v-if="!item.showFullContent">
            <span v-html="formatContent(getFirstLine(item.content))"></span>
            <span v-if="item.content.includes('\n') || item.content.length > getFirstLine(item.content).length">...</span>
          </p>
          <p class="board-content" v-else>
            <span v-html="formatContent(item.content)"></span>
          </p>
          <button v-if="!item.showFullContent && item.content.length > 30" @click="toggleContent(item)" class="more-button">
            더보기
          </button>
        </div>

        <!-- 좋아요와 댓글 수 -->
        <!-- <div class="interaction-info">
          <span>좋아요 {{ item.likesCount }}개</span>
          <span>댓글 {{ item.commentCount }}개</span>
        </div> -->

      </div>
      <!-- 무한 스크롤을 위한 sentinel -->
      <div ref="sentinel" class="sentinel"></div>
    </div>

    <div v-if="loading" class="loading">Loading...</div>
    
    <!-- Modal Component -->
    <board-detail 
      v-if="isModalOpen" 
      :board="selectedBoard" 
      :initialImageIndex="selectedImageIndex" 
      :tag="tag"
      :liked="selectedBoard.liked" 
      @close="closeModal" 
    />
  </div>
</template>

<script setup>
import { useRouter, useRoute } from 'vue-router';
import { ref, onMounted, onUnmounted, watch } from 'vue';
import axios from 'axios';
import BoardDetail from './BoardDetail.vue'; // 모달 컴포넌트

const boards = ref([]);
const cursorId = ref('');
const hasNext = ref(true);
const loading = ref(false);
const sentinel = ref(null);
const tags = ref(['GUIDE', 'FREEMARKET', 'ACCOMPANY', 'TIP']); // 태그 목록
const likedBoardIds = ref(new Set());
const isModalOpen = ref(false);
const selectedBoard = ref(null);
const selectedImageIndex = ref(0);
const userInfo = localStorage.getItem('userInfo');
const token =localStorage.getItem('jwtToken');  

const router = useRouter();
const route = useRoute();

const tag = ref(route.params.tag || 'GUIDE');
const emptyHeartIcon = new URL('@/assets/icon/boardIcons/heart.svg', import.meta.url).href;
const filledHeartIcon = new URL('@/assets/icon/boardIcons/filledheart.svg', import.meta.url).href;

const fetchLikedBoards = async () => {
    try {
        const token = localStorage.getItem('jwtToken');
        const response = await axios.get('http://localhost:8080/api/v1/board_likes', {
            headers: { Authorization: `Bearer ${token}` },
        });

        likedBoardIds.value = new Set(response.data); // 좋아요한 게시글 ID 저장
    } catch (error) {
        console.error('Error fetching liked boards:', error.response?.data || error.message);
    }
};

const fetchBoardItems = async (reset = false) => {
    if (loading.value || (!reset && !hasNext.value)) return;

    loading.value = true;

    if (reset) {
        boards.value = [];
        cursorId.value = '';
        hasNext.value = true;
    }

    const initialSize = 3;
    let currentSize = initialSize;
    let newContents = [];
    let currentTag = '';
    const targetSize = 3;  // 목표로 하는 게시물 수

    try {
      while (newContents.length < targetSize && hasNext.value) {
        const response = await axios.get(`http://localhost:8080/api/v1/board/${tag.value}`, {
          params: { cursor: cursorId.value || '', size: currentSize }
        });
        let data = response.data;
        let parsedData = [];
            
        console.log("Parsed Data:", data);
        if (typeof data === 'string') {
          const jsonParts = data.match(/\{.*?\}(?=\{|\s*$)/g) || [];
          if (jsonParts.length > 0) {
            try {
              const parsed = JSON.parse(jsonParts[0]);
              parsedData = parsed.result?.comment || [];
              currentTag = data.result?.tag || tag.value;
              cursorId.value = parsed.result?.cursorId || '';
              hasNext.value = parsed.result?.hasNext;
              } catch (error) {
                console.error("JSON 파싱 실패:", error);
              }
          }
        } else {
          parsedData = data.result?.comment || [];
          currentTag = data.result?.tag || tag.value;
          cursorId.value = data.result?.cursorId || '';
          hasNext.value = data.result?.hasNext;
        }
            
        console.log("Parsed data:", parsedData);

        // 활성 게시물만 필터링
        const filteredContents = parsedData.filter(board => board.active === true)
          .map(board => ({
            ...board,
            tag: currentTag  // 각 게시글에 태그 정보 추가
          }));
            
            // 좋아요 상태 반영
        filteredContents.forEach(board => {
          board.liked = likedBoardIds.value.has(board.id);
        });
            
        newContents = [...newContents, ...filteredContents];
            
        console.log("New contents after filtering:", newContents);

        if (newContents.length < targetSize && hasNext.value) {
          // currentSize *= 2;  // 다음 요청 시 더 많은 데이터를 가져오기
          console.log("Increasing request size to:", currentSize);
        } else {
          break;  // 목표 크기에 도달하거나 더 이상 가져올 데이터가 없으면 루프 종료
        }
      }
      boards.value = [...boards.value, ...newContents];

        if (boards.value.length === 0) {
            console.warn("No boards found.");
          } else {
            console.log("Total boards after update:", boards.value.length);
          }
    } catch (error) {
        console.error("API 호출 에러:", error.response?.data || error.message);
    } finally {
        loading.value = false;
    }
};

const openModal = (board, imageIndex) => {
  console.log(board.tag);

  selectedBoard.value = {
    ...board,
    imageUrls: board.imageObj.map(image => image.imageUrl),
  };
  selectedImageIndex.value = imageIndex; // Set the initial image index
  isModalOpen.value = true;
};


const closeModal = () => {
  isModalOpen.value = false;
};

const selectTag = (selectedTag) => {
  tag.value = selectedTag;
  fetchBoardItems(true);
};

// "더보기" 버튼 클릭 시 전체 내용 토글 함수
const toggleContent = (item) => {
    item.showFullContent = !item.showFullContent;
};

const observerOptions = {
    root: null,
    rootMargin: '0px',
    threshold: 1.0
};

const intersectionObserver = new IntersectionObserver((entries) => {
    entries.forEach(entry => {
        if (entry.isIntersecting && !loading.value && hasNext.value) {
            console.log('Intersection observed, fetching more items...'); // 디버깅 로그
            fetchBoardItems();
        }
    });
}, observerOptions);

const formatContent = (text) => {
  return text.replace(/\n/g, '<br>');
}

const getFirstLine = (text) => {
  return text.split('\n')[0]; // \n 기준으로 첫 번째 줄 반환
}

const goToProfile = (nickName) => {
  router.push(`/otherprofile/${nickName}`)
}

const goToCreate = () => {
  router.push('/board/create'); // 글 작성 페이지로 이동
};

const goToChat = async (item) => {

  const user = JSON.parse(userInfo);   
  const responseChat = await axios.post('http://localhost:8080/api/v1/chat', {
    sender: user.nickname,   // 로그인한 사용자 (채팅방 생성자)
    receiver: item.memberNickname,  // 입력된 상대방 이름
  }, {
    headers: {
      Authorization: `Bearer ${token}`,
    }
  });

  console.log(user.nickname);
  console.log(item.memberNickname);

  router.push('/chat');
};

const upLike = async (boardId) => {
    try {
        const token = localStorage.getItem('jwtToken');
        const boardIndex = boards.value.findIndex((board) => board.id === boardId);
        if (boardIndex === -1) return;

        const board = boards.value[boardIndex];
        const url = 'http://localhost:8080/api/v1/board_like';
        const headers = {
            'Content-Type': 'application/json',
            'Authorization': `Bearer ${token}`
        };

        if (board.liked) {
            await axios.delete(url, {
                headers,
                data: { boardId }
            });

            board.likesCount -= 1;
            board.liked = false;
            likedBoardIds.value.delete(boardId); // Set에서 삭제
            console.log('좋아요 취소 완료');
        } else {
            await axios.post(url, { boardId }, { headers });

            board.likesCount += 1;
            board.liked = true;
            likedBoardIds.value.add(boardId); // Set에 추가
            console.log('좋아요 추가 완료');
        }
    } catch (error) {
        console.error('좋아요 처리 에러:', error.response?.data || error.message);
    }
};

watch(
    () => route.params.tag,
    (newTag) => {
        tag.value = newTag || 'GUIDE';
        fetchBoardItems(true);
    }
);

onMounted(async () => {
    await fetchLikedBoards(); // 좋아요한 게시글 정보 먼저 로드
    await fetchBoardItems(); // 게시글 로드 후 좋아요 상태 반영
    if (sentinel.value) {
        intersectionObserver.observe(sentinel.value);
    }
});


onUnmounted(() => {
    if (sentinel.value) {
        intersectionObserver.unobserve(sentinel.value);
    }
});

router.beforeEach(async (to, from, next) => {
    if (to.path.startsWith('/board')) {
        await fetchLikedBoards(); // 좋아요한 게시글 정보 다시 로드
        await fetchBoardItems(true); // 페이지 변경 시 데이터 초기화
    }
    next();
});
</script>

<style scoped>
.actions {
  display: flex;
  justify-content: flex-end;
}

.create-button {
  border-radius: 6.954px;
  border: 0.993px solid var(--Main_1, #439AFF);
  background: var(--White_100, #FFF);
  color: var(--Main_1, #439AFF);
  width: 10rem;
  height: 3.5rem;
  justify-content: center;
  align-items: center;
  flex-shrink: 0;
  font-size: 2rem;
  
}

.create-button:hover {
  background-color: #73b3ff;
}

.layout {
  position: relative;
}

.sidebar {
  position: fixed;
  width: 15rem;
  height: 16.7rem;
  margin-top: 12rem;
  transition: top 0.3s ease;
  transform: translateX(7rem);
  border-radius: 7px;
  border: 0.5px solid var(--Main_-3, #94C7FF);
  background: var(--White_100, #FFF);
  box-shadow: 0px 4px 8px 0px rgba(19, 24, 48, 0.15);
  margin-left: -0.5rem;
}

.tag-list {
  list-style: none;
  padding: 0; /* 리스트 패딩 제거 */
  margin: 0; /* 리스트 마진 제거 */
}

.tag-item {
  margin: 0; /* 태그 항목 간격 제거 */
}

.tag-item button {
  width: 100%; /* 부모 요소의 너비에 맞게 설정 */
  height: 3.4rem; /* 버튼 높이 설정 */
  align-items: center; /* 수직 가운데 정렬 */
  /* justify-content: center; 수평 가운데 정렬 */
  border: none; /* 기본 테두리 제거 */
  background: var(--White_100, #FFF); /* 기본 배경색: 흰색 */
  color: var(--Black_60, #627086); /* 기본 텍스트 색상 */
  border-radius: 7px;
  cursor: pointer; /* 커서 변경 */
  font-size: 13px;
  font-style: italic;
  font-weight: 400;
  transition: background-color 0.3s, color 0.3s; /* 부드러운 전환 효과 */
  box-shadow: none; /* 기본 그림자 제거 */
}

.tag-item button:hover {
  background-color: var(--Main_-3, #94C7FF); /* 호버 시 활성화된 배경색 */
  color: var(--White_100, #FFF); /* 호버 시 텍스트 색상 변경 */
}

.tag-item button.active {
  background-color: var(--Main_-3, #94C7FF); /* 활성화된 배경색 */
  color: var(--White_100, #FFF); /* 활성화된 텍스트 색상 */
}


.tag-list {
  list-style-type: none;
  padding: 0;
}

.tag-item {
  margin-bottom: 1rem;
}

.board-container {
  width: 90rem;
  margin: 27rem ;
  margin-top: 1.5rem;
  margin-bottom: 1.5rem;
}

.board-card {
  background: #FFFFFF;
  border: 0.07rem solid #E0E0E0; /* 1px = 0.07rem */
  border-radius: 1.5rem; /* 8px = 0.5rem */
  padding: 2rem; /* 16px = 1rem */
  margin-bottom: 3rem; /* 카드 간의 아래 여백 추가 */
  margin-top: 2rem; /* 카드 간의 아래 여백 추가 */
  box-shadow: 0px 1.5px 4.6px 0px rgba(0, 0, 0, 0.25);
  transition: transform 0.3s ease-in-out, box-shadow 0.3s ease-in-out;
}

.board-card:hover {
  transform: scale(1.01); /* 호버 시 이미지 확대 */
  box-shadow: 0 6px 12px rgba(0, 0, 0, 0.15); /* 호버 시 그림자 진하게 */
}

.user-info {
  display: flex;
  align-items: center;
}

.profile-image {
  width: 5rem; /* 40px = 2.5rem */
  height: 5rem; /* 40px = 2.5rem */
  border-radius: 50%;
  margin-right: 0.75rem; /* 12px = 0.75rem */
}

.nickname {
  font-size: 2rem;
}

.board-title {
  font-weight: bold;
  font-size: 1.6rem; /* Adjust as needed */
  margin-bottom: 0; /* Remove margin below title */
  margin-left: 1rem;
}

.image-container {
  display: flex;
  padding: 1rem;
  box-sizing: border-box;
  width: 100%;
  gap: 1rem; /* 이미지 사이의 간격 */
}

.board-image {
  flex: 1;
  max-width: calc((100% - 2rem) / 3); /* 3개 이미지일 때 각각의 최대 너비 */
  height: 26rem;
  object-fit: cover;
  border-radius: 0.8rem;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
  transition: transform 0.3s ease-in-out, box-shadow 0.3s ease-in-out;
  cursor: pointer;
}

.board-footer {
  width: 2rem;
  height: 2rem;
  flex-shrink: 0;
  display: flex;
  margin-left: 1rem;
  gap: 0.8rem;
  margin-bottom: 1rem;
}

.interaction-count {
  font-size: 0.875rem; /* 14px = 0.875rem */
  color: #666;
}

.content-container {
  display: flex;
  align-items: center; /* 세로 중앙 정렬 */
  gap: 1rem; /* content와 버튼 간의 간격 */
  margin-left: 1rem;
}

.board-content {
  font-size: 2.2rem;
  margin-top: 0; 
  margin-bottom: 0; 
  white-space: pre-wrap; /* Ensure text formatting with line breaks is preserved */
  color: #666;
  line-height: 1.2; /* 줄 간 간격을 줄임 */
}

.more-button {
  background: none;
  border: none;
  color: #999; /* Grey color similar to Instagram UI */
  font-size: 1.5rem; /* Adjusted size to fit the style */
  cursor: pointer;
  padding: 0;
  text-decoration: none;
  transition: color 0.2s;
  margin-top: auto;
  margin-left: -0.5rem;
}

.more-button:hover {
  color: #555; /* Darker grey on hover */
}

.interaction-info {
  margin-left: 1rem;
  font-size: 1.3rem;
  color: #666;
  display: flex;
  gap: 0.3rem;
  font-size: 2rem;
  margin-top: 0.4rem;
}

.loading {
  text-align: center;
  padding: 1.25rem; /* 20px = 1.25rem */
  font-size: 1.125rem; /* 18px = 1.125rem */
  color: #666;
}

.svg-icon {
  width: 2rem;
  height: 2rem;
  cursor: pointer;
  margin-right: 0.5rem;
  transition: transform 0.2s;
}

.svg-icon:hover {
  transform: scale(1.2);
}
</style>
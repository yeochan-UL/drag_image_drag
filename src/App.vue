<template>
    <div class="app-container">
        <header class="header">
            <h2>드래그 앤 드롭 이미지 편집기</h2>
        </header>
        <section class="controls-section">
            <div class="control-group">
                <label>
                    배경 너비 (px):
                    <input type="number" v-model.number="bgWidth" placeholder="너비 입력" />
                </label>
                <label>
                    배경 높이 (px):
                    <input type="number" v-model.number="bgHeight" placeholder="높이 입력" />
                </label>
            </div>
            <div class="control-group">
                <button @click="triggerFileInput">이미지 추가</button>
                <input type="file" ref="fileInput" accept="image/*" @change="handleFileChange" style="display: none;" />
            </div>
        </section>
        <section class="canvas-section">
            <!-- 캔버스 래퍼: 창 크기에 따라 캔버스 전체를 스케일 -->
            <div class="canvas-wrapper" :style="{
                width: (bgWidth * canvasScale).toFixed(0) + 'px',
                height: (bgHeight * canvasScale).toFixed(0) + 'px'
            }">
                <div class="canvas" ref="canvasRef" :style="{
                    width: bgWidth + 'px',
                    height: bgHeight + 'px',
                    backgroundImage: 'url(' + conImage + ')',
                    transform: 'scale(' + canvasScale + ')',
                    transformOrigin: 'top left'
                }" @mousedown="canvasMouseDown" @mousemove="onDrag" @mouseup="endDrag" @mouseleave="endDrag">
                    <img v-for="(item, index) in items" :key="item.id" :src="item.src" :style="{
                        position: 'absolute',
                        left: item.x + 'px',
                        top: item.y + 'px',
                        cursor: 'move',
                        width: item.width + 'px',
                        height: item.height + 'px'
                    }" @mousedown.stop="selectItem(index, $event)" @dblclick.stop="enableResize(index)" />
                </div>
            </div>
        </section>
        <span>ⓒ 찬찬잉</span>
        <div v-if="resizingItem !== null" class="resize-modal">
            <div class="resize-content">
                <h3>이미지 사이즈 조절</h3>
                <div class="input-group">
                    <label>
                        Width (px):
                        <input type="number" v-model.number="resizeWidth" />
                    </label>
                    <label>
                        Height (px):
                        <input type="number" v-model.number="resizeHeight" />
                    </label>
                </div>
                <div class="resize-buttons">
                    <button class="btn apply" @click="applyResize">적용</button>
                    <button class="btn cancel" @click="cancelResize">취소</button>
                    <button class="btn delete" @click="deleteItem">삭제</button>
                </div>
            </div>
        </div>
    </div>
</template>

<script setup>
import { ref, reactive, onMounted, computed } from 'vue';
// assets 폴더의 container.png를 배경으로 사용 (alias '@'가 설정되어 있다고 가정)
import conImage from './assets/container.png';

// 캔버스 크기 관련 상태
const bgWidth = ref(500);
const bgHeight = ref(500);

// 드래그할 이미지 항목 배열 (각 항목에 width, height 속성 포함)
const items = ref([]);

// 드래그 상태 관리 객체
const dragging = reactive({
    active: false,
    itemIndex: null,
    offsetX: 0,
    offsetY: 0,
});

// 파일 input 참조
const fileInput = ref(null);
const triggerFileInput = () => {
    fileInput.value.click();
};

const handleFileChange = (event) => {
    const file = event.target.files[0];
    if (!file) return;
    const reader = new FileReader();
    reader.onload = (e) => {
        const img = new Image();
        img.onload = () => {
            // 이미지가 로드된 후 자연 크기를 사용하여 항목 추가
            items.value.push({
                id: Date.now(),
                src: e.target.result,
                x: 50,
                y: 50,
                width: img.naturalWidth,
                height: img.naturalHeight,
            });
            saveItems();
        };
        img.src = e.target.result;
    };
    reader.readAsDataURL(file);
    event.target.value = null;
};

const canvasRef = ref(null);

// 반응형: 창 크기에 따라 캔버스 전체 스케일 계산
const windowWidth = ref(window.innerWidth);

onMounted(() => {
    window.addEventListener('resize', () => {
        windowWidth.value = window.innerWidth;
    });
    const savedItems = localStorage.getItem('draggableItems');
    if (savedItems) {
        try {
            items.value = JSON.parse(savedItems);
        } catch (e) {
            console.error('저장된 이미지 데이터를 불러오는 중 에러 발생', e);
        }
    }
});

const canvasScale = computed(() => {
    // 화면 너비의 90%를 기준으로 스케일 적용
    const containerWidth = windowWidth.value * 0.9;

    if (bgWidth.value > containerWidth) {
        return containerWidth / bgWidth.value;
    }

    return 1;
});

// 마우스 이벤트 (좌표 계산 시 스케일 보정)
const selectItem = (index, event) => {
    const rect = canvasRef.value.getBoundingClientRect();
    dragging.active = true;
    dragging.itemIndex = index;
    dragging.offsetX = (event.clientX - rect.left) / canvasScale.value - items.value[index].x;
    dragging.offsetY = (event.clientY - rect.top) / canvasScale.value - items.value[index].y;
};

const canvasMouseDown = (event) => {
    // 추가 기능
    console.log(event, '이벤트');
};

const onDrag = (event) => {
    if (!dragging.active || dragging.itemIndex === null) return;
    const rect = canvasRef.value.getBoundingClientRect();
    items.value[dragging.itemIndex].x = (event.clientX - rect.left) / canvasScale.value - dragging.offsetX;
    items.value[dragging.itemIndex].y = (event.clientY - rect.top) / canvasScale.value - dragging.offsetY;
};

const endDrag = () => {
    if (dragging.active) {
        dragging.active = false;
        dragging.itemIndex = null;
        saveItems();
    }
};

const saveItems = () => {
    localStorage.setItem('draggableItems', JSON.stringify(items.value));
};

// 사이즈 조절 관련 상태
const resizingItem = ref(null);
const resizeWidth = ref(150);
const resizeHeight = ref(150);

const enableResize = (index) => {
    resizingItem.value = index;
    resizeWidth.value = items.value[index].width;
    resizeHeight.value = items.value[index].height;
};

const applyResize = () => {
    if (resizingItem.value !== null) {
        items.value[resizingItem.value].width = resizeWidth.value;
        items.value[resizingItem.value].height = resizeHeight.value;
        saveItems();
        resizingItem.value = null;
    }
};

const cancelResize = () => {
    resizingItem.value = null;
};

const deleteItem = () => {
    if (resizingItem.value !== null) {
        items.value.splice(resizingItem.value, 1);
        saveItems();
        resizingItem.value = null;
    }
};
</script>

<style scoped>
.app-container {
    max-width: 1000px;
    margin: 0 auto;
    font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif;
    padding: 20px;
    color: #333;
}

.header {
    text-align: center;
    margin-bottom: 20px;
}

.controls-section {
    display: flex;
    flex-wrap: wrap;
    gap: 20px;
    justify-content: center;
    margin-bottom: 20px;
}

.control-group {
    display: flex;
    flex-direction: column;
    align-items: flex-start;
    gap: 10px;
}

.control-group label {
    font-size: 14px;
    color: #555;
}

.control-group input {
    padding: 5px;
    border: 1px solid #ccc;
    border-radius: 4px;
}

.control-group button {
    padding: 8px 16px;
    background-color: #3b82f6;
    color: white;
    border: none;
    border-radius: 4px;
    cursor: pointer;
}

.control-group button:hover {
    background-color: #2563eb;
}

.canvas-section {
    display: flex;
    justify-content: center;
    margin-bottom: 20px;
}

/* 캔버스 래퍼는 스케일된 캔버스 크기를 유지 */
.canvas-wrapper {
    overflow: hidden;
    border: 2px solid #ddd;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
}

/* 캔버스 기본 스타일은 그대로 유지 */
.canvas {
    position: relative;
    background-size: cover;
    background-position: center;
}

/* 리사이즈 모달 */
.resize-modal {
    position: fixed;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    background: rgba(0, 0, 0, 0.5);
    display: flex;
    align-items: center;
    justify-content: center;
    z-index: 1000;
}

.resize-content {
    background: #fff;
    padding: 30px;
    border-radius: 8px;
    width: 300px;
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
    text-align: center;
}

.resize-content h3 {
    margin-bottom: 20px;
}

.input-group {
    display: flex;
    flex-direction: column;
    gap: 10px;
    margin-bottom: 20px;
}

.input-group label {
    font-size: 14px;
    display: flex;
    flex-direction: column;
    align-items: flex-start;
}

.input-group input {
    padding: 5px;
    border: 1px solid #ccc;
    border-radius: 4px;
    width: 100%;
    margin-top: 5px;
}

.resize-buttons {
    display: flex;
    justify-content: center;
    gap: 20px;
}

.btn {
    padding: 8px 16px;
    border: none;
    border-radius: 4px;
    cursor: pointer;
}

.btn.apply {
    background-color: #10b981;
    color: white;
}

.btn.apply:hover {
    background-color: #059669;
}

.btn.cancel {
    background-color: #ef4444;
    color: white;
}

.btn.cancel:hover {
    background-color: #dc2626;
}

.btn.delete {
    background-color: #6b7280;
    color: white;
}

.btn.delete:hover {
    background-color: #4b5563;
}

/* 반응형 스타일 */
@media (max-width: 768px) {
    .app-container {
        padding: 10px;
    }

    .controls-section {
        flex-direction: column;
        align-items: center;
    }

    .canvas-wrapper {
        margin: 0 auto;
    }
}
</style>
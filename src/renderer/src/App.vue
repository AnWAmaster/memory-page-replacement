<script setup>
import { computed, ref, reactive } from 'vue'
import { ElMessage } from 'element-plus'
import 'element-plus/es/components/message/style/css'
//算法部分
const residentSetMaxSize = ref(0);
const visitedPageId = ref(0);
const pagesInMemory = ref([]);
const eliminatedPages = ref([]);
const pageFaultNum = ref(0);
const usePageNum = ref(0);
const nextEliminatedPageId = ref(-1);
const replaceAlgorithm = {
  algorithm: "",
  replace: (pageId) => {
    if (replaceAlgorithm.algorithm == "") {
      console.log("未选择置换算法");
      return;
    }
    let eliminatedPageId = replaceAlgorithm.algorithm.replace(pageId);
    return eliminatedPageId;
  }
};
const FIFO = {
  replace: (pageId) => {
    let hitIdx = -1;
    let eliminatedPageId = -1;
    //先检查是否命中
    for (let i = 0; i < pagesInMemory.value.length; i++) {
      if (pagesInMemory.value[i] == pageId) {
        hitIdx = i;
        break;
      }
    }
    //未命中(命中则不进行操作)
    if (hitIdx == -1) {
      //缺页+1
      pageFaultNum.value += 1;
      //长度超了就换出
      if (pagesInMemory.value.length == residentSetMaxSize.value) {
        eliminatedPageId = pagesInMemory.value.shift();
      }
      //再换入
      pagesInMemory.value.push(pageId);
    }
    nextEliminatedPageId.value = pagesInMemory.value[0];
    return eliminatedPageId;
  },
}
const LRUByCounter = {
  counter: reactive([]),
  replace: (pageId) => {
    let eliminatedPageId = -1;
    //先把每个计数器都加一
    for (let i = 0; i < LRUByCounter.counter.length; i++) {
      LRUByCounter.counter[i] = LRUByCounter.counter[i] + 1;
    }
    let hitIdx = -1;
    //先检查是否命中
    for (let i = 0; i < pagesInMemory.value.length; i++) {
      if (pagesInMemory.value[i] == pageId) {
        hitIdx = i;
        break;
      }
    }
    //命中
    if (hitIdx != -1) {
      //重置计数器
      LRUByCounter.counter[hitIdx] = 0;
    } else {
      //缺页+1
      pageFaultNum.value += 1;
      //长度超了就换出
      if (pagesInMemory.value.length == residentSetMaxSize.value) {
        let idx = 0;
        for (let i = 1; i < LRUByCounter.counter.length; i++) {
          if (LRUByCounter.counter[i] > LRUByCounter.counter[idx]) {
            idx = i;
          }
        }
        //赋一下编号
        eliminatedPageId = pagesInMemory.value[idx];
        //从内存中换出
        pagesInMemory.value.splice(idx, 1);
        //同时删除计数器
        LRUByCounter.counter.splice(idx, 1);
      }
      //再换入
      pagesInMemory.value.push(pageId);
      //为这个新页面增加一个计时器
      LRUByCounter.counter.push(0);
    }
    //切换高亮的页面块
    let idx = 0;
    for (let i = 1; i < LRUByCounter.counter.length; i++) {
      if (LRUByCounter.counter[i] > LRUByCounter.counter[idx]) {
        idx = i;
      }
    }
    nextEliminatedPageId.value = pagesInMemory.value[idx];
    return eliminatedPageId;
  },
}
const LRUByStack = {
  stack: reactive([]),
  replace: (pageId) => {
    let eliminatedPageId = -1;
    let hitIdx = -1;
    //先检查是否命中
    for (let i = 0; i < pagesInMemory.value.length; i++) {
      if (pagesInMemory.value[i] == pageId) {
        hitIdx = i;
        break;
      }
    }
    //命中
    if (hitIdx != -1) {
      //找到栈中的该页面
      let hitIdxInStack;
      for (let i = 0; i < LRUByStack.stack.length; i++) {
        if (LRUByStack.stack[i] == pageId) {
          hitIdxInStack = i;
          break;
        }
      }
      //将该页面放到栈底
      LRUByStack.stack.splice(hitIdxInStack, 1);
      LRUByStack.stack.push(pageId);
    } else {
      //缺页+1
      pageFaultNum.value += 1;
      //长度超了就换出
      if (pagesInMemory.value.length == residentSetMaxSize.value) {
        //换出栈顶
        eliminatedPageId = LRUByStack.stack.shift();
        //从内存中换出
        for (let i = 0; i < pagesInMemory.value.length; i++) {
          if (pagesInMemory.value[i] == eliminatedPageId) {
            pagesInMemory.value.splice(i, 1);
            break;
          }
        }
      }
      //再换入
      pagesInMemory.value.push(pageId);
      LRUByStack.stack.push(pageId);
    }
    //切换高亮的页面块
    nextEliminatedPageId.value = LRUByStack.stack[0];
    return eliminatedPageId;
  },
};
const checkAndRemoveEliminatedPage = (pageId) => {
  let idx = -1;
  for (let i = 0; i < eliminatedPages.value.length; i++) {
    if (eliminatedPages.value[i] == pageId) {
      idx = i;
      break;
    }
  }
  if (idx != -1) {
    eliminatedPages.value.splice(idx, 1);
  }
}
//获取缺页率
const getPageFaultFrequency = computed(() => {
  if (usePageNum.value == 0) { return 0; }
  return pageFaultNum.value / usePageNum.value;
})
const addPage = () => {
  // console.log(visitedPageId.value)
  //先增加数量
  usePageNum.value += 1;
  //移除淘汰队列中当前要插入的页面
  checkAndRemoveEliminatedPage(visitedPageId.value);
  //进行置换
  let eliminatedPageId = replaceAlgorithm.replace(visitedPageId.value);
  //未命中
  // console.log(eliminatedPageId)
  if (eliminatedPageId != -1) {
    //将淘汰页存入淘汰队列
    eliminatedPages.value.push(eliminatedPageId);
  }
  //重置
  visitedPageId.value = 0;
}
const algorithmOptions = [
  { name: "FIFO", value: FIFO },
  { name: "LRU-计数器", value: LRUByCounter },
  { name: "LRU-栈", value: LRUByStack },
]

//页面交互展示效果部分
//是否开始
const isStarted = ref(false);
//是否展示重置确认面板
const resetDialogVisible = ref(false);
//是否展示开始菜单确认面板
const startDialogVisible = ref(false);
//选择的算法名
const selectedAlgorithmName = ref("");
//高亮的、最容易被换出的页面的下标
const highlightIdx = ref(0);
//点击开始按钮触发
const handleStartBtnClick = () => {
  if (isStarted.value) {
    resetDialogVisible.value = true;
  } else {
    startDialogVisible.value = true;
  }
}
const start = () => {
  if (replaceAlgorithm.algorithm == "") {
    ElMessage({
      showClose: true,
      message: '未选择表单信息',
      type: 'error',
      duration: 2000
    })
    return;
  }
  isStarted.value = true;
  startDialogVisible.value = false;
}
//选择算法时触发
const handleSelectAlgorithm = (idx) => {
  // console.log(algorithmOptions[idx].value)
  replaceAlgorithm.algorithm = algorithmOptions[idx].value;
  selectedAlgorithmName.value = algorithmOptions[idx].name;
}


//重置
const resetAll = () => {
  //系统变量
  residentSetMaxSize.value = 0;
  visitedPageId.value = 0;
  pagesInMemory.value = [];
  eliminatedPages.value = [];
  pageFaultNum.value = 0;
  usePageNum.value = 0;
  nextEliminatedPageId.value = -1;
  //算法变量
  replaceAlgorithm.algorithm = "";
  LRUByCounter.counter = [];
  LRUByStack.stack = [];
  //页面变量
  isStarted.value = false;
  resetDialogVisible.value = false;
  startDialogVisible.value = false;
  selectedAlgorithmName.value = "";
  highlightIdx.value = 0;
}
const highlightStyle = {
  backgroundColor: '#f56c6c',
  color: 'white'
}

</script>

<template>
  <!-- 顶部按钮 -->
  <div style="text-align: center;">
    <span style="font-size: 25px; vertical-align: middle;margin-right: 5px;">页面置换算法实验</span>
    <el-button :type="isStarted ? 'danger' : 'primary'" @click="handleStartBtnClick">{{ isStarted ? "重置" : "开始"
    }}</el-button>
    <!-- 重置面板的确认框 -->
    <el-dialog center v-model="resetDialogVisible" title="重置确认" width="25%" align-center>
      <span>确认要重置面板吗?</span>
      <template #footer>
        <span>
          <el-button @click="resetDialogVisible = false">取消</el-button>
          <el-button type="danger" @click="isStarted = false; resetAll(); resetDialogVisible = false">
            确认
          </el-button>
        </span>
      </template>
    </el-dialog>
    <el-dialog center v-model="startDialogVisible" title="开始选项" width="45%" style="text-align: center;">
      <el-form label-position="top">
        <el-form-item label="页面置换算法">
          <el-select v-model="selectedAlgorithmName" placeholder="选择算法" @change="handleSelectAlgorithm">
            <el-option v-for="(item, idx) in algorithmOptions" :key="item.name" :label="item.name" :value="idx" />
          </el-select>
        </el-form-item>
        <el-form-item label="驻留集大小">
          <el-input-number v-model="residentSetMaxSize" :min="1" :max="10" />
        </el-form-item>
      </el-form>
      <template #footer>
        <span>
          <el-button @click="startDialogVisible = false">取消</el-button>
          <el-button type="primary" @click="start()">
            开始
          </el-button>
        </span>
      </template>
    </el-dialog>
  </div>
  <!-- 顶中部提示 -->
  <div style="font-size: 22px;;text-align: center;" v-if="isStarted">
    <span style="margin-right: 20px;">算法：{{ selectedAlgorithmName }}</span>
    <span style="">驻留集大小：{{ residentSetMaxSize }}</span>
  </div>
  <!-- 中部页面队列 -->
  <div v-if="isStarted">
    <!-- 内存中页面队列 -->
    <div class="queue-head">内存中页面(按进入页面顺序)</div>
    <div class="pages-queue">
      <div class="pages-item" v-for="(item, idx) in pagesInMemory"
        :style="nextEliminatedPageId == item ? highlightStyle : {}">
        {{ item }}
      </div>
    </div>
    <!-- 淘汰队列 -->
    <div class="queue-head">淘汰页面(按淘汰顺序)</div>
    <div class="pages-queue">
      <div class="pages-item" v-for="(item, idx) in eliminatedPages">
        {{ item }}
      </div>
    </div>
  </div>
  <!-- 中下部算法细节 -->
  <div v-if="isStarted">
    <!-- LRU-计数器 -->
    <div v-if="replaceAlgorithm.algorithm == LRUByCounter">
      <div class="queue-head">LRU计数器详情</div>
      <div class="pages-lru-counter-queue">
        <div class="pages-lru-counter-item" :style="nextEliminatedPageId == pagesInMemory[idx] ? highlightStyle : {}"
          v-for="(item, idx) in LRUByCounter.counter">
          <div>
            {{ pagesInMemory[idx] }}
          </div>
          <div class="counter">
            {{ item }}次
          </div>
        </div>
      </div>
    </div>
    <!-- LRU-栈 -->
    <div v-if="replaceAlgorithm.algorithm == LRUByStack">
      <div class="queue-head">LRU栈详情</div>
      <div class="queue-head">栈底</div>
      <div class="pages-queue">
        <div class="pages-item" :style="nextEliminatedPageId == item ? highlightStyle : {}"
          v-for="(item, idx) in LRUByStack.stack">
          {{ item }}
        </div>
      </div>
    </div>
  </div>
  <div style="display: flex;justify-content: center;" v-if="isStarted">
    <span class="queue-head">缺页数/操作次数：{{ pageFaultNum }}/{{ usePageNum }} 缺页率：{{ getPageFaultFrequency }}</span>
  </div>
  <!-- 下部表单 -->
  <div style="display: flex;justify-content: center;margin-top:10px" v-if="isStarted">
    <el-form label-position="top"
      style="padding:15px 15px 0 15px;border: 1px solid #cacdd1;border-radius: 10px;box-shadow: 0px 0px 10px 1pX #cacdd1;">
      <el-form-item label="输入访问页面编号">
        <el-input-number :value-on-clear="0" v-model="visitedPageId" :min="0" />
      </el-form-item>
      <el-form-item id="formBtnGroup" style="justify-content: center;">
        <el-button type="primary" @click="addPage()">访问</el-button>
        <el-button @click="visitedPageId = 0">重置</el-button>
      </el-form-item>
    </el-form>
  </div>
</template>
<style lang="less">
#formBtnGroup {
  .el-form-item__content {
    justify-content: center;
  }
}

.pages-queue {
  border: 1px solid #cacdd1;
  height: 70px;
  width: 100%;
  border-radius: 10px;

  .pages-item {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    margin: 5px;
    border: 1px solid #cacdd1;
    height: 60px;
    width: 70px;
    font-size: 28px;
    border-radius: 10px;
    box-shadow: 0px 0px 10px 1pX #cacdd1;
  }
}


.pages-lru-counter-queue {
  border: 1px solid #cacdd1;
  height: 70px;
  border-radius: 10px;
  width: 100%;
  overflow-x: auto;

  .pages-lru-counter-item {
    display: inline-flex;
    align-items: center;
    flex-direction: column;
    justify-content: center;
    margin: 5px;
    border: 1px solid #cacdd1;
    height: 60px;
    width: 70px;
    font-size: 28px;
    border-radius: 10px;
    box-shadow: 0px 0px 10px 1pX #cacdd1;

    .counter {
      font-size: 25px;
    }
  }
}


.queue-head {
  font-size: 27px;

}
</style>

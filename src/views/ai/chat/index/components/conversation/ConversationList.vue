<!--  AI 对话  -->
<template>
  <el-aside width="260px" class="conversation-container" style="height: 100%">
    <!-- 左顶部：对话 -->
    <div style="height: 100%">
      <el-button class="w-1/1 btn-new-conversation" type="primary" @click="createConversation">
        <Icon icon="ep:plus" class="mr-5px" />
        新建对话
      </el-button>

      <!-- 左顶部：搜索对话 -->
      <el-input
        v-model="searchName"
        size="large"
        class="mt-10px search-input"
        placeholder="搜索历史记录"
        @keyup="searchConversation"
      >
        <template #prefix>
          <Icon icon="ep:search" />
        </template>
      </el-input>

      <!-- 左中间：对话列表 -->
      <div class="conversation-list">
        <el-empty v-if="loading" description="." :v-loading="loading" />

        <div v-for="conversationKey in Object.keys(conversationMap)" :key="conversationKey">
          <div
            class="conversation-item classify-title"
            v-if="conversationMap[conversationKey].length"
          >
            <el-text class="mx-1" size="small" tag="b">{{ conversationKey }}</el-text>
          </div>
          <div
            class="conversation-item"
            v-for="conversation in conversationMap[conversationKey]"
            :key="conversation.id"
            @click="handleConversationClick(conversation.id)"
            @mouseover="hoverConversationId = conversation.id"
            @mouseout="hoverConversationId = ''"
          >
            <div
              :class="
                conversation.id === activeConversationId ? 'conversation active' : 'conversation'
              "
            >
              <div class="title-wrapper">
                <img class="avatar" :src="conversation.roleAvatar || roleAvatarDefaultImg" />
                <span class="title">{{ conversation.title }}</span>
              </div>
              <div class="button-wrapper" v-show="hoverConversationId === conversation.id">
                <el-button class="btn" link @click.stop="handlerTop(conversation)">
                  <el-icon title="置顶" v-if="!conversation.pinned"><Top /></el-icon>
                  <el-icon title="置顶" v-if="conversation.pinned"><Bottom /></el-icon>
                </el-button>
                <el-button class="btn" link @click.stop="updateConversationTitle(conversation)">
                  <el-icon title="编辑">
                    <Icon icon="ep:edit" />
                  </el-icon>
                </el-button>
                <el-button class="btn" link @click.stop="deleteChatConversation(conversation)">
                  <el-icon title="删除对话">
                    <Icon icon="ep:delete" />
                  </el-icon>
                </el-button>
              </div>
            </div>
          </div>
        </div>
        <!--  底部站位  -->
        <div style="height: 160px; width: 100%"></div>
      </div>
    </div>

    <!-- 左底部：工具栏 -->
    <!-- TODO @fan：下面两个 icon，可以使用类似 <Icon icon="ep:question-filled" /> 替代哈 -->
    <div class="tool-box">
      <div @click="handleRoleRepository">
        <Icon icon="ep:user" />
        <el-text size="small">角色仓库</el-text>
      </div>
      <div @click="handleClearConversation">
        <Icon icon="ep:delete" />
        <el-text size="small">清空未置顶对话</el-text>
      </div>
    </div>

    <!-- ============= 额外组件 ============= -->

    <!-- 角色仓库抽屉 -->
    <el-drawer v-model="drawer" title="角色仓库" size="754px">
      <Role />
    </el-drawer>
  </el-aside>
</template>

<script setup lang="ts">
import { ChatConversationApi, ChatConversationVO } from '@/api/ai/chat/conversation'
import { ref } from 'vue'
import Role from '../role/index.vue'
import { Bottom, Top } from '@element-plus/icons-vue'
import roleAvatarDefaultImg from '@/assets/ai/gpt.svg'

const message = useMessage() // 消息弹窗

// 定义属性
const searchName = ref<string>('') // 对话搜索
const activeConversationId = ref<string | null>(null) // 选中的对话，默认为 null
const hoverConversationId = ref<string | null>(null) // 悬浮上去的对话
const conversationList = ref([] as ChatConversationVO[]) // 对话列表
const conversationMap = ref<any>({}) // 对话分组 (置顶、今天、三天前、一星期前、一个月前)
const drawer = ref<boolean>(false) // 角色仓库抽屉 TODO @fan：roleDrawer 会不会好点哈
const loading = ref<boolean>(false) // 加载中
const loadingTime = ref<any>() // 加载中定时器

// 定义组件 props
const props = defineProps({
  activeId: {
    type: String || null,
    required: true
  }
})

// 定义钩子
const emits = defineEmits([
  'onConversationCreate',
  'onConversationClick',
  'onConversationClear',
  'onConversationDelete'
])

/**
 * 对话 - 搜索
 */
const searchConversation = async (e) => {
  // 恢复数据
  if (!searchName.value.trim().length) {
    conversationMap.value = await conversationTimeGroup(conversationList.value)
  } else {
    // 过滤
    const filterValues = conversationList.value.filter((item) => {
      return item.title.includes(searchName.value.trim())
    })
    conversationMap.value = await conversationTimeGroup(filterValues)
  }
}

/**
 * 对话 - 点击
 */
const handleConversationClick = async (id: string) => {
  // 过滤出选中的对话
  const filterConversation = conversationList.value.filter((item) => {
    return item.id === id
  })
  // 回调 onConversationClick
  // TODO @fan: 这里 idea 会报黄色警告，有办法解下么？
  const res = emits('onConversationClick', filterConversation[0])
  // 切换对话
  if (res) {
    activeConversationId.value = id
  }
}

/**
 * 对话 - 获取列表
 */
const getChatConversationList = async () => {
  try {
    // 0. 加载中
    loadingTime.value = setTimeout(() => {
      loading.value = true
    }, 50)
    // 1. 获取 对话数据
    const res = await ChatConversationApi.getChatConversationMyList()
    // 2. 排序
    res.sort((a, b) => {
      return b.createTime - a.createTime
    })
    conversationList.value = res
    // 3. 默认选中
    if (!activeId?.value) {
      // await handleConversationClick(res[0].id)
    } else {
      // tip: 删除的刚好是选中的，那么需要重新挑选一个来进行选中
      // const filterConversationList = conversationList.value.filter(item => {
      //   return item.id === activeId.value
      // })
      // if (filterConversationList.length <= 0) {
      //   await handleConversationClick(res[0].id)
      // }
    }
    // 4. 没有任何对话情况
    if (conversationList.value.length === 0) {
      activeConversationId.value = null
      conversationMap.value = {}
      return
    }
    // 5. 对话根据时间分组(置顶、今天、一天前、三天前、七天前、30天前)
    conversationMap.value = await conversationTimeGroup(conversationList.value)
  } finally {
    // 清理定时器
    if (loadingTime.value) {
      clearTimeout(loadingTime.value)
    }
    // 加载完成
    loading.value = false
  }
}

const conversationTimeGroup = async (list: ChatConversationVO[]) => {
  // 排序、指定、时间分组(今天、一天前、三天前、七天前、30天前)
  const groupMap = {
    置顶: [],
    今天: [],
    一天前: [],
    三天前: [],
    七天前: [],
    三十天前: []
  }
  // 当前时间的时间戳
  const now = Date.now()
  // 定义时间间隔常量（单位：毫秒）
  const oneDay = 24 * 60 * 60 * 1000
  const threeDays = 3 * oneDay
  const sevenDays = 7 * oneDay
  const thirtyDays = 30 * oneDay
  for (const conversation: ChatConversationVO of list) {
    // 置顶
    if (conversation.pinned) {
      groupMap['置顶'].push(conversation)
      continue
    }
    // 计算时间差（单位：毫秒）
    const diff = now - conversation.updateTime
    // 根据时间间隔判断
    if (diff < oneDay) {
      groupMap['今天'].push(conversation)
    } else if (diff < threeDays) {
      groupMap['一天前'].push(conversation)
    } else if (diff < sevenDays) {
      groupMap['三天前'].push(conversation)
    } else if (diff < thirtyDays) {
      groupMap['七天前'].push(conversation)
    } else {
      groupMap['三十天前'].push(conversation)
    }
  }
  console.log('----groupMap', groupMap)
  return groupMap
}

/**
 * 对话 - 新建
 */
const createConversation = async () => {
  // 1. 新建对话
  const conversationId = await ChatConversationApi.createChatConversationMy(
    {} as unknown as ChatConversationVO
  )
  // 2. 获取对话内容
  await getChatConversationList()
  // 3. 选中对话
  await handleConversationClick(conversationId)
  // 4. 回调
  emits('onConversationCreate')
}

/**
 * 对话 - 更新标题
 */
const updateConversationTitle = async (conversation: ChatConversationVO) => {
  // 1. 二次确认
  const { value } = await ElMessageBox.prompt('修改标题', {
    inputPattern: /^[\s\S]*.*\S[\s\S]*$/, // 判断非空，且非空格
    inputErrorMessage: '标题不能为空',
    inputValue: conversation.title
  })
  // 2. 发起修改
  await ChatConversationApi.updateChatConversationMy({
    id: conversation.id,
    title: value
  } as ChatConversationVO)
  message.success('重命名成功')
  // 3. 刷新列表
  await getChatConversationList()
  // 4. 过滤当前切换的
  const filterConversationList = conversationList.value.filter((item) => {
    return item.id === conversation.id
  })
  if (filterConversationList.length > 0) {
    // tip：避免切换对话
    if (activeConversationId.value === filterConversationList[0].id) {
      emits('onConversationClick', filterConversationList[0])
    }
  }
}

/**
 * 删除聊天对话
 */
const deleteChatConversation = async (conversation: ChatConversationVO) => {
  try {
    // 删除的二次确认
    await message.delConfirm(`是否确认删除对话 - ${conversation.title}?`)
    // 发起删除
    await ChatConversationApi.deleteChatConversationMy(conversation.id)
    message.success('对话已删除')
    // 刷新列表
    await getChatConversationList()
    // 回调
    emits('onConversationDelete', conversation)
  } catch {}
}

/**
 * 对话置顶
 */
// TODO @fan：应该是 handleXXX，handler 是名词哈
const handlerTop = async (conversation: ChatConversationVO) => {
  // 更新对话置顶
  conversation.pinned = !conversation.pinned
  await ChatConversationApi.updateChatConversationMy(conversation)
  // 刷新对话
  await getChatConversationList()
}

// TODO @fan:类似 ============ 分块的，最后后面也有 ============ 哈
// ============ 角色仓库

/**
 * 角色仓库抽屉
 */
const handleRoleRepository = async () => {
  drawer.value = !drawer.value
}

// ============= 清空对话

/**
 * 清空对话
 */
const handleClearConversation = async () => {
  // TODO @fan：可以使用 await message.confirm( 简化，然后使用 await 改成同步的逻辑，会更简洁
  ElMessageBox.confirm('确认后对话会全部清空，置顶的对话除外。', '确认提示', {
    confirmButtonText: '确认',
    cancelButtonText: '取消',
    type: 'warning'
  })
    .then(async () => {
      await ChatConversationApi.deleteChatConversationMyByUnpinned()
      ElMessage({
        message: '操作成功!',
        type: 'success'
      })
      // 清空 对话 和 对话内容
      activeConversationId.value = null
      // 获取 对话列表
      await getChatConversationList()
      // 回调 方法
      emits('onConversationClear')
    })
    .catch(() => {})
}

// ============ 组件 onMounted

const { activeId } = toRefs(props)
watch(activeId, async (newValue, oldValue) => {
  // 更新选中
  activeConversationId.value = newValue as string
})

// 定义 public 方法
defineExpose({ createConversation })

onMounted(async () => {
  // 获取 对话列表
  await getChatConversationList()
  // 默认选中
  if (props.activeId != null) {
    activeConversationId.value = props.activeId
  } else {
    // 首次默认选中第一个
    if (conversationList.value.length) {
      activeConversationId.value = conversationList.value[0].id
      // 回调 onConversationClick
      await emits('onConversationClick', conversationList.value[0])
    }
  }
})
</script>

<style scoped lang="scss">
.conversation-container {
  position: relative;
  display: flex;
  flex-direction: column;
  justify-content: space-between;
  padding: 10px 10px 0;
  overflow: hidden;

  .btn-new-conversation {
    padding: 18px 0;
  }

  .search-input {
    margin-top: 20px;
  }

  .conversation-list {
    overflow: auto;
    height: 100%;

    .classify-title {
      padding-top: 10px;
    }

    .conversation-item {
      margin-top: 5px;
    }

    .conversation {
      display: flex;
      flex-direction: row;
      justify-content: space-between;
      flex: 1;
      padding: 0 5px;
      cursor: pointer;
      border-radius: 5px;
      align-items: center;
      line-height: 30px;

      &.active {
        background-color: #e6e6e6;

        .button {
          display: inline-block;
        }
      }

      .title-wrapper {
        display: flex;
        flex-direction: row;
        align-items: center;
      }

      .title {
        padding: 2px 10px;
        max-width: 220px;
        font-size: 14px;
        font-weight: 400;
        color: rgba(0, 0, 0, 0.77);
        overflow: hidden;
        white-space: nowrap;
        text-overflow: ellipsis;
      }

      .avatar {
        width: 25px;
        height: 25px;
        border-radius: 5px;
        display: flex;
        flex-direction: row;
        justify-items: center;
      }

      // 对话编辑、删除
      .button-wrapper {
        right: 2px;
        display: flex;
        flex-direction: row;
        justify-items: center;
        color: #606266;

        .btn {
          margin: 0;
        }
      }
    }
  }

  // 角色仓库、清空未设置对话
  .tool-box {
    position: absolute;
    bottom: 0;
    left: 0;
    right: 0;
    //width: 100%;
    padding: 0 20px;
    background-color: #f4f4f4;
    box-shadow: 0 0 1px 1px rgba(228, 228, 228, 0.8);
    line-height: 35px;
    display: flex;
    justify-content: space-between;
    align-items: center;
    color: var(--el-text-color);

    > div {
      display: flex;
      align-items: center;
      color: #606266;
      padding: 0;
      margin: 0;
      cursor: pointer;

      > span {
        margin-left: 5px;
      }
    }
  }
}
</style>
<template>
  <div id="pictureManagePage">
    <a-flex justify="space-between">
      <h2>图片管理</h2>
      <a-space>
        <a-button type="primary" href="/add_picture" target="_blank">+ 创建图片</a-button>
        <a-button type="primary" href="/add_picture/batch" target="_blank" ghost>+ 批量创建图片</a-button>
        <!-- 批量编辑按钮 -->
        <a-button
            type="primary"
            :disabled="!selectedRowKeys.length"
            @click="showBatchModal"
            ghost
        >
          <template #icon><EditOutlined /></template>
          批量编辑
        </a-button>
      </a-space>
    </a-flex>
    <div style="margin-bottom: 16px" />
    <!-- 批量编辑弹窗 -->
    <template>
      <a-modal
          v-model:visible="batchVisible"
          title="批量编辑图片"
          ok-text="提交"
          cancel-text="取消"
          @ok="handleBatchEdit"
          @cancel="resetBatchForm"
      >
        <a-form layout="vertical">
          <a-form-item label="分类">
            <a-auto-complete
                v-model:value="batchForm.category"
                placeholder="请选择或输入分类"
                :options="categoryOptions"
                allow-clear
            />
          </a-form-item>

          <a-form-item label="标签">
            <a-select
                v-model:value="batchForm.tags"
                mode="tags"
                placeholder="请选择标签"
                :options="tagOptions"
                :token-separators="[',']"
                allow-clear
            />
          </a-form-item>

          <a-form-item label="命名规则">
            <a-input
                v-model:value="batchForm.nameRule"
                placeholder="支持变量：{index} 序号，{timestamp} 时间戳"
            />
          </a-form-item>

          <a-form-item label="已选图片（共 {{ selectedRowKeys.length }} 张）">
            <div class="selected-ids">
              <a-tag
                  v-for="id in selectedRowKeys"
                  :key="id"
                  closable
                  @close="removeSelectedId(id)"
              >
                {{ id }}
              </a-tag>
            </div>
          </a-form-item>
        </a-form>
      </a-modal>
    </template>
    <!-- 搜索表单 -->
    <a-form layout="inline" :model="searchParams" @finish="doSearch">
      <a-form-item label="关键词">
        <a-input
          v-model:value="searchParams.searchText"
          placeholder="从名称和简介搜索"
          allow-clear
        />
      </a-form-item>
      <a-form-item label="类型">
        <a-input v-model:value="searchParams.category" placeholder="请输入类型" allow-clear />
      </a-form-item>
      <a-form-item label="标签">
        <a-select
          v-model:value="searchParams.tags"
          mode="tags"
          placeholder="请输入标签"
          style="min-width: 180px"
          allow-clear
        />
      </a-form-item>
      <a-form-item name="reviewStatus" label="审核状态">
        <a-select
          v-model:value="searchParams.reviewStatus"
          style="min-width: 180px"
          placeholder="请选择审核状态"
          :options="PIC_REVIEW_STATUS_OPTIONS"
          allow-clear
        />
      </a-form-item>
      <a-form-item>
        <a-button type="primary" html-type="submit">搜索</a-button>
      </a-form-item>
    </a-form>
    <div style="margin-bottom: 16px" />
    <!-- 表格 -->
    <a-table
        :columns="columns"
        :data-source="dataList"
        :pagination="pagination"
        :row-selection="rowSelection"
        @change="doTableChange"
        rowKey="id"
    >
      <template #bodyCell="{ column, record }">
        <template v-if="column.dataIndex === 'url'">
          <a-image :src="record.url" :width="120" />
        </template>
        <template v-if="column.dataIndex === 'tags'">
          <a-space wrap>
            <a-tag v-for="tag in JSON.parse(record.tags || '[]')" :key="tag">
              {{ tag }}
            </a-tag>
          </a-space>
        </template>
        <template v-if="column.dataIndex === 'picInfo'">
          <div>格式：{{ record.picFormat }}</div>
          <div>宽度：{{ record.picWidth }}</div>
          <div>高度：{{ record.picHeight }}</div>
          <div>宽高比：{{ record.picScale }}</div>
          <div>大小：{{ (record.picSize / 1024).toFixed(2) }}KB</div>
        </template>
        <template v-if="column.dataIndex === 'reviewMessage'">
          <div>审核状态：{{ PIC_REVIEW_STATUS_MAP[record.reviewStatus] }}</div>
          <div>审核信息：{{ record.reviewMessage }}</div>
          <div>审核人：{{ record.reviewerId }}</div>
          <div v-if="record.reviewTime">
            审核时间：{{ dayjs(record.reviewTime).format('YYYY-MM-DD HH:mm:ss') }}
          </div>
        </template>
        <template v-if="column.dataIndex === 'createTime'">
          {{ dayjs(record.createTime).format('YYYY-MM-DD HH:mm:ss') }}
        </template>
        <template v-if="column.dataIndex === 'editTime'">
          {{ dayjs(record.editTime).format('YYYY-MM-DD HH:mm:ss') }}
        </template>
        <template v-else-if="column.key === 'action'">
          <a-space wrap>
            <a-button
              v-if="record.reviewStatus !== PIC_REVIEW_STATUS_ENUM.PASS"
              type="link"
              @click="handleReview(record, PIC_REVIEW_STATUS_ENUM.PASS)"
            >
              通过
            </a-button>
            <a-button
              v-if="record.reviewStatus !== PIC_REVIEW_STATUS_ENUM.REJECT"
              type="link"
              danger
              @click="handleReview(record, PIC_REVIEW_STATUS_ENUM.REJECT)"
            >
              拒绝
            </a-button>
            <a-button type="link" :href="`/add_picture?id=${record.id}`" target="_blank">
              编辑
            </a-button>
            <a-button danger @click="doDelete(record.id)">删除</a-button>
          </a-space>
        </template>
      </template>
    </a-table>
  </div>
</template>
<script lang="ts" setup>
// 新增接口和响应式数据
import { listPictureTagCategoryUsingGet } from '@/api/pictureController.ts'
import { computed, onMounted, reactive, ref } from 'vue'
import {
  deletePictureUsingPost,
  doPictureReviewUsingPost,
  listPictureByPageUsingPost,
  editPictureByBatchUsingPost
} from '@/api/pictureController.ts'
import { message } from 'ant-design-vue'
import {
  PIC_REVIEW_STATUS_ENUM,
  PIC_REVIEW_STATUS_MAP,
  PIC_REVIEW_STATUS_OPTIONS,
} from '../../constants/picture.ts'
import dayjs from 'dayjs'
import { EditOutlined } from '@ant-design/icons-vue'

// 表格列配置
const columns = [
  {
    title: 'id',
    dataIndex: 'id',
    width: 80,
  },
  {
    title: '图片',
    dataIndex: 'url',
  },
  {
    title: '名称',
    dataIndex: 'name',
  },
  {
    title: '简介',
    dataIndex: 'introduction',
    ellipsis: true,
  },
  {
    title: '类型',
    dataIndex: 'category',
  },
  {
    title: '标签',
    dataIndex: 'tags',
  },
  {
    title: '图片信息',
    dataIndex: 'picInfo',
  },
  {
    title: '用户 id',
    dataIndex: 'userId',
    width: 80,
  },
  {
    title: '审核信息',
    dataIndex: 'reviewMessage',
  },
  {
    title: '创建时间',
    dataIndex: 'createTime',
  },
  {
    title: '编辑时间',
    dataIndex: 'editTime',
  },
  {
    title: '操作',
    key: 'action',
  },
]

// 定义数据
const dataList = ref<API.Picture[]>([])
const total = ref(0)

const categoryOptions = ref<{ value: string; label: string }[]>([])
const tagOptions = ref<{ value: string; label: string }[]>([])
// 获取标签分类选项
const getTagCategoryOptions = async () => {
  try {
    const res = await listPictureTagCategoryUsingGet()
    if (res.data.code === 0 && res.data.data) {
      categoryOptions.value = (res.data.data.categoryList ?? []).map(category => ({
        value: category,
        label: category
      }))
      tagOptions.value = (res.data.data.tagList ?? []).map(tag => ({
        value: tag,
        label: tag
      }))
    }
  } catch (e) {
    message.error('获取预置分类标签失败')
  }
}
// 初始化获取数据
onMounted(() => {
  getTagCategoryOptions()
})
// 搜索条件
const searchParams = reactive<API.PictureQueryRequest>({
  current: 1,
  pageSize: 10,
  sortField: 'createTime',
  sortOrder: 'descend',
})

// 批量编辑相关状态
const batchVisible = ref(false)
const selectedRowKeys = ref<string[]>([])
const batchForm = reactive({
  category: '',
  nameRule: '',
  tags: [] as string[],
})

// 表格行选择配置
const rowSelection = reactive({
  selectedRowKeys: selectedRowKeys,
  onChange: (keys: string[]) => {
    selectedRowKeys.value = keys
  },
  checkStrictly: false,
})

// 获取数据
const fetchData = async () => {
  const res = await listPictureByPageUsingPost({
    ...searchParams,
  })
  if (res.data.code === 0 && res.data.data) {
    dataList.value = res.data.data.records ?? []
    total.value = res.data.data.total ?? 0
  } else {
    message.error('获取数据失败，' + res.data.message)
  }
}

// 页面加载时获取数据，请求一次
onMounted(() => {
  fetchData()
})

// 分页参数
const pagination = computed(() => {
  return {
    current: searchParams.current,
    pageSize: searchParams.pageSize,
    total: total.value,
    showSizeChanger: true,
    showTotal: (total) => `共 ${total} 条`,
  }
})

// 表格变化之后，重新获取数据
const doTableChange = (page: any) => {
  searchParams.current = page.current
  searchParams.pageSize = page.pageSize
  fetchData()
}

// 显示批量编辑弹窗
const showBatchModal = () => {
  if (selectedRowKeys.value.length === 0) {
    message.warning('请至少选择一张图片')
    return
  }
  batchVisible.value = true
}

// 处理批量编辑
const handleBatchEdit = async () => {
  try {
    // 构建请求参数
    const requestData = {
      category: batchForm.category || undefined, // 空字符串不传
      nameRule: batchForm.nameRule || undefined,
      tags: batchForm.tags.length ? batchForm.tags : undefined,
      pictureIdList: selectedRowKeys.value.map(id => BigInt(id).toString())
    }

    const res = await editPictureByBatchUsingPost(requestData)

    if (res.data.code === 0) {
      message.success(`成功批量更新 ${selectedRowKeys.value.length} 张图片`)
      // 重置状态
      batchVisible.value = false
      selectedRowKeys.value = []
      resetBatchForm()
      // 刷新数据
      fetchData()
    } else {
      message.error(res.data.message || '批量更新失败')
    }
  } catch (e) {
    message.error('请求失败，请检查网络连接')
  }
}

// 移除单个选中的ID
const removeSelectedId = (id: string) => {
  selectedRowKeys.value = selectedRowKeys.value.filter(item => item !== id)
}

// 重置表单
const resetBatchForm = () => {
  batchForm.category = ''
  batchForm.nameRule = ''
  batchForm.tags = []
}

// 搜索数据
const doSearch = () => {
  // 重置页码
  searchParams.current = 1
  fetchData()
}

// 删除数据
const doDelete = async (id: string) => {
  if (!id) {
    return
  }
  const res = await deletePictureUsingPost({ id })
  if (res.data.code === 0) {
    message.success('删除成功')
    // 刷新数据
    fetchData()
  } else {
    message.error('删除失败')
  }
}

// 审核图片
const handleReview = async (record: API.Picture, reviewStatus: number) => {
  const reviewMessage =
      reviewStatus === PIC_REVIEW_STATUS_ENUM.PASS ? '管理员操作通过' : '管理员操作拒绝'
  const res = await doPictureReviewUsingPost({
    id: record.id,
    reviewStatus,
    reviewMessage,
  })
  if (res.data.code === 0) {
    message.success('审核操作成功')
    // 重新获取列表数据
    fetchData()
  } else {
    message.error('审核操作失败，' + res.data.message)
  }
}
</script>

<style scoped>
#pictureManagePage {
  padding: 24px;
}

.selected-id-list {
  max-height: 200px;
  overflow-y: auto;
  padding: 8px;
  border: 1px solid #f0f0f0;
  border-radius: 4px;
}

.selected-id-list .ant-tag {
  margin: 4px;
  background: #f5f5f5;
  border: 1px dashed #ddd;
}

.empty-tip {
  color: #999;
  padding: 8px 0;
}
</style>

<template>
  <div class="user-update-page">
    <a-card title="修改个人信息" style="width: 600px">
      <a-form
          :model="formState"
          name="userUpdate"
          :label-col="{ span: 4 }"
          :wrapper-col="{ span: 18 }"
          @finish="onFinish"
      >
        <a-form-item label="用户头像">
          <a-upload name="file"
          list-type="picture-card"
          class="avatar-uploader"
          :show-upload-list="false"
          :before-upload="beforeUpload"
          :customRequest="customUpload"
          >
            <img
                v-if="formState.userAvatar"
                :src="formState.userAvatar"
                alt="avatar"
                style="width: 100%"
            />
            <div v-else>
              <plus-outlined />
              <div class="ant-upload-text">上传头像</div>
            </div>
          </a-upload>
        </a-form-item>

        <a-form-item label="用户名" name="userName">
          <a-input v-model:value="formState.userName" />
        </a-form-item>

        <a-form-item label="个人简介" name="userProfile">
          <a-textarea
              v-model:value="formState.userProfile"
              :auto-size="{ minRows: 2, maxRows: 5 }"
          />
        </a-form-item>

        <a-form-item :wrapper-col="{ offset: 4, span: 18 }">
          <a-button type="primary" html-type="submit" :loading="submitting">
            提交修改
          </a-button>
        </a-form-item>
      </a-form>
    </a-card>
  </div>
</template>

<script lang="ts" setup>
import { reactive, ref, onMounted } from 'vue'
import { message } from 'ant-design-vue'
import { PlusOutlined } from '@ant-design/icons-vue'
import { useLoginUserStore } from '@/stores/useLoginUserStore.ts'
import { updateMyUserUsingPost, uploadAvatarUsingPost  } from '@/api/userController.ts'

const loginUserStore = useLoginUserStore()
const submitting = ref(false)

// 表单数据
const formState = reactive({
  userAvatar: '',
  userName: '',
  userProfile: ''
})

// 初始化表单数据
onMounted(() => {
  const user = loginUserStore.loginUser
  formState.userAvatar = user.userAvatar || ''
  formState.userName = user.userName || ''
  formState.userProfile = user.userProfile || ''
})

// 文件上传相关
// const fileList = ref([])
// const beforeUpload = (file: File) => {
//   const isImage = file.type.startsWith('image/')
//   if (!isImage) {
//     message.error('只能上传图片文件!')
//   }
//   return isImage
// }

// 处理上传变化
const customUpload = async ({ file, onSuccess, onError }: any) => {
  const formData = new FormData()
  formData.append('file', file) // 确保参数名完全一致

  try {
    // 直接使用文件对象调用接口
    const res = await uploadAvatarUsingPost(file)
    if (res.data.code === 0 && res.data.data) {
      formState.userAvatar = res.data.data
      onSuccess()
      message.success('头像上传成功')
    }
  } catch (e) {
    onError(e)
    message.error('上传失败，请检查控制台')
    console.error('详细错误：', e)
  }
}

// 完善文件验证
const beforeUpload = (file: File) => {
  const isValidType = ['image/jpeg', 'image/png'].includes(file.type)
  const isValidSize = file.size / 1024 / 1024 < 2

  if (!isValidType) {
    message.error('仅支持JPG/PNG格式!')
    return false
  }
  if (!isValidSize) {
    message.error('文件大小不能超过2MB!')
    return false
  }
  return true
}

// 提交表单
const onFinish = async () => {
  submitting.value = true
  try {
    const res = await updateMyUserUsingPost(formState)
    if (res.data.code === 0) {
      message.success('修改成功')
      // 更新本地存储的用户信息
      loginUserStore.setLoginUser({
        ...loginUserStore.loginUser,
        ...formState
      })
    } else {
      message.error(res.data.message || '修改失败')
    }
  } catch (e) {
    message.error('请求失败，请检查网络')
  } finally {
    submitting.value = false
  }
}
</script>

<style scoped>
.user-update-page {
  padding: 24px;
  display: flex;
  justify-content: center;
}

.avatar-uploader :deep(.ant-upload) {
  width: 128px;
  height: 128px;
}
</style>

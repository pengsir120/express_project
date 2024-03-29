<template>
  <a-modal 
    v-model:open="visible" 
    @ok="handleOk" 
    @cancel="handleCancel" 
    title="创建视频" 
    :zIndex="2000"
    okText="创建"
    cancelText="取消"
  >
    <a-form
      ref="formRef"
      :model="form"
      :rules="rules"
      :label-col="labelCol"
      :wrapper-col="wrapperCol"
      layout="horizontal"
      style="max-width: 600px"
    >
      <a-form-item label="标题" name="title">
        <a-input v-model:value="form.title" />
      </a-form-item>
      <a-form-item label="时长" name="duration">
        <a-input disabled v-model:value="form.duration" />
      </a-form-item>
      <a-form-item label="上传视频">
        <a-upload v-model:file-list="fileList" :customRequest="uploadVideo" @remove="uploadRemove" list-type="picture-card">
          <div v-if="fileList.length < 1">
            <PlusOutlined />
            <div style="margin-top: 8px">Upload</div>
          </div>
        </a-upload>
      </a-form-item>
      <a-form-item label="上传封面">
        <a-upload v-model:file-list="picFileList" :customRequest="uploadPic" @preview="previewVisible = true" list-type="picture-card">
          <div v-if="picFileList.length < 1">
            <PlusOutlined />
            <div style="margin-top: 8px">Upload</div>
          </div>
        </a-upload>
      </a-form-item>
    </a-form>
    <a-modal :open="previewVisible" :footer="null" @cancel="previewVisible = false" :zIndex="2024">
      <img style="width: 100%" :src="previewImage" />
    </a-modal>
  </a-modal>
</template>

<script setup>
import { ref } from "vue"
import { PlusOutlined } from '@ant-design/icons-vue';
import { getVideoHash, captureFrame } from '@/utils/getVideoInfo'
import userGetGlobalProperties from '@/utils/userGetGlobalProperties'

const { $bus, $request } = userGetGlobalProperties()

const labelCol = { style: { width: '150px' } };
const wrapperCol = { span: 14 };
const visible = ref(false)
const previewVisible = ref(false)
const form = ref({})
const fileList = ref([])
const picFileList = ref([])
const formRef = ref()
const rules = ref({
  title: [{ required: true, message: '请输入标题', trigger: 'change' }],
  duration: [{ required: true, message: '请上传视频', trigger: 'change' }],
})

const show = () => {
  visible.value = true
}
const handleCancel = () => {
  form.value = {}
  fileList.value = []
  picFileList.value = []
  formRef.value.resetFields()
  visible.value = false
}
const uploadRemove = () => {
  form.value.duration = undefined
}

const handleOk = () => {
  formRef.value.validate().then(() => {
    $request({
      url: '/video/createvideo',
      method: 'POST',
      data: form.value
    }).then(res => {
      if(res.status == 201) {
        $bus.emit('getVideoList')
        handleCancel()
      }
    })
  })
  
}



const uploadVideo = async ({file, onProgress, onSuccess}) => {
  const video = document.createElement("video")
  video.preload = 'metadata'
  video.src = URL.createObjectURL(file)
  video.onloadedmetadata = async () => {
    // 视频时长
    URL.revokeObjectURL(video.src)
    // const duration = getVideoTime(video.duration)
    // form.value.duration = duration
    form.value.duration = video.duration

    // 视频封面
    if(picFileList.value.length == 0) {
      const coverInfo = await captureFrame(file, Math.random() * video.duration)
      const picMd5 = await getVideoHash(coverInfo.file)
      const picFormData = new FormData()
      picFormData.append("fileHash", picMd5)
      picFormData.append('file', coverInfo.file)
      const picRes = await $request({
        url: "/video/upload",
        method: "post",
        data: picFormData,
        headers: {
          'Content-Type': 'multipart/form-data'
        }
      })
      const picResUrl = picRes.data.url
      form.value.cover = picResUrl
      picFileList.value = [{
        uid: -1,
        name: coverInfo.name,
        status: 'done',
        url: picResUrl
      }]
      previewImage.value = picResUrl || coverInfo.url
    }

    const md5 = await getVideoHash(file)
    const videoFormData = new FormData()
    videoFormData.append("file", file)
    videoFormData.append("fileHash", md5)
    const res = await $request({
      url: "/video/upload",
      method: "post",
      data: videoFormData,
      headers: {
        'Content-Type': 'multipart/form-data'
      }
    })
    form.value.url = res.data.url
    onSuccess()
  }
}

const previewImage = ref('')
const uploadPic = async ({file, onProgress, onSuccess}) => {
  const picFormData = new FormData()
  picFormData.append('file', file)
  const res = await $request({
    url: "/video/upload",
    method: "post",
    data: picFormData,
    headers: {
      'Content-Type': 'multipart/form-data'
    }
  })
  previewImage.value = res.data.url
  form.value.cover = res.data.url
  onSuccess()
}

defineExpose({
  show
})
</script>
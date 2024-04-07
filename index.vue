<template>
  <div class="upload_page">
    <div v-for="(item,index) in fileList" :key='item.id' class="upload_item">
          <el-image style="width: 100px; height: 100px" :src="item.url" fit="cover" />
          <el-tooltip :content="item.originalName || item.name" placement="bottom">
            <div class="upload_item_name">{{item.originalName}}</div>
          </el-tooltip>
          <span class="el-upload-list__item_mask">
            <span
              v-if="suffix.image.includes(item.extension)"
              class="el-upload-list__item_icon"
              @click="handlePictureCardPreview(index)"
            >
              <el-icon :size="iconSize" :color="iconColor"><ZoomIn /></el-icon>
            </span>
            <span
              v-if="suffix.video.includes(item.extension)"
              class="el-upload-list__item_icon"
              @click="handleVideoPreview(item)"
            >
              <el-icon :size="iconSize" :color="iconColor"><ZoomIn /></el-icon>
            </span>
            <span
              v-if="suffix.audio.includes(item.extension)"
              class="el-upload-list__item_icon"
              @click="handleAudioPreview(item)"
            >
              <el-icon :size="iconSize" :color="iconColor"><ZoomIn /></el-icon>
            </span>
            <span
              class="el-upload-list__item_icon" 
              @click="handleDownload(item)"
            >
              <el-icon :size="iconSize" :color="iconColor"><Download /></el-icon>
            </span>
            <span
              class="el-upload-list__item_icon"
              @click="handleRemove(item)"
            >
              <el-icon :size="iconSize" :color="iconColor"><Delete /></el-icon>
            </span>
          </span>
    </div>

    <el-dialog v-model="dialogVideoVisible" :title="mediaTitle" width="800" @close="videoClose">
      <video ref="videoEle" :src="mediaUrl" height="500px" controls width="100%" autoplay></video>
    </el-dialog>
    <el-dialog v-model="dialogAudioVisible" :title="mediaTitle" width="20%" @close="audioClose">
      <audio ref="audioEle" :src="mediaUrl" controls width="100%" autoplay></audio>
    </el-dialog>

    <el-upload
        ref="Uploader"
        action="upload"
        :auto-upload="false"
        list-type="picture-card"
        :name="uploadId"
        multiple
        :accept="accept"
        :show-file-list="false"
        :before-upload="handleBeforeUpload"
        :http-request="httpRequest"
        :on-change="handleChange"
        :on-remove="handleRemove"
    >
      <el-icon><Plus /></el-icon>
    </el-upload>
  </div>
</template>

<script  setup>
import { fileUpload,fileDownload } from '@/api/common';
import { reactive, ref, onMounted, watch, defineEmits, defineProps } from 'vue';
import { ElMessage, ElMessageBox } from 'element-plus';
import { api as viewerApi } from "v-viewer";
import doc from './images/doc.png';
import pdf from './images/pdf.png';
import txt from './images/txt.png';
import xls from './images/xls.png';
import rar from './images/rar.png';
import zip from './images/zip.png';
import other from './images/other.png';
import video from './images/video.png';
import audio from './images/audio.png';
import "viewerjs/dist/viewer.css";

// 定义组件的 props
const props = defineProps({
    name: { type: String, required: false, default: 'files' }, // 上传字段的名称
    images: { type: Array, required: false, default: () => [] }, // 图片数组
    size: { type: Number, required: false, default: 100 }, // 文件大小
    imageUrlKey: { type: String, required: false, default: () => "url" }, // 预览文件的 key
    accept: { type: String, required: false, default: () => "" } // 上传文件的接受类型
});


// 定义事件
const emit = defineEmits(['uploadSuccess']);
const upload = (data)=>{
    emit("uploadSuccess",data)
}

// 上传文件相关的一些状态和方法
const iconSize = ref('20'); // 图标大小
const iconColor = ref('#fff'); // 图标颜色
const uploadId = ref(Math.random().toString(36).substr(2).toLocaleUpperCase()); // 上传组件的唯一标识
const fileList = ref([]); // 文件列表
const fm = ref(new FormData()); // FormData 对象
const uploadFiles = ref([]); // 上传的文件列表
const fileTotal = ref(0); // 文件总数
const Uploader = ref(null); // 上传组件的引用
const $viewer = ref(null); // Viewer 实例

// 文件预览相关状态和方法
const videoEle = ref(null); // 视频元素
const audioEle = ref(null); // 音频元素
const mediaTitle = ref(''); // 媒体标题
const mediaUrl = ref(''); // 媒体 URL
const dialogVideoVisible = ref(false); // 视频预览弹窗是否可见
const dialogAudioVisible = ref(false); // 音频预览弹窗是否可见

/**
 * 监听文件初始化
 */
watch(() => props.images,(nvl,avl)=>{
  fileList.value = nvl.map(item => item);
},{ deep: true, immediate: true })

/**
 * 视频预览弹窗关闭
 */
const videoClose = (file) =>{
  videoEle.value.pause()
  dialogVideoVisible.value = false
}
/**
 * 音频预览弹窗关闭
 */
const audioClose = (file) =>{
  audioEle.value.pause()
  dialogAudioVisible.value = false
}

/**
 * 视频预览
 */
const handleVideoPreview = (file) =>{
  mediaTitle.value = file.originalName
  mediaUrl.value = file.domainUrl + file.link.replace(/^\/{2}/, '')
  dialogVideoVisible.value = true
}

/**
 * 音频预览
 */
const handleAudioPreview = (file) =>{
  mediaTitle.value = file.originalName
  mediaUrl.value = file.domainUrl + file.link.replace(/^\/{2}/, '')
  dialogAudioVisible.value = true
}

/**
 * 图片预览
 */
const handlePictureCardPreview = (index) => {
  // 存在多类型的文件,过滤出图片做预览
  const imageList = fileList.value.filter(item=>suffix.value.image.includes(item.extension));
  // 取到点击预览的文件的id  (可自行调整)
  const fileIndexsId = fileList.value[index].id
  // 取到过滤后的预览的文件的索引  (可自行调整)
  const fileIndexs = imageList.findIndex(item=>item?.id === fileIndexsId)
  
  if (!imageList || imageList.length === 0) {
    console.error("请传入正确的图片数组");
    return;
  }
  const previewConfig = {
    options: {
      toolbar: true,
      initialViewIndex: fileIndexs
    },
    images: imageList
  };
  if (imageList[0][props.imageUrlKey]) {
    previewConfig.options.url = props.imageUrlKey;
  }
  $viewer.value = viewerApi(previewConfig);
}

const fileHandlers = {
  // 业务图片是根据后台返回的ip跟相对地址组成，可自行更改
  image: file => file.domainUrl + file.link.replace(/^\/{2}/, ''),
  xls: () => window.location.origin + xls,
  doc: () => window.location.origin + doc,
  pdf: () => window.location.origin + pdf,
  txt: () => window.location.origin + txt,
  zip: () => window.location.origin + zip,
  rar: () => window.location.origin + rar,
  video: () => window.location.origin + video,
  audio: () => window.location.origin + audio,
};

const suffix = ref({
  image: ['jpg', 'jpeg', 'png', 'gif'], // 图片类型
  xls: ['xls', 'xlsx'], // Excel 类型
  doc: ['doc', 'docx'], // Word 类型
  pdf: ['pdf'], // PDF 类型
  txt: ['txt'], // 文本类型
  zip: ['zip'], // 压缩包类型
  rar: ['rar'], // RAR 类型
  video: ['avid', 'mp4', 'ogv', 'webm', 'mov'], // 视频类型
  audio: ['mp3', 'midi', 'cd'], // 音频类型
})

/**
 * 文件类型处理方法，根据文件类型返回对应的处理 URL
 */
const checkSuffix = (file) => {
  let fileType
  for (let handlerType in suffix.value) {
    if (suffix.value[handlerType].includes(file.extension)) {
      fileType =  fileHandlers[handlerType](file);
      break
    }
  }
  return fileType || ( window.location.origin + other )
};

/**
 * 文件上传处理
 */
const httpRequest =  async (file) => {
   fm.value.append(props.name, file.file);
   //当fm getall的数组长度与filetotal的长度一致，文件准备完成
   if (fm.value.getAll(props.name).length === fileTotal.value) {
       try {
          const { data } = await fileUpload(fm.value);
          fileList.value.push(...data.data.map(item => ({ ...item, url: checkSuffix(item) })));
          upload(fileList.value);
       } catch (error) {
          console.log(`上传文件出错`, error);
       } finally {
          //无论成功与失败都要清空文件列表!!
          uploadFiles.value = [];
          fm.value.delete(props.name);
       }
   }
}

 /**
  * 上传前回调
  */
const handleBeforeUpload = (file) => {
   const isSize = file.size / 1024 / 1024 < props.size;
   if (!isSize) {
      ElMessage({
        message: "上传图片大小不能超过 4MB!",
        type: 'error',
      });
   }
   return isSize;
}

/**
 * 文件上传change
 */
const handleChange = (file,) => {
    //获取饿了么添加文件进来时的状态
    (file.status == 'ready') && uploadFiles.value.push(file.raw);
    //获取原始文件的个数
    fileTotal.value = document.getElementsByName(uploadId.value)[0].files.length;
    //如果原始文件和上传个数相同的时候就说明已经全部添加完成
    if (uploadFiles.value.length === fileTotal.value) {
        const ele = Uploader.value;
        ele.submit();
    }
}

/**
 * 文件下载
 * 后台返回的为文件流的处理
 */
const handleDownload = async (file) => {
  try {
    const res = await fileDownload(file.id);
    const fileName = res.headers['content-disposition']
      ? res.headers['content-disposition']
        .split(';')[1]
        .split('=')[1]
        .replace('"', '')
        .replace('"', '')
      : new Date().getTime() + '.' + file.extension;
    const blob = res.data;
    const url = URL.createObjectURL(new Blob([blob]));
    const link = document.createElement('a');
    link.href = url;
    link.setAttribute('download', decodeURIComponent(fileName));
    document.body.appendChild(link);
    link.click();
    document.body.removeChild(link);
  } catch (error) {
    console.error(`下载文件出错:`, error);
  }
};

/**
 * 移除图片回调
 */
const handleRemove = (file) => {
  const index = fileList.value.findIndex(item => item.id === file.id);
  if (index !== -1) fileList.value.splice(index, 1);
  upload(fileList.value);
}

</script>
<style lang="scss" scoped>
.upload_page{
  width: 100%;
  display: flex;
  flex-wrap: wrap;
  .upload_item{
    position: relative;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: space-around;
    width: 148px;
    height: 148px;
    border-radius: 6px;
    margin: 0 10px 10px 0;
    overflow: hidden;
    box-sizing: border-box;
    background-color: #fafafa;
    border: 1px dashed #d9d9d9;
    cursor: pointer;
    
    img{
      width: 120px;
      height: 120px;
    }
    
    .upload_item_name{
      width: 100%;
      text-align: center;
      white-space: nowrap;
      overflow: hidden;
      text-overflow: ellipsis; 
    }
    
    .el-upload-list__item_mask{
      display: flex;
      align-items: center;
      justify-content: space-evenly;
      position: absolute;
      top: -148px;
      left: 0;
      width: 100%;
      height: 120px;
      background-color: rgba(0, 0, 0, 0.5);
      transition: top 0.5s;
    }
    
    &:hover .el-upload-list__item_mask{
      top: 0;
    }
  }
}
</style>

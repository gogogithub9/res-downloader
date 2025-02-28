<script setup lang="ts">
import {ref, onMounted, onUnmounted, watch} from "vue"
import {ipcRenderer} from 'electron'
import {ElMessage, ElLoading, ElTable} from "element-plus"
import localStorageCache from "../common/localStorage"
import {Delete, Promotion} from "@element-plus/icons-vue"

interface resData {
  url: string,
  url_sign: string,
  size: any,
  platform: string,
  type: string,
  type_str: string,
  progress_bar: any,
  save_path: string,
  decode_key: string,
  description: string,
}

const tableData = ref<resData[]>([])


const isInitApp = ref(false)

const multipleTableRef = ref<InstanceType<typeof ElTable>>()
const multipleSelection = ref<resData[]>([])
const loading = ref()
const resType = ref(["all"])
const typeOptions = ref([
  {
    value: "all",
    label: "全部",
  },
  {
    value: "image",
    label: "图片",
  }, {
    value: "audio",
    label: "音频"
  }, {
    value: "video",
    label: "视频"
  }, {
    value: "m3u8",
    label: "m3u8"
  }, {
    value: "live",
    label: "直播流"
  }, {
    value: "xls",
    label: "文档"
  }, {
    value: "doc",
    label: "doc"
  }, {
    value: "pdf",
    label: "pdf"
  }
])

const tableHeight = ref(400)

onMounted(() => {
  let resTypeCache = localStorageCache.get("res-type-arr")
  if (resTypeCache) {
    resType.value = resTypeCache.split(",")
  }

  let tableDataCache = localStorageCache.get("res-table-data")
  if (tableDataCache) {
    tableData.value = tableDataCache
  }

  ipcRenderer.on('on_get_queue', (res, data) => {
    // @ts-ignore
    if (resType.value.includes("all") || resType.value.includes(data.type_str)) {
      tableData.value.push(data)
      localStorageCache.set("res-table-data", tableData.value, -1)
    }
  })

  ipcRenderer.on('on_down_file_schedule', (res: any, data: any) => {
    loading.value && loading.value.setText(`${data.schedule}%`)
  })

  ipcRenderer.invoke('invoke_app_is_init').then((isInit: boolean) => {
    if (!isInit && !isInitApp.value) {
      isInitApp.value = true
      ipcRenderer.invoke('invoke_init_app')
    }
  })

  loading.value = ElLoading.service({
    lock: true,
    text: 'Loading',
    background: 'rgba(0, 0, 0, 0.7)',
  })

  ipcRenderer.invoke('invoke_start_proxy', {upstream_proxy: localStorageCache.get("upstream_proxy")}).then(() => {
    loading.value.close()
  }).catch((err) => {
    ElMessage({
      message: err,
      type: 'warning',
    })
    loading.value.close()
  })
  window.addEventListener("resize", handleResize);
  handleResize()
})

onUnmounted(() => {
  ipcRenderer.removeListener('on_get_queue', (res) => {
    // console.log(res)
  })

  ipcRenderer.removeListener('on_down_file_schedule', (res) => {
    // console.log(res)
  })
})

watch(resType, (res, res1) => {
  localStorageCache.set("res-type-arr", resType.value.join(","), -1)
}, {deep: true})

const handleResize = () => {
  const height = document.documentElement.clientHeight || window.innerHeight;
  tableHeight.value = height - 115
}

const handleSelectionChange = (val: resData[]) => {
  multipleSelection.value = val
}

const handleBatchDown = async () => {
  if (multipleSelection.value.length <= 0) {
    return
  }

  let save_dir = localStorageCache.get("save_dir")

  if (!save_dir) {
    ElMessage({
      message: '请设置保存目录',
      type: 'warning'
    })
    return
  }

  loading.value = ElLoading.service({
    lock: true,
    text: '下载中',
    background: 'rgba(0, 0, 0, 0.7)',
  })
  const quality = localStorageCache.get("quality") ? localStorageCache.get("quality") : -1
  for (const item of multipleSelection.value) {
    let downRes = await ipcRenderer.invoke('invoke_down_file', {
      data: Object.assign({}, item),
      save_path: save_dir,
      quality: quality,
    })

    if (downRes !== false) {
      item.progress_bar = "100%"
      item.save_path = downRes.fullFileName
    }
  }
  loading.value.close()
  multipleTableRef.value!.clearSelection()
}


const handleDown = async (index: number, row: any) => {
  const save_dir = localStorageCache.get("save_dir")
  if (!save_dir) {
    ElMessage({
      message: '请设置保存目录',
      type: 'warning'
    })
    return
  }

  loading.value = ElLoading.service({
    lock: true,
    text: '下载中',
    background: 'rgba(0, 0, 0, 0.7)',
  })

  const quality = localStorageCache.get("quality") ? localStorageCache.get("quality") : -1
  ipcRenderer.invoke('invoke_down_file', {
    data: Object.assign({}, tableData.value[index]),
    save_path: save_dir,
    quality: quality,
  }).then((res) => {
    if (res !== false) {
      tableData.value[index].progress_bar = "100%"
      tableData.value[index].save_path = res.fullFileName
      localStorageCache.set("res-table-data", tableData.value, -1)
    } else {
      ElMessage({
        message: "下载失败",
        type: 'warning',
      })
    }
    loading.value.close()
  }).catch((err) => {
    ElMessage({
      message: "下载失败",
      type: 'warning',
    })
    loading.value.close()
  })
}

const decodeWxFile = (index: number) => {
  loading.value = ElLoading.service({
    lock: true,
    text: "解密中",
    background: 'rgba(0, 0, 0, 0.7)',
  })

  ipcRenderer.invoke('invoke_select_wx_file', {
    index: index,
    data: Object.assign({}, tableData.value[index]),
  }).then((res) => {
    if (res !== false) {
      ElMessage({
        message: "解密成功: " + res.fullFileName,
        type: 'success',
      })
      tableData.value[index].progress_bar = "100%"
      tableData.value[index].save_path = res.fullFileName
      localStorageCache.set("res-table-data", tableData.value, -1)
    } else {
      ElMessage({
        message: "解密失败",
        type: 'warning',
      })
    }
    loading.value.close()
  }).catch((err) => {
    ElMessage({
      message: "解密失败",
      type: 'warning',
    })
    loading.value.close()
  })
}

const handlePreview = (index: number, row: any) => {
  ipcRenderer.invoke('invoke_resources_preview', {url: row.url}).catch(() => {
  })
}

const handleClear = () => {
  tableData.value = []
  localStorageCache.del("res-table-data")
  ipcRenderer.invoke('invoke_file_del', {
    url_sign: "all"
  })
}

const handleCopy = (text: string) => {
  let el = document.createElement('input')
  el.setAttribute('value', text)
  document.body.appendChild(el)
  el.select()
  document.execCommand('copy')
  document.body.removeChild(el)
  ElMessage({
    message: "复制成功",
    type: 'success',
  })
}

const handleDel = (index: number) => {
  let arr = tableData.value
  arr.splice(index, 1);
  tableData.value = arr
  localStorageCache.set("res-table-data", tableData.value, -1)
  ipcRenderer.invoke('invoke_file_del', {
    url_sign: tableData.value[index].url_sign
  })
}

const openFileDir = (index: number) => {
  ipcRenderer.invoke('invoke_open_file_dir', {
    save_path: tableData.value[index].save_path
  })
}

const handleInitApp = () => {
  ipcRenderer.invoke('invoke_app_is_init').then((isInit: boolean) => {
    if (isInit) {
      isInitApp.value = false
    } else {
      ipcRenderer.invoke('invoke_init_app')
    }
  })
}

</script>

<template lang="pug">
el-container.container
  el-header(style="display:flex;align-items: center")
    el-button(type="primary" @click="handleBatchDown") 批量下载
    el-button(v-if="isInitApp" @click="handleInitApp")
      el-icon
        Promotion
      p 安装检测(如果看到此按钮说明软件安装未完成则需要手动点击此按钮)
    el-button(@click="handleClear")
      el-icon
        Delete
      p 清空列表
    el-select(
      v-model="resType"
      multiple
      collapse-tags
      collapse-tags-tooltip
      :max-collapse-tags="3"
      placeholder="资源拦截类型"
      style="width: auto;min-width:130px"
    )
      el-option(v-for="item in typeOptions"
        :key="item.value"
        :label="item.label"
        :value="item.value")
  el-table(ref="multipleTableRef" @selection-change="handleSelectionChange" :data="tableData" :height="tableHeight" max-height="100%" stripe)
    el-table-column(type="selection")
    el-table-column(label="预览" show-overflow-tooltip width="150px")
      template(#default="scope")
        div.show_res
          video.video(v-if="scope.row.type_str === 'video'" :src="scope.row.url" controls preload="none")
          img.img(v-if="scope.row.type_str === 'image'" :src="scope.row.url" crossorigin="anonymous")
          audio.audio(v-if="scope.row.type_str === 'audio'" controls preload="none")
            source(:src="scope.row.url" :type="scope.row.type")
          div(v-if="scope.row.type_str !== 'video' && scope.row.type_str !== 'image' && scope.row.type_str !== 'audio'") {{scope.row.type_str}}类型无法预览
          div {{scope.row.description}}
    el-table-column(prop="type_str" label="类型" show-overflow-tooltip)
    el-table-column(prop="platform" label="主机地址")
    el-table-column(prop="size" label="资源大小")
    el-table-column(prop="save_path" label="保存目录" width="135px" :show-overflow-tooltip="true")
    el-table-column(prop="progress_bar" label="下载进度")
    el-table-column(label="操作" width="135px" )
      template(#default="scope")
        div.actions
          template(v-if="scope.row.type_str !== 'm3u8' && scope.row.type_str !== 'live'" )
            el-button(link type="primary" @click="handleDown(scope.$index, scope.row)") {{scope.row.decode_key || scope.row.decryptor_array ? "解密下载(视频号)" : "下载"}}
            el-button(v-if="scope.row.decode_key || scope.row.decryptor_array" link type="primary" @click="decodeWxFile(scope.$index)") 视频解密(视频号)
            el-button(link type="primary" @click="handlePreview(scope.$index, scope.row)") 窗口预览
          el-button(link type="primary" @click="handleCopy(scope.row.url)") 复制链接
          el-button(link type="primary" @click="handleDel(scope.$index)") 删除
          el-button(v-if="scope.row.save_path" link type="primary" @click="openFileDir(scope.$index)") 打开文件目录
</template>

<style scoped lang="less">
.container {
  padding: 0.5rem;
  .el-table{
    padding-top: 1rem;
  }
  .el-button {
    margin: 0.1rem;
  }

  .el-button p {
    padding-left: 0.2rem;
  }

  .el-row {
    display: flex;
    justify-content: space-between;
  }

  header {
    height: unset !important;
  }

  .el-form-item {
    display: flex;
    justify-content: center;
    align-items: center;

    .select-dir {
      background: #fff5f5;
    }
  }

  .show_res {
    .img {
      max-height: 200px;
    }
    .video {
      width: auto;
      max-height: 200px;
    }
  }

  .actions {
    display: flex;
    flex-direction: column;
    align-items: flex-start;
  }
}
</style>

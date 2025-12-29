<template>
  <div class="title">
    <h1>容器列表</h1>
  </div>
  <el-button type="primary" @click="dialogCreateContainer = true">
    创建容器
  </el-button>
  <div style="text-align:right">
    <el-input v-model="inputSearchBox" style="width: 200px"></el-input>
    <el-button @click="searchContainer">搜索</el-button>
  </div>
<!--  创建容器的对话框-->
  <el-dialog
  v-model="dialogCreateContainer"
  title="创建容器"
  width="500px"
  :before-close="handleClose"
  >
    <el-form :model="form" :rules="rules">
      <el-form-item prop="containerName">
        <span style="color: red">&nbsp;*&nbsp;</span>
        <span>容器名称：</span>
        <el-input v-model="form.containerName"></el-input>
      </el-form-item>
    </el-form>
    <template #footer>
      <div class="dialog-footer">
        <el-button @click="dialogOperate">取消</el-button>
        <el-button type="primary" :disabled="isDisabled" @click="confirmCreateContainer">确认</el-button>
      </div>
    </template>
  </el-dialog>
  <el-table :data="containerData" style="width: 100%">
    <el-table-column fixed align="center" label="容器">
      <template #default="scope">
        <span>{{ scope.row.name }}</span>
      </template>
    </el-table-column>
    <el-table-column prop="bytes" align="center" label="空间占用（字节）"/>
    <el-table-column prop="last_modified" align="center" label="上次修改时间"/>
    <el-table-column prop="count" align="center" label="文件数量"/>
    <el-table-column align="center" fixed="right" label="操作">
      <template #default="scope">
        <el-button type="primary" size="small" @click="openContainer(scope.row)">打开</el-button>
        <el-button type="danger" size="small" @click="deleteContainer(scope.row)">删除</el-button>
      </template>
    </el-table-column>
  </el-table>
  <el-dialog
  v-model="dialogOpenContainer"
  title="文件列表"
  width="80%"
  height="80%"
  :before-close="clearTable"
  >
    <template #header>
      /{{containerName}}
    </template>
    <el-button type="primary" @click="triggerUpload" style="margin-right: 10px">上传文件</el-button>
    <input type="file" ref="fileInput" @change="handleFileUpload" style="display: none"/>
    <el-button type="info" @click="inputDocumentName" style="margin-right: 10px">新建文件夹</el-button>
    <el-input v-show="showDocumentName" v-model="createDocumentName" style="width: 200px; margin-right: 10px"></el-input>
    <el-button v-show="showConfirmCreateDocument" @click="confirmCreateDocument" style="margin-right: 10px">创建</el-button>
    <el-button v-show="showCancelCreateDocument" @click="cancelCreateDocument" style="margin-right: 10px">取消</el-button>
    <div style="text-align:right">
      <el-input v-model="inputSearchBox" style="width: 200px"></el-input>
      <el-button @click="searchBox">搜索</el-button>
    </div>
    <el-table :data="filesData" style="width: 100%; height: 100%">
      <el-table-column label="文件名" align="center">
        <template #default="scope">
          <div>
            {{scope.row.name}}
          </div>
        </template>
      </el-table-column>
      <el-table-column label="文件大小（字节）" align="center" >
        <template #default="scope">
          <div v-if="scope.row.name.slice(-1) !== '/'">
            {{scope.row.bytes}}
          </div>
          <div v-if="scope.row.name.slice(-1) === '/'">
            目录
          </div>
        </template>
      </el-table-column>
      <el-table-column label="上次修改时间" prop="last_modified" align="center" />
      <el-table-column fixed="right" label="操作" align="center">
        <template #default="scope">
          <el-button v-if="scope.row.name.slice(-1) !== '/'" type="primary" size="small" @click="triggerDownload(scope.row)">下载</el-button>
          <el-button v-if="scope.row.name.slice(-1) !== '/'" type="danger" size="small" @click="deleteFile(scope.row)">删除</el-button>
          <el-button v-if="scope.row.name.slice(-1) === '/'" type="danger" @click="deleteDocument(scope.row)">删除文件夹</el-button>
        </template>
      </el-table-column>
    </el-table>
  </el-dialog>
  <el-dialog
      v-model="dialogDeleteContainer"
      title="是否确定删除？"
      width="500px"
  >
    <template #footer>
      <div class="dialog-footer">
        <el-button @click="dialogDeleteContainer = false">取消</el-button>
        <el-button type="primary" @click="confirmDeleteContainer">确定</el-button>
      </div>
    </template>
  </el-dialog>
  <el-dialog
      v-model="dialogDeleteFile"
      title="是否确定删除？"
      width="500px"
  >
    <template #footer>
      <div class="dialog-footer">
        <el-button @click="dialogDeleteFile = false">取消</el-button>
        <el-button type="primary" @click="confirmDeleteFile">确定</el-button>
      </div>
    </template>
  </el-dialog>
  <div class="page">
<!--    <el-pagination layout="prev,pager,next" :total="50"></el-pagination>-->
  </div>
</template>

<script setup lang="ts">
defineOptions({
  name: "Containers",
})
import {onMounted, ref} from "vue";
import {del, get, put} from "@/api/request"
import {ElMessage, ElMessageBox} from "element-plus";
// 用于存放容器表格数据
const containerData = ref();
// 用于获取容器表格数据
const fetchContainerData = async() => {
  containerData.value = await get('/')
}
// 用于搜索的文本
const inputSearchBox = ref()
// 触发容器搜索
const searchContainer = async () => {
  containerData.value = await get('?prefix=' + inputSearchBox.value)
  inputSearchBox.value = ''
}
// 触发文件搜索
const searchBox = async () => {
  filesData.value = await get(containerName.value + '?prefix=' + inputSearchBox.value)
  inputSearchBox.value = ''
}
// 存放容器名称
const containerName = ref()
// 用于存放容器内的数据
const filesData = ref();
// 用于获取容器内的文件
const fetchFilesData = async () => {
  filesData.value = await get(containerName.value);
}
// 存放待上传的文件
const fileInput:any = ref(null)
// 触发文件选择
const triggerUpload = () => {
  fileInput.value.click();
}
// 用于存放文件名称
const filesName = ref()
// 控制是否禁用按钮
const isDisabled = ref(true);
// 验证Input的有效性
const rules = ref({
  containerName: [
    {
      validator: (rule: any, value: any, callback: any) => {
        const regex = /[!*‘;:@&=+$,/?#\[\]<>'"%{}|\\^`~\s]/;
        if (regex.test(value)) {
          isDisabled.value = true;
          callback(new Error('非法字符'));
        }
        else if (form.value.containerName === '') {
          isDisabled.value = true;
          callback();
        }
        else {
          isDisabled.value = false;
          callback();
        }
      },
      trigger: ["blur", "change"]
    }
  ]
})
// 控制是否弹出创建容器的对话框
const dialogCreateContainer = ref(false)
// 清除Input内容并关闭对话框
const dialogOperate = () => {
  dialogCreateContainer.value = false
  form.value.containerName = ''
}
// 点击叉号时弹出是否确定关闭对话框的选项
const handleClose = (done: () => void) => {
  ElMessageBox.confirm(
      '是否取消？',
      {
        confirmButtonText: "是",
        cancelButtonText: "否"
      }
  )
  .then(() => {
    form.value.containerName = ''
    done()
  })
  .catch(() => {
  })
}
// 点击“创建”按钮时弹出是否创建容器的选项
const confirmCreateContainer = () => {
  ElMessageBox.confirm(
      '是否创建？',
      {
        confirmButtonText: "创建",
        cancelButtonText: "取消"
      }
  )
  .then(async () => {
    await put(form.value.containerName)
    await fetchContainerData()
    form.value.containerName = ''
    dialogCreateContainer.value = false
    ElMessage({
      type: 'success',
      message: "创建成功"
    })
  })
  .catch(() => {
  })
}
// 触发新建文件夹输入框和按钮的显示
const inputDocumentName = () => {
  showDocumentName.value = true
  showConfirmCreateDocument.value = true
  showCancelCreateDocument.value = true
}
// 显示文件夹名称的输入框
const showDocumentName = ref(false)
// 显示文件夹输入框的确定按钮
const showConfirmCreateDocument = ref(false)
// 触发新建文件夹的方法
const confirmCreateDocument = () => {
  postDocumentName()
  createDocumentName.value = ''
  showDocumentName.value = false
  showConfirmCreateDocument.value = false
  showCancelCreateDocument.value = false
}
// 显示文件夹输入框的取消按钮
const showCancelCreateDocument = ref(false)
// 关闭输入框和按钮
const cancelCreateDocument = () => {
  createDocumentName.value = ''
  showDocumentName.value = false
  showConfirmCreateDocument.value = false
  showCancelCreateDocument.value = false
}
// 存放用户输入的文件夹名称
const createDocumentName = ref('')
// 创建文件夹的方法
const postDocumentName = async () => {
  await put(containerName.value + '/' + createDocumentName.value + '/')
  await fetchFilesData()
}
// 删除文件夹的方法
const deleteDocument = async (row: any) => {
  await del(containerName.value + '/' + row.name)
  await fetchFilesData()
}
// 用于存放Input数据
const form = ref({containerName: ""})
// 打开容器显示文件的弹窗控制
const dialogOpenContainer = ref(false)
// 获取容器名称用于打开容器
const openContainer = (row: any) => {
  containerName.value = row.name
  fetchFilesData()
  dialogOpenContainer.value = true
}
// 关闭文件列表时清除filesData
const clearTable = () => {
  dialogOpenContainer.value = false
  filesData.value = []
}
// 获取容器名称用于删除容器
const deleteContainer = (row: any) => {
  containerName.value = row.name
  dialogDeleteContainer.value = true
}
// 控制弹出是否删除容器的对话框
const dialogDeleteContainer = ref(false)
// 删除容器的方法
const confirmDeleteContainer = async () => {
  await del(containerName.value)
  await fetchContainerData()
  dialogDeleteContainer.value = false
  // console.log(containerName.value)
}
// 获取文件名称用于删除文件
const deleteFile = (row: any) => {
  filesName.value = row.name
  dialogDeleteFile.value = true
}
// 控制弹出是否删除文件的对话框
const dialogDeleteFile = ref(false)
// 删除容器文件的方法
const confirmDeleteFile = async () => {
  await del(containerName.value + '/' + filesName.value)
  await fetchFilesData()
  await fetchContainerData()
  dialogDeleteFile.value = false
}
// 上传文件的方法
const handleFileUpload = async (event: any) => {
  const file = event.target.files[0]
  const fileName = file.name
  const data = new Blob([file], { type: file.type });
  const config = {headers: {'Content-Type': file.type}};
  // console.log(file)
  // console.log(fileName)
  // console.log(containerName.value + '/' + fileName)
  await put(containerName.value + '/' + fileName, data, config)
  await fetchFilesData()
  await fetchContainerData()
}
// 触发下载文件的方法
const triggerDownload = async (row: any) => {
  filesName.value = row.name
  await downloadFile()
}
// 下载文件的方法
const downloadFile = async () => {
  try {
    const url = containerName.value + '/' + filesName.value
    const blob:any = await get(url, {responseType: 'blob'});
    const link:any = document.createElement("a");
    link.href = window.URL.createObjectURL(blob);
    link.download = url.split("/").at(-1)
    link.click();
    window.URL.revokeObjectURL(link.href);
    // console.log(url)
  } catch (error) {
    console.error('download failed',error)
  }
}
// 当页面加载或刷新时自动获取表格数据
onMounted(() => {
  fetchContainerData()
})
</script>

<style scoped>
.title {
  font-size: 20px;
  margin-bottom: 1%;
}
.page {
  display: flex;
  justify-content: center;
}
</style>
<script setup lang="ts">
import { computed, watch, reactive } from "vue";

enum Status {
  wait,
  pause,
  uploading,
}

const SIZE = 10 * 1024 * 1024;

const state = reactive<{
  container: { file: any; hash: string; worker: null | any };
  data: Array<{
    chunk: any;
    hash: string;
    index: number;
    percentage: number;
    size: number;
  }>;
  requestList: Array<any>;
  hashPercentage: number;
  status: Status;
  fakeUploadPercentage: number;
}>({
  container: { file: null, hash: "", worker: null },
  hashPercentage: 0,
  requestList: [],
  status: Status.wait,
  fakeUploadPercentage: 0,
  data: [],
});

const uploadDisabled = computed<boolean>(() => {
  return (
    !state.container.file ||
    state.status === Status.pause ||
    state.status === Status.uploading
  );
});

const uploadPercentage = computed<number>(() => {
  if (!state.container.file || !state.data.length) return 0;
  const loaded = state.data
    .map((item) => item.size * item.percentage)
    .reduce((acc, cur) => acc + cur, 0);
  return parseInt((loaded / state.container.file.size).toFixed(2), 10);
});

watch(uploadPercentage, (value) => {
  if (value > state.fakeUploadPercentage) {
    state.fakeUploadPercentage = value;
  }
});

// utils
function request({
  url,
  method = "post",
  data,
  headers = {},
  onProgress = (e: any) => {},
  requestList,
}: any) {
  return new Promise((resolve) => {
    const xhr = new XMLHttpRequest();
    xhr.upload.onprogress = onProgress;
    xhr.open(method, url);
    Object.keys(headers).forEach((key) => {
      xhr.setRequestHeader(key, headers[key]);
    });
    xhr.send(data);
    xhr.addEventListener("load", (e) => {
      if (requestList) {
        const xhrIndex = requestList.findIndex((item: any) => item === xhr);
        requestList.splice(xhrIndex, 1);
      }
      resolve({ data: e.target.response });
    });
    requestList?.push(xhr);
  });
}

function transformBytes(val: number) {
  return Number((val / 1024).toFixed(0));
}

// 重置数据
function resetData() {
  state.requestList.forEach((xhr) => xhr?.absort());
  state.requestList = [];
  if (state.container.worker) {
    state.container.worker.onmessage = null;
  }
}

// 生成文件hash
function calculateHash(fileChunkList) {
  return new Promise((resolve) => {
    state.container.worker = new Worker("/hash.js");
    state.container.worker.postMessage({ fileChunkList });
    state.container.worker.onmessage = (e) => {
      const { percentage, hash } = e.data;
      state.hashPercentage = percentage;
      if (hash) {
        resolve(hash);
      }
    };
  });
}

// 生成文件切片
function createFileChunk(file: any, size = SIZE) {
  const fileChunkList = [];
  let cur = 0;
  while (cur < file.size) {
    fileChunkList.push({ file: file.slice(cur, cur + size) });
    cur += size;
  }
  return fileChunkList;
}

function createProgressHandler(item: any) {
  return (e: any) => {
    item.percentage = parseInt(String((e.loaded / e.total) * 100));
  };
}

async function uploadChunks() {
  const requestList = state.data
    .map(({ chunk, hash, index }) => {
      const formData = new FormData();
      formData.append("chunk", chunk);
      formData.append("hash", hash);
      formData.append("fileHash", state.container.hash);
      formData.append("filename", state.container.file!.name);
      return { formData, index };
    })
    .map(({ formData, index }) => {
      // send a request
      return request({
        url: "http://localhost:3000",
        data: formData,
        onProgress: createProgressHandler(state.data[index]),
        requestList: state.requestList,
      });
    });
  // upload filelist

  await Promise.all(requestList);
  // merge request
  if (requestList.length === state.data.length) {
    await mergeRequest();
  }
}

async function mergeRequest() {
  await request({
    url: "http://localhost:3000/merge",
    headers: {
      "content-type": "application/json",
    },
    data: JSON.stringify({
      filename: state.container.file.name,
      size: SIZE,
      fileHash: state.container.hash,
    }),
  });
  window.alert("upload success, check /target directory");
  state.status = Status.wait;
}

// event handler
async function handleDelete() {
  try {
    const { data } = await request({ url: "http://localhost:3000/delete" });
    if (JSON.parse(data).code === 0) {
      window.alert("delete success");
    }
  } catch (error) {
    console.log(error);
  }
}

function handlePause() {
  state.status = Status.pause;
  resetData();
}

function handleFileChange(e: any) {
  const [file] = e.target.files;
  if (!file) return;
  resetData();
  state.container.file = file;
}

async function handleUpload() {
  if (!state.container.file) return;
  state.status = Status.uploading;
  const fileChunkList = createFileChunk(state.container.file);
  state.container.hash = (await calculateHash(fileChunkList)) as string;

  state.data = fileChunkList.map(({ file }, index) => ({
    chunk: file,
    fileHash: state.container.hash,
    index,
    hash: state.container.file.name + "-" + index,
    percentage: 0,
    size: file.size,
  }));
  await uploadChunks();
}
</script>

<template>
  <div class="container">
    <div>
      <input
        type="file"
        :disabled="state.status !== Status.wait"
        @change="handleFileChange"
      />
      <button @click="handleUpload" :disabled="uploadDisabled">upload</button>
    </div>
    <div class="upload-container">
      <div>
        <div>total percentage: {{ state.fakeUploadPercentage }}%</div>
      </div>
      <div>
        <div v-for="(item, index) in state.data" key="index">
          {{ item.hash }} - {{ transformBytes(item.size) }}kb -
          {{ item.percentage }}
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.container {
  display: flex;
  gap: 2rem;
  flex-direction: column;
  justify-content: center;
  align-items: center;
}
</style>

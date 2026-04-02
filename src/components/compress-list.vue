<template>
  <div class="compress-list">
    <div class="compress-list__list">
      <compress-item v-for="(it, idx) in modelValue.images" :data="it" :key="idx" @cancel="handleCancel(idx)" @refresh="handleRefresh(idx)" />
    </div>
    <div class="compress-list__total">
      <p class="compress-list__total__progress">
        <span>完成 {{ (total.progress * 100).toFixed(2) }}%</span>
        <span>成功 {{ total.success }}</span>
        <span>失败 {{ total.error }}</span>
      </p>
      <p>
        <span>共减少</span>
        <span class="blue">{{ bytes(total.reduce) }}</span>
        <span class="green">{{ (total.ratio * 100).toFixed(2) }}%</span>
      </p>
      <span v-if="canRetry && retryCountdown > 0" style="padding: 0 8px; color: var(--el-text-color-secondary)">
        {{ retryCountdown }}s 后自动重试
      </span>
      <el-button type="primary" @click="handleRetryErrors" :disabled="!canRetry">重试失败</el-button>
      <el-button type="primary" @click="handleReplaceAll" :disabled="!total.complate">覆盖原图</el-button>
      <el-button type="primary" @click="handleCopyAll" :disabled="!total.complate">复制所有</el-button>
    </div>
  </div>
</template>

<style lang="scss" scoped>
.compress-list {
  flex: 1;
  display: flex;
  flex-direction: column;
  overflow: hidden;

  &__list {
    flex: 1;
    overflow-y: auto;
    padding-bottom: 10px;

    &::-webkit-scrollbar {
      display: none;
    }
  }

  &__total {
    position: relative;
    height: 50px;
    display: flex;
    align-items: center;
    padding: 0 10px;
    color: var(--el-text-color-primary);

    &:before {
      content: '';
      position: absolute;
      left: 0;
      top: 0;
      width: v-bind(progress);
      height: 5px;
      border-radius: 2px;
      background: #90caf9;
      transition: width 0.5s;
    }

    &__progress {
      flex: 1;
    }

    p {
      span {
        padding: 0 8px;

        &.blue {
          color: var(--el-color-primary);
        }

        &.green {
          color: var(--el-color-success);
        }
      }
    }
  }
}
</style>

<script lang="ts" setup>
import axios, { AxiosError, CancelTokenSource } from 'axios';
import { computed, nextTick, onBeforeUnmount, onMounted, PropType, ref, watch } from 'vue';
import { Action, ElButton, ElMessage, ElMessageBox } from 'element-plus';

import compressItem from './compress-item.vue';
import { bytes, random } from '../utils';

const props = defineProps({
  modelValue: { type: Object as PropType<TinypngConfig.List>, required: true }
});
const emit = defineEmits(['update:modelValue']);

const cancelTokens: CancelTokenSource[] = [];

const total = computed(() => {
  const progress = props.modelValue.images.reduce((a, b) => a + (b.compress?.progress ?? 0), 0);
  const totalSize = props.modelValue.images.reduce((a, b) => a + b.size, 0);
  const totalSurplus = props.modelValue.images.reduce((a, b) => a + (b.compress?.size ?? b.size), 0);
  const reduce = totalSize - totalSurplus;
  const success = props.modelValue.images.filter(it => it.compress?.progress === 1).length;
  const error = props.modelValue.images.filter(it => it.compress?.error).length;
  const canceled = props.modelValue.images.filter(it => it.compress?.canceled).length;
  const totalCount = props.modelValue.images.length;
  return {
    progress: progress && progress / totalCount,
    success,
    error,
    reduce,
    ratio: reduce && reduce / totalSize,
    complate: success + error + canceled === totalCount && success > 0
  };
});

const progress = computed(() => total.value.progress * 100 + '%');

const hasCompressing = computed(() =>
  props.modelValue.images.some(it => it.compress?.progress > 0 && it.compress?.progress < 1 && !it.compress?.error && !it.compress?.canceled)
);
const hasError = computed(() => props.modelValue.images.some(it => Boolean(it.compress?.error)));
const canRetry = computed(() => !hasCompressing.value && hasError.value);

const isStopping = ref(false);


const isRetrying = ref(false);

const retryCountdown = ref(0);
let retryTimer: number | null = null;

function stopRetryCountdown() {
  if (retryTimer !== null) {
    window.clearInterval(retryTimer);
    retryTimer = null;
  }
  retryCountdown.value = 0;
}

function startRetryCountdown(seconds = 60) {
  stopRetryCountdown();
  retryCountdown.value = seconds;
  retryTimer = window.setInterval(async () => {
    if (!canRetry.value || isRetrying.value) {
      stopRetryCountdown();
      return;
    }

    retryCountdown.value -= 1;
    if (retryCountdown.value <= 0) {
      stopRetryCountdown();
      if (canRetry.value && !isRetrying.value) {
        await handleRetryErrors();
      }
    }
  }, 1000);
}

async function updateItem(idx: number, value: DeepPartial<TinypngConfig.List.Image>) {
  const { images } = props.modelValue;
  emit('update:modelValue', {
    ...props.modelValue,
    images: [
      ...images.slice(0, idx),
      { ...images[idx], ...value, compress: { ...images[idx].compress, ...value.compress } },
      ...images.slice(idx + 1)
    ]
  } as TinypngConfig.List);
  await nextTick();
}

function isNotStarted(it: TinypngConfig.List.Image) {
  return !it.compress?.error && !it.compress?.canceled && (it.compress?.progress ?? 0) === 0;
}

async function stopAllNotStarted(reason: string) {
  if (isStopping.value) return;
  isStopping.value = true;

  for (let i = 0; i < props.modelValue.images.length; i++) {
    const it = props.modelValue.images[i];
    if (isNotStarted(it)) {
      await updateItem(i, { compress: { error: true, msg: reason } });
    }
  }
}

async function cacheError(idx: number, error: AxiosError<TinypngApi.Error>) {
  console.error('error: ', error);
  if (error.message !== 'canceled') {
    await updateItem(idx, { compress: { error: true, msg: error.response?.data?.message ?? error.message } });
  }
  return { data: null };
}

async function compressOne(idx: number): Promise<boolean> {
  try {
    if (!window.preload) return true;
    const it = props.modelValue.images[idx];
    if (it.compress.canceled) return true;
    if (isStopping.value || it.compress.error) return true;
    if ((it.compress.progress ?? 0) === 0) {
      await updateItem(idx, { compress: { progress: 0.001 } });
    }
    if (!(it.compress.progress >= 0.67 && it.compress.downloadUrl)) {
      await updateItem(idx, { compress: { progress: Math.max(it.compress.progress ?? 0, 0.001) } });
      const buf = await window.preload.readFile(it.path);
      const fakeIp = Array.from({ length: 4 }, () => random(0, 255)).join('.');
      cancelTokens[idx] = axios.CancelToken.source();
      const { data } = await axios
        .post<TinypngApi.Upload.Response>('/backend/opt/shrink', buf, {
          baseURL: random(['https://tinypng.com', 'https://tinyjpg.com', 'https://tinify.cn']),
          headers: { 'content-type': 'image/png', 'X-Forwarded-For': fakeIp },
          cancelToken: cancelTokens[idx].token,
          onUploadProgress: e => updateItem(idx, { compress: { progress: (e.total ? e.loaded / e.total : 0) * 0.33 } }),
          onDownloadProgress: e => updateItem(idx, { compress: { progress: (e.total ? e.loaded / e.total : 0) * 0.33 + 0.34 } })
        })
        .catch(cacheError.bind(void 0, idx));
      if (!data) return !props.modelValue.images[idx].compress.error;
      await updateItem(idx, { compress: { size: data.output.size, downloadUrl: data.output.url, progress: 0.67 } });
    }
    await updateItem(idx, { compress: { progress: 0.67 } });
    cancelTokens[idx] = axios.CancelToken.source();
    const { data: compressed } = await axios
      .get<ArrayBuffer>(props.modelValue.images[idx].compress.downloadUrl!, {
        responseType: 'arraybuffer',
        cancelToken: cancelTokens[idx].token,
        onDownloadProgress: e => updateItem(idx, { compress: { progress: (e.total ? e.loaded / e.total : 0) * 0.32 + 0.67 } })
      })
      .catch(cacheError.bind(void 0, idx));
    if (!compressed) return !props.modelValue.images[idx].compress.error;
    await window.preload.writeFile(it.compress.path, compressed);
    await updateItem(idx, { compress: { progress: 1 } });
    return true;
  } catch (e) {
    console.error('e: ', e);
    await updateItem(idx, { compress: { error: true, msg: String(e) } });
    return false;
  }
}

async function handleCancel(idx: number) {
  cancelTokens[idx]?.cancel();
  await updateItem(idx, { compress: { canceled: true } });
}

async function handleRefresh(idx: number) {
  await updateItem(idx, { compress: { canceled: false, error: false, msg: '' } });
  isStopping.value = false;
  await compressOne(idx);
}

async function handleCopyAll() {
  const files = await window.preload!.readDir(props.modelValue.tempdir);
  utools.copyFile(files) ? ElMessage.success('复制成功！') : ElMessage.error('复制失败！');
}

async function handleReplaceAll() {
  const result: Action = await ElMessageBox.confirm('您确定要覆盖此文件吗？此操作不可还原！', {
    confirmButtonText: '确认',
    cancelButtonText: '取消'
  }).catch(t => t);
  if (result !== 'confirm') return;
  await window
    .preload!.replaceFiles(props.modelValue.images.filter(it => it.compress.progress === 1).map(it => [it.compress.path, it.path]))
    .then(() => ElMessage.success('覆盖成功！'))
    .catch(error => ElMessage.success(error));
}

onMounted(async () => {
  if (!window.preload) return;
  async function runCompression(indices?: number[]) {
    const max = 3;
    const list = indices ? [...indices] : props.modelValue.images.map((_, i) => i);
    let cursor = 0;

    async function worker() {
      while (cursor < list.length && !isStopping.value) {
        const idx = list[cursor++];
        const it = props.modelValue.images[idx];
        if (!it) continue;
        if (it.compress?.progress === 1 || it.compress?.canceled || it.compress?.error) continue;

        const ok = await compressOne(idx);
        const latest = props.modelValue.images[idx];
        if (!ok && latest?.compress?.error) {
          await stopAllNotStarted('已中止：压缩过程中存在失败任务');
          break;
        }
      }
    }

    await Promise.all(Array.from({ length: max }, () => worker()));
  }

  await runCompression();
});

async function handleRetryErrors() {
  if (!canRetry.value) return;
  stopRetryCountdown();
  if (isRetrying.value) return;
  isRetrying.value = true;
  isStopping.value = false;

  try {
    const errorIdxs: number[] = [];
    for (let i = 0; i < props.modelValue.images.length; i++) {
      const it = props.modelValue.images[i];
      if (it.compress?.error) errorIdxs.push(i);
    }
    if (!errorIdxs.length) return;

    for (const idx of errorIdxs) {
      await updateItem(idx, {
        compress: {
          canceled: false,
          error: false,
          msg: '',
          progress: 0,
          downloadUrl: '',
          size: void 0
        }
      });
    }

    // 只重试失败项
    const max = 3;
    let cursor = 0;
    async function worker() {
      while (cursor < errorIdxs.length && !isStopping.value) {
        const idx = errorIdxs[cursor++];
        const ok = await compressOne(idx);
        const latest = props.modelValue.images[idx];
        if (!ok && latest?.compress?.error) {
          await stopAllNotStarted('已中止：压缩过程中存在失败任务');
          break;
        }
      }
    }
    await Promise.all(Array.from({ length: max }, () => worker()));
  } finally {
    isRetrying.value = false;
  }
}

watch(
  [canRetry, () => total.value.error],
  ([nextCanRetry], [prevCanRetry]) => {
    if (nextCanRetry) {
      if (!prevCanRetry || retryCountdown.value === 0) startRetryCountdown(60);
      else startRetryCountdown(60);
    } else {
      stopRetryCountdown();
    }
  },
  { immediate: true }
);

onBeforeUnmount(() => {
  stopRetryCountdown();
});
</script>

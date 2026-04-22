<script setup lang="tsx">
import { ref } from "vue";
import ReCropper from "@/components/ReCropper";
import { formatBytes } from "@pureadmin/utils";

defineOptions({
  name: "ReCropperPreview"
});

defineProps({
  imgSrc: String
});

const emit = defineEmits(["cropper"]);

const infos = ref();
const popoverRef = ref();
const showPopover = ref(false);
const cropError = ref(false);
const cropperImg = ref<string>("");

function onCropper({ base64, blob, info }) {
  cropError.value = false;
  infos.value = info;
  cropperImg.value = base64;
  emit("cropper", { base64, blob, info });
}

function onCropperError() {
  // Keep dialog usable when old remote cover cannot be cropped due CORS.
  showPopover.value = true;
  cropError.value = true;
}

function hidePopover() {
  popoverRef.value?.hide?.();
}

defineExpose({ hidePopover });
</script>

<template>
  <div v-loading="!showPopover" element-loading-background="transparent">
    <el-popover
      ref="popoverRef"
      :visible="showPopover"
      placement="right"
      width="18vw"
    >
      <template #reference>
        <div class="w-[18vw]">
          <ReCropper
            :src="imgSrc"
            circled
            @cropper="onCropper"
            @error="onCropperError"
            @readied="showPopover = true"
          />
          <p v-show="showPopover" class="mt-1 text-center">
            温馨提示：右键上方裁剪区可打开功能菜单
          </p>
          <p v-if="cropError" class="mt-1 text-center text-[var(--el-color-warning)]">
            当前封面无法直接裁剪，请在右键菜单中先上传新图片。
          </p>
        </div>
      </template>
      <div class="flex flex-wrap justify-center items-center text-center">
        <el-image
          v-if="cropperImg"
          :src="cropperImg"
          :preview-src-list="Array.of(cropperImg)"
          fit="cover"
        />
        <div v-if="infos" class="mt-1">
          <p>
            图像大小：{{ parseInt(infos.width) }} ×
            {{ parseInt(infos.height) }}像素
          </p>
          <p>
            文件大小：{{ formatBytes(infos.size) }}（{{ infos.size }} 字节）
          </p>
        </div>
      </div>
    </el-popover>
  </div>
</template>

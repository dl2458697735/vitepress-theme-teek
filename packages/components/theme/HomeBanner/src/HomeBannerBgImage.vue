<script setup lang="ts" name="HomeBannerBgImage">
import type { Banner } from "@teek/config";
import { withBase } from "vitepress";
import { computed, onMounted, onUnmounted, shallowRef } from "vue";
import { useNamespace, useLocale, useSwitchData } from "@teek/composables";
import { isString } from "@teek/helper";
import { useTeekConfig } from "@teek/components/theme/ConfigProvider";

defineOptions({ name: "HomeBannerBgImage" });

const ns = useNamespace("banner-bg-image");
const { t } = useLocale();
const { getTeekConfigRef } = useTeekConfig();

// Banner 配置项
const bannerConfig = getTeekConfigRef<Required<Banner>>("banner", {
  bgStyle: undefined,
  imgSrc: undefined,
  imgInterval: 15000,
  imgShuffle: false,
  mask: true,
  maskBg: "rgba(0, 0, 0, 0.4)",
});

// 局部图片背景风格
const isPartImgBgStyle = computed(() => bannerConfig.value.bgStyle === "partImg");
// 全屏图片背景风格
const isFullImgBgStyle = computed(() => bannerConfig.value.bgStyle === "fullImg");

const dataArray = computed(() => [bannerConfig.value.imgSrc || []].flat().map(item => item && withBase(item)));

// 缓存下一张图片的 Blob 数据和 URL 对象
const nextImageCache = shallowRef<{ blob: Blob | null; objectUrl: string | null }>({
  blob: null,
  objectUrl: null,
});

// banner 背景图片定时轮播
const {
  data: imageSrc,
  start,
  index,
} = useSwitchData(dataArray, {
  timeout: bannerConfig.value.imgInterval,
  shuffle: bannerConfig.value.imgShuffle,
  onAfterUpdate: async () => {
    console.log("onAfterUpdate", index.value);
    const nextIndex = (index.value + 1) % dataArray.value.length;
    let nextImgApiUrl = dataArray.value[nextIndex];

    if (nextImgApiUrl) {
      // 清除上一次的缓存（避免内存泄漏）
      if (nextImageCache.value.objectUrl) {
        URL.revokeObjectURL(nextImageCache.value.objectUrl);
      }

      try {
        // ✅ 核心：用 Image 对象发起请求（自动标记为 Image 类型，支持预览）
        const img = new Image();
        // 允许跨域（若图片有 CORS 限制，需服务器配合）
        img.crossOrigin = "anonymous";
        // 图片加载成功后的处理
        img.onload = async () => {
          console.log("图片加载成功（已被识别为 Image 类型）");

          // ✅ 将 Image 转为 Blob 缓存（保留你的核心需求）
          const canvas = document.createElement("canvas");
          const ctx = canvas.getContext("2d");
          if (!ctx) throw new Error("Canvas 初始化失败");

          // 让 canvas 尺寸与图片一致
          canvas.width = img.width;
          canvas.height = img.height;
          // 绘制图片到 canvas
          ctx.drawImage(img, 0, 0);

          // 将 canvas 转为 Blob
          canvas.toBlob(correctedBlob => {
            if (!correctedBlob) throw new Error("Canvas 转 Blob 失败");

            // 生成 Blob 本地 URL 并缓存
            const objectUrl = URL.createObjectURL(correctedBlob);
            nextImageCache.value = {
              blob: correctedBlob,
              objectUrl: objectUrl,
            };
            console.log("Blob 缓存成功，类型：", correctedBlob.type);
          });
        };

        // 图片加载失败的处理
        img.onerror = err => {
          throw new Error(`图片加载失败: ${err}`);
        };

        // 如果是网络图片，则添加时间戳
        if (nextImgApiUrl.startsWith("http")) {
          nextImgApiUrl = nextImgApiUrl + "?t=" + Date.now();
        }
        // ✅ 发起图片请求（浏览器会自动标记为 Image 类型，支持预览）
        img.src = nextImgApiUrl;
      } catch (error) {
        console.warn("预加载图片失败:", error);
      }
    }
  },
});

onMounted(() => {
  start();
});

// 组件卸载时释放 Blob URL，避免内存泄漏
onUnmounted(() => {
  if (nextImageCache.value.objectUrl) {
    URL.revokeObjectURL(nextImageCache.value.objectUrl);
  }
});

const getStyle = () => {
  const { imgSrc, maskBg, imgInterval } = bannerConfig.value;
  const imgBgVar = ns.cssVarName("banner-img-bg");
  const maskBgColorVar = ns.cssVarName("banner-mask-bg-color");
  const imgSwitchIntervalVar = ns.cssVarName("banner-img-switch-interval-s");

  // 如果没有传入图片，则加载默认图片
  if (!imgSrc?.length) return { [imgBgVar]: ns.cssVar("bg-img-default") };
  // 关键：显示时直接使用预加载缓存的 Blob URL，该 URL 指向本地缓存的二进制数据，与预加载的是同一份
  const currentImgUrl = nextImageCache.value.objectUrl || imageSrc.value;

  return {
    [imgBgVar]: `url(${currentImgUrl}) center center / cover no-repeat`,
    [maskBgColorVar]: isString(maskBg) ? maskBg : `rgba(0, 0, 0, ${maskBg})`,
    [imgSwitchIntervalVar]: imgInterval / 1000 + "s",
  };
};
</script>

<template>
  <div
    :class="[ns.b(), { part: isPartImgBgStyle, full: isFullImgBgStyle }]"
    :style="getStyle()"
    :aria-label="t('tk.homeBanner.bgImgLabel')"
  >
    <div v-if="bannerConfig.mask && bannerConfig.imgSrc" class="mask" :aria-label="t('tk.homeBanner.maskLabel')" />
    <slot v-if="isPartImgBgStyle" />
  </div>
  <slot v-if="isFullImgBgStyle" />
</template>

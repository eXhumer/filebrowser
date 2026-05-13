<template>
  <div class="video-wrapper">
    <component :is="'video-player'" :key="isMinimal ? 'minimal' : 'default'">
      <component
        :is="isMinimal ? 'video-minimal-skin' : 'video-skin'"
        class="video-fill"
        ref="skinEl"
      >
        <video preload="auto">
          <source :src="source" :type="sourceType || undefined" />
          <track
            kind="subtitles"
            v-for="(sub, index) in subtitles"
            :key="index"
            :src="sub"
            :label="subLabel(sub)"
            :default="index === 0"
          />
          <p class="vjs-no-js">
            Sorry, your browser doesn't support embedded videos, but don't
            worry, you can <a :href="source">download it</a> and watch it with
            your favorite video player!
          </p>
        </video>
      </component>
    </component>
  </div>
</template>

<script setup lang="ts">
import { computed, nextTick, onMounted, ref, watch } from "vue";
import "@videojs/html/video/skin";
import "@videojs/html/video/skin.css";
import "@videojs/html/video/minimal-skin";
import "@videojs/html/video/minimal-skin.css";

const props = withDefaults(
  defineProps<{
    source: string;
    subtitles?: string[];
    isMinimal?: boolean;
  }>(),
  {
    isMinimal: false,
  }
);

const skinEl = ref<Element | null>(null);

const gradientSheet = new CSSStyleSheet();
gradientSheet.replaceSync(
  ".media-overlay { background-image: none !important; } :host { --media-border-radius: 0 !important; }"
);

function suppressGradient(el: Element | null) {
  const root = (el as any)?.shadowRoot as ShadowRoot | null;
  if (!root) return;
  if (!root.adoptedStyleSheets.includes(gradientSheet)) {
    root.adoptedStyleSheets = [...root.adoptedStyleSheets, gradientSheet];
  }
}

const savedTime = ref(0);
const savedPaused = ref(true);

onMounted(() => suppressGradient(skinEl.value));
watch(
  () => props.isMinimal,
  async () => {
    const videoEl = skinEl.value?.querySelector(
      "video"
    ) as HTMLVideoElement | null;
    savedTime.value = videoEl?.currentTime ?? 0;
    savedPaused.value = videoEl?.paused ?? true;
    videoEl?.pause();

    await nextTick();

    suppressGradient(skinEl.value);

    const newVideoEl = skinEl.value?.querySelector(
      "video"
    ) as HTMLVideoElement | null;
    if (!newVideoEl) return;

    const restore = () => {
      newVideoEl.currentTime = savedTime.value;
      if (!savedPaused.value) newVideoEl.play().catch(() => {});
    };

    if (newVideoEl.readyState >= 1) {
      restore();
    } else {
      newVideoEl.addEventListener("loadedmetadata", restore, { once: true });
    }
  }
);

const sourceType = computed(() => getSourceType(props.source));

const getSourceType = (source: string): string => {
  const fileExtension = source ? source.split("?")[0].split(".").pop() : "";
  if (fileExtension?.toLowerCase() === "mkv") {
    return "video/mp4";
  }
  return "";
};

const subLabel = (subUrl: string) => {
  let url: URL;
  try {
    url = new URL(subUrl);
  } catch {
    url = new URL(subUrl, window.location.origin);
  }

  return decodeURIComponent(
    url.pathname
      .split("/")
      .pop()!
      .replace(/\.[^/.]+$/, "")
  );
};
</script>

<style scoped>
.video-wrapper {
  display: block;
  width: 100%;
  height: 100%;
}

.video-fill {
  display: block;
  width: 100%;
  height: 100%;
}
</style>

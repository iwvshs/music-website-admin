<script setup lang="ts">
import { getPlaylistSongIds, getSongList } from "@/api/system";
import { message } from "@/utils/message";
import { computed, onMounted, ref } from "vue";

/**
 * Author: Tanh
 * Date: 2026-03-22
 * Description: Playlist song binding dialog component
 */

interface SongOption {
  key: number;
  label: string;
  artistName?: string;
  album?: string;
}

const props = withDefaults(
  defineProps<{
    playlistId: number;
    playlistTitle?: string;
  }>(),
  {
    playlistTitle: ""
  }
);

const loading = ref(false);
const songOptions = ref<Array<SongOption>>([]);
const selectedSongIds = ref<Array<number>>([]);

const selectedCount = computed(() => selectedSongIds.value.length);

const filterMethod = (query: string, item: SongOption) => {
  if (!query) return true;
  const keyword = query.toLowerCase();
  return (
    item.label.toLowerCase().includes(keyword) ||
    (item.artistName || "").toLowerCase().includes(keyword) ||
    (item.album || "").toLowerCase().includes(keyword)
  );
};

async function fetchAllSongs() {
  const pageSize = 200;
  let pageNum = 1;
  const optionMap = new Map<number, SongOption>();

  while (true) {
    const result: any = await getSongList({
      pageNum,
      pageSize,
      artistId: null,
      songName: null,
      album: null
    });

    if (result.code !== 0) {
      throw new Error(result.message || "获取歌曲列表失败");
    }

    const items = result?.data?.items;
    if (!Array.isArray(items) || items.length === 0) {
      break;
    }

    items.forEach(item => {
      const songId = Number(item.songId);
      if (!Number.isFinite(songId)) return;
      optionMap.set(songId, {
        key: songId,
        label: `${item.songName || "未知歌曲"} - ${item.artistName || "未知歌手"}${item.album ? ` (${item.album})` : ""}`,
        artistName: item.artistName,
        album: item.album
      });
    });

    const total = Number(result?.data?.total || 0);
    if ((total > 0 && optionMap.size >= total) || items.length < pageSize) {
      break;
    }

    pageNum += 1;
    if (pageNum > 200) {
      break;
    }
  }

  songOptions.value = Array.from(optionMap.values());
}

async function fetchBoundSongIds() {
  const result: any = await getPlaylistSongIds(props.playlistId);
  if (result.code !== 0) {
    throw new Error(result.message || "获取已绑定歌曲失败");
  }

  const ids = Array.isArray(result.data) ? result.data : [];
  selectedSongIds.value = ids
    .map(id => Number(id))
    .filter(id => Number.isFinite(id));
}

async function init() {
  loading.value = true;
  try {
    await Promise.all([fetchAllSongs(), fetchBoundSongIds()]);
  } catch (error: any) {
    message(error?.message || "加载歌单歌曲失败", { type: "error" });
  } finally {
    loading.value = false;
  }
}

function getSelectedSongIds() {
  return [...selectedSongIds.value];
}

defineExpose({
  getSelectedSongIds
});

onMounted(() => {
  init();
});
</script>

<template>
  <div v-loading="loading">
    <el-alert
      :title="`正在管理歌单：${playlistTitle || playlistId}`"
      type="info"
      :closable="false"
      show-icon
      class="mb-3"
    />
    <div class="mb-2 text-[14px] text-[var(--el-text-color-secondary)]">
      已选择 {{ selectedCount }} 首歌曲
    </div>
    <el-transfer
      v-model="selectedSongIds"
      :data="songOptions"
      :titles="['待选歌曲', '已绑定歌曲']"
      filterable
      :filter-method="filterMethod"
      filter-placeholder="输入歌名/歌手/专辑搜索"
      target-order="push"
      class="playlist-song-transfer"
    />
  </div>
</template>

<style scoped lang="scss">
.playlist-song-transfer {
  display: flex;
  justify-content: center;

  :deep(.el-transfer-panel) {
    width: 42%;
    min-width: 320px;
  }

  :deep(.el-transfer-panel__list) {
    height: 340px;
  }
}
</style>

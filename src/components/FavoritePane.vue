<script setup lang="ts">
import {computed, ref, watch} from "vue";
import {Album, commands, FavoriteRespData, FavoriteSort, UserProfileRespData} from "../bindings.ts";
import {useNotification} from "naive-ui";
import AlbumCard from "./AlbumCard.vue";

const notification = useNotification();

const props = defineProps<{
  userProfile: UserProfileRespData | undefined
}>();

const selectedAlbum = defineModel<Album | undefined>("selectedAlbum", {required: true});
const currentTabName = defineModel<"search" | "favorite" | "chapter">("currentTabName", {required: true});

const sortOptions: { label: string, value: string }[] = [
  {label: "收藏时间", value: "FavoriteTime"},
  {label: "更新时间", value: "UpdateTime"},
];

const favoriteRespData = ref<FavoriteRespData>();
const sortSelected = ref<FavoriteSort>("FavoriteTime");
const pageSelected = ref<number>(1);
const folderSelected = ref<number>(0);

const favoritePageCount = computed(() => {
  if (favoriteRespData.value === undefined) {
    return 0;
  }
  const total = parseInt(favoriteRespData.value.total);
  return Math.floor(total / 20) + 1;
});
const folderOptions = computed<{ label: string, value: number }[]>(() => [
  {label: "全部", value: 0},
  ...(favoriteRespData.value?.folder_list || []).map(folder => ({
    label: folder.name,
    value: parseInt(folder.FID),
  }))
]);

async function getFavourite(folderId: number, page: number, sort: FavoriteSort) {
  console.log(folderId, page, sort);
  sortSelected.value = sort;
  pageSelected.value = page;
  const result = await commands.getFavoriteFolder(folderId, page, sort);
  if (result.status === "error") {
    notification.error({title: "获取收藏失败", description: result.error});
    return;
  }
  favoriteRespData.value = result.data;
}

watch(() => props.userProfile, async () => {
  if (props.userProfile === undefined) {
    favoriteRespData.value = undefined;
    return;
  }
  await getFavourite(0, 1, sortSelected.value);
}, {immediate: true});

</script>

<template>
  <div class="h-full flex flex-col">
    <div class="flex">
      <n-select v-model:value="folderSelected"
                :options="folderOptions"
                :show-checkmark="false"
                size="tiny"
                @update-value="getFavourite($event, 1, sortSelected)"/>
      <n-select v-model:value="sortSelected"
                :options="sortOptions"
                :show-checkmark="false"
                size="tiny"
                @update-value="getFavourite(folderSelected, 1, $event)"/>
    </div>
    <div v-if="favoriteRespData!==undefined" class="flex flex-col gap-row-1 overflow-auto p-2">
      <div class="flex flex-col gap-row-2 overflow-auto">
        <album-card v-for="albumInFavorite in favoriteRespData?.list"
                    :key="albumInFavorite.id"
                    :album-info="albumInFavorite"
                    v-model:selected-album="selectedAlbum"
                    v-model:current-tab-name="currentTabName"/>
      </div>
      <n-pagination :page-count="favoritePageCount"
                    :page="pageSelected"
                    @update:page="getFavourite(folderSelected, $event, sortSelected)"/>
    </div>
  </div>
</template>

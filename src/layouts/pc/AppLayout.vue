<template>
  <n-layout>
    <HeaderBar class-name="h-[64px] px-[16px]" />

    <n-layout has-sider class="h-[calc(100vh_-_64px)]">
      <SideMenu />

      <n-layout-content class="overflow-auto">
        <main>
          <div class="sticky top-0 z-199 bg-[var(--n-color)]">
            <div class="flex items-center gap-x-[16px] px-[36px] py-[24px]">
              <h1
                class="inline-block text-[22px] font-bold text-text-primary-light dark:text-text-primary-dark"
              >
                {{ title }}
              </h1>

              <SubPageTitleRefreshButton v-if="route.name === 'Sub'" />
            </div>

            <n-divider :style="{ marginTop: 0, marginBottom: 0 }" />
          </div>

          <div class="px-[36px] py-[24px]">
            <router-view v-slot="{ Component }">
              <Transition name="fade" mode="out-in">
                <component :is="Component" :key="route.fullPath" />
              </Transition>
            </router-view>
          </div>
        </main>
      </n-layout-content>
    </n-layout>
  </n-layout>
</template>

<script setup lang="ts">
import { storeToRefs } from 'pinia';
import { computed } from 'vue';
import { useRoute } from 'vue-router';

import SubPageTitleRefreshButton from '../../components/pc/SubPageTitleRefreshButton.vue';
import { useModuleStore } from '../../store/useModuleStore.ts';
import { useSubscriptionStore } from '../../store/useSubscriptionStore.ts';
import HeaderBar from './HeaderBar.vue';
import SideMenu from './SideMenu.vue';

const { modules } = storeToRefs(useModuleStore());
const { subs, collections } = storeToRefs(useSubscriptionStore());
const getItem = (type: Components.SubType, name: string) => {
  return type === 'sub'
    ? subs.value.find(s => s.name === name)
    : collections.value.find(c => c.name === name);
};

const getModule = (name: string) => {
  return modules.value.find(c => c.name === name);
};

const route = useRoute();
const title = computed(() => {
  if (
    route.name === 'Compare'
    || route.name === 'EditSub'
    || route.name === 'CreateSub'
  ) {
    const type = route.params.type === 'sub' ? '单条订阅' : '组合订阅';
    const item = route.params.name
      ? getItem(
        route.params.type as Components.SubType,
        route.params.name as string,
      )
      : null;

    switch (route.name) {
      case 'Compare':
        return `【${item?.displayName || item?.name || '未知'}】处理结果对比`;
      case 'EditSub':
        return `编辑${type}【${item?.displayName || item?.name || '未知'}】`;
      case 'CreateSub':
        return `创建${type}`;
    }
  }

  if (route.name === 'ModulePage') {
    const module = getModule(route.params.name as string);
    return module ? `模块【${module.displayName}】主页` : '模块主页';
  }

  return route.meta.title;
});
</script>

<style scoped>
.fade-enter-active,
.fade-leave-active {
  transition: all 0.15s;
}

.fade-enter-from {
  opacity: 0;
  transform: translateX(10px);
}

.fade-leave-to {
  opacity: 0;
  transform: translateX(-10px);
}
</style>

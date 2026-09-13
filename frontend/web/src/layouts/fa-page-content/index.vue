<!-- 布局内容 -->
<template>
  <div id="app-scroll-main" class="layout-content" :style="containerStyle">
    <div id="app-content-header">
      <!-- 节日滚动 -->
      <FaFestivalTextScroll />

      <!-- 路由信息调试 -->
      <div
        v-if="isOpenRouteInfo === 'true'"
        class="px-2 py-1.5 mb-3 text-sm text-g-500 bg-g-200 border border-(--default-border) rounded-md"
      >
        router meta：{{ route.meta }}
      </div>
    </div>

    <RouterView v-if="isRefresh" v-slot="{ Component, route: router }" :style="contentStyle">
      <Transition :name="actualTransition" mode="out-in">
        <div v-if="Component" class="route-view-shell flex min-h-0 min-w-0 w-full flex-1 flex-col">
          <!--
            外层缓存「一级出口组件」：目录菜单为壳组件 NestedRouterParent，一级叶子为页面组件。
            此处 KeepAlive 必须常驻，不能按当前路由 meta.keepAlive 做 v-if 开关：
            卸载 KeepAlive 会连同已缓存实例一起销毁，导致每次切回都重新挂载（接口重复请求）。
            叶子的取舍由 include/exclude 表达（include 只在多标签模式下生效）。
          -->
          <KeepAlive :max="10" :include="keepAliveInclude" :exclude="keepAliveExclude">
            <component
              class="fa-page-view min-h-0 min-w-0 w-full flex-1"
              :is="Component"
              :key="routeShellCacheKey(router)"
            />
          </KeepAlive>
        </div>
      </Transition>
    </RouterView>

    <!-- 返回顶部：宽屏滚动容器是 #app-content；窄屏改为文档滚动，target 置空 -->
    <ElBacktop
      :key="backtopTargetKey"
      :target="backtopScrollTarget"
      :right="28"
      :bottom="28"
      class="z-90"
    >
      <FaSvgIcon icon="ri:arrow-up-circle-line" class="text-2xl text-theme" />
    </ElBacktop>
  </div>
</template>
<script setup lang="ts">
/**
 * 布局滚动容器 + 业务路由出口；与 settings.refresh 联动可整体重建 RouterView。
 *
 * 缓存开关数据源：后端菜单 `keep_alive` → MenuProcessor 写入 `meta.keepAlive`；工作栏 tab 随路由写入同一 meta。
 * include / wrapPageWithKeepAlive 均用 `!== false`，与静态路由里显式 `keepAlive: false`、后端布尔字段对齐。
 */
import type { CSSProperties } from "vue";
import { useMediaQuery } from "@vueuse/core";
import { useRoute, useRouter, type RouteLocationNormalizedLoaded } from "vue-router";
import { useSettingsStore, useWorktabStore } from "@stores";

defineOptions({ name: "FaPageContent" });

/**
 * KeepAlive 缓存键（外层出口）。
 *
 * 本处 `RouterView` 处于 depth=1，其 `Component` 是 **一级路由组件**：
 * 目录菜单为壳组件 `NestedRouterParent`，一级叶子（如 /home）为页面组件。
 * 因此缓存单位必须是「壳组件」，不能用叶子路由身份（name/params）当键：
 * 否则同一壳会被缓存成 N 份实例，旧实例只被 move 到游离容器而不销毁，
 * 其内部 RouterView 仍随全局 route 重渲染 —— 切页时目标页面会在所有存活壳里
 * 被重复挂载，同一接口被并发触发 N 次（N = 存活壳数，随缓存淘汰动态变化）。
 * 用壳组件名作键后，同一壳全局只有一个实例，页面的重复挂载随之消失，
 * 叶子级缓存由壳内的 KeepAlive 负责。
 */
function routeShellCacheKey(r: RouteLocationNormalizedLoaded): string {
  const shell = r.matched[1]?.components?.default as { name?: string } | undefined;
  if (shell?.name) return shell.name;
  return r.matched[1]?.path ?? r.path;
}

const route = useRoute();
const router = useRouter();

/**
 * 解析 path 在外层 RouterView 出口（depth=1）实际渲染的组件。
 * KeepAlive 的 include / exclude 按「组件 name」匹配，所以必须回解组件，不能用路由 name；
 * depth=1 命中目录时为壳组件 NestedRouterParent（isShell=true），命中一级叶子时为页面组件。
 */
function resolveDepth1(path: string): { name: string; isShell: boolean } {
  try {
    const matched = router.resolve({ path }).matched;
    const comp = matched[1]?.components?.default as
      | { name?: string; __name?: string }
      | undefined;
    return { name: comp?.name ?? comp?.__name ?? "", isShell: matched.length > 2 };
  } catch {
    return { name: "", isShell: false };
  }
}

const isNarrowViewport = useMediaQuery("(max-width: 800px)");
const backtopScrollTarget = computed(() => (isNarrowViewport.value ? "" : "#app-content"));
const backtopTargetKey = computed(() => (isNarrowViewport.value ? "win" : "main"));

const { pageTransition, containerWidth, refresh, showWorkTab } = storeToRefs(useSettingsStore());
const { keepAliveExclude, opened } = storeToRefs(useWorktabStore());

/**
 * 多标签开启时：只把工作栏已打开标签对应的一级组件名放进 include（组件 name，非路由 name）。
 * - 目录标签：一级组件是壳 NestedRouterParent，必须恒缓存 —— 壳实例一旦销毁，其内部叶子缓存也随之丢失。
 * - 一级叶子标签：按标签自身 keepAlive 取舍。
 * 关闭多标签时不传 include，避免白名单过窄误伤缓存。
 */
const keepAliveInclude = computed(() => {
  if (!showWorkTab.value) return undefined;
  const names = new Set<string>();
  for (const t of opened.value) {
    const { name, isShell } = resolveDepth1(t.path);
    if (name && (isShell || t.keepAlive !== false)) names.add(name);
  }
  // 兜底当前路由：避免 opened 尚未写入时当前页面命中不到白名单而不被缓存
  const current = resolveDepth1(route.path);
  if (current.name && (current.isShell || route.meta.keepAlive !== false)) {
    names.add(current.name);
  }
  return names.size ? Array.from(names) : undefined;
});

const isRefresh = shallowRef(true);
const isOpenRouteInfo = import.meta.env.VITE_OPEN_ROUTE_INFO;

/** 浏览器首次进入：关闭路由过渡动画，避免首屏闪动 */
const isFirstLoad = ref(true);

const actualTransition = computed(() => {
  if (isFirstLoad.value) return "";
  return pageTransition.value;
});

const containerStyle = computed(
  (): CSSProperties => ({
    width: "100%",
    minWidth: 0,
    maxWidth: containerWidth.value,
    flex: "1",
    minHeight: "0",
    display: "flex",
    flexDirection: "column",
  })
);

/** 纵向滚动由外层 `#app-content` 承担，`.layout-content` 仅做限宽居中，路由视图填满剩余高度 */
const contentStyle = computed(
  (): CSSProperties => ({ flex: "1", minHeight: "0", minWidth: 0, width: "100%" })
);

const reload = () => {
  isRefresh.value = false;
  nextTick(() => {
    isRefresh.value = true;
  });
};

watch(refresh, reload, { flush: "post" });

// 组件挂载后标记首次加载完成
onMounted(() => {
  // 延迟一帧，确保首次渲染完成
  nextTick(() => {
    isFirstLoad.value = false;
  });
});
</script>

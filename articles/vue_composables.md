# Vue 3 Composables 实战：组合式函数、响应式参数与资源释放

在多个 Vue 页面中复用交互逻辑时，直接复制 watch、事件监听和组件生命周期代码会使修复成本不断上升。Composables 将有状态的逻辑封装为函数，并让调用组件继续拥有独立的展示界面。本文以浏览器媒体查询和窗口尺寸相关能力为例，介绍可复用组合式函数的接口、生命周期及测试。

## 1. 明确组合式函数的职责

函数可以封装响应式状态及外部资源，但不应决定调用者必须渲染什么样的界面。两个组件可以使用相同函数而拥有不同的展示结构。

```vue
<div>
  <section aria-label="布局偏好">
    <h2>当前布局</h2>
    <p>{{ compact ? '紧凑模式' : '标准模式' }}</p>
  </section>
</div>
```

## 2. 以 ref 保存当前观测值

先实现媒体查询读取。浏览器对象仅在客户端生命周期中访问，避免服务端执行时遇到不存在的 window。

```vue
<div>
  <p role="status">{{ matches ? '满足媒体条件' : '未满足媒体条件' }}</p>
</div>
```

```js
import { ref, onMounted, onUnmounted } from 'vue';
export function useMediaQuery(query) {
  const matches = ref(false);
  let media;
  const sync = () => { matches.value = media.matches; };
  onMounted(() => { media = window.matchMedia(query); sync(); media.addEventListener('change', sync); });
  onUnmounted(() => media?.removeEventListener('change', sync));
  return matches;
}
```

## 3. 让调用方自由组合展示

组合式函数只返回值或方法，不需要将 HTML 字符串交给调用者。模板可以将同一数据用于不同的布局和辅助说明。

```vue
<div>
  <aside v-if="compact" aria-label="快捷操作">
    <button type="button" @click="openMenu">菜单</button>
  </aside>
  <nav v-else aria-label="主导航"><slot name="navigation" /></nav>
</div>
```

```vue
<script setup>
import { useMediaQuery } from './useMediaQuery';
const compact = useMediaQuery('(max-width: 768px)');
</script>
```

## 4. 支持响应式参数

调用方传入 ref 或 getter 时，组合式函数不应只读取一次初始值。Vue 3.3 及以上可以用 toValue 统一读取普通值、ref 和 getter。

```vue
<div>
  <label>自定义媒体条件 <input v-model="query" /></label>
  <p>当前条件：{{ query }}</p>
</div>
```

```js
import { toValue, watch } from 'vue';
watch(() => toValue(query), next => { reconnect(next); });
```

## 5. 用 watchEffect 清理旧订阅

当媒体查询参数变化，旧 MediaQueryList 的监听必须移除。清理回调既处理重新运行也处理 watcher 停止。

```vue
<div>
  <output aria-live="polite">{{ matches ? '匹配' : '不匹配' }}</output>
</div>
```

```js
import { watchEffect, toValue, ref, onMounted } from 'vue';
const matches = ref(false);
const mounted = ref(false);
onMounted(() => { mounted.value = true; });
watchEffect(onCleanup => {
  if (!mounted.value) return;
  const media = window.matchMedia(toValue(query));
  const sync = () => { matches.value = media.matches; };
  sync(); media.addEventListener('change', sync);
  onCleanup(() => media.removeEventListener('change', sync));
});
```

## 6. 通过 readonly 限制状态修改

如果值完全由外部系统决定，调用者不应直接修改它。返回 readonly ref 可以清晰表达状态所有权。

```vue
<div>
  <section><h3>系统偏好</h3><p>{{ prefersDark ? '深色偏好' : '浅色偏好' }}</p></section>
</div>
```

```js
import { readonly } from 'vue';
return { matches: readonly(matches) };
```

## 7. 区分实例独立与全局共享

每次调用 useMediaQuery 都会创建自己的响应式值和监听。多数页面数量有限，这种方式简单；当大量组件共享同一个外部资源时，再评估应用级共享和引用计数。

```vue
<div>
  <section><LayoutToolbar /><LayoutPreview /></section>
</div>
```

不要在支持 SSR 的项目中无条件创建跨请求共享的用户状态。共享浏览器监听应有明确的初始化与销毁策略。

## 8. 给初始化状态建立语义

服务端不能知道客户端实际媒体条件。若必须区分尚未测量与不匹配，应提供 ready 状态，而不是把初始 false 当成真实结果。

```vue
<div>
  <p v-if="!ready" role="status">正在读取设备偏好</p>
  <p v-else>{{ matches ? '启用紧凑布局' : '启用默认布局' }}</p>
</div>
```

客户端挂载前保留确定性结构，有助于避免水合阶段不一致。纯样式响应式问题仍然优先使用 CSS。

## 9. 测试监听与清理

使用模拟 matchMedia 的测试环境，检查初始读取、change 回调和组件卸载。模拟对象需要支持 addEventListener 与 removeEventListener。

```vue
<div>
  <button type="button" @click="showPreview = !showPreview">切换预览</button>
  <PreviewPanel v-if="showPreview" />
</div>
```

测试中可以记录订阅数量：挂载后增加，卸载后恢复。参数变化后触发旧对象的回调，应当不再影响当前组件。

## 10. 统一组合式函数 API 风格

推荐返回具名 ref 和操作方法，调用方解构后仍保留响应性。将参数默认值、支持范围及错误行为写入函数契约。

```vue
<div>
  <section><p>{{ matches ? '当前符合条件' : '当前不符合条件' }}</p><small v-if="!ready">尚未完成初始化</small></section>
</div>
```

在组件 setup 或有效作用域内调用依赖生命周期的 composable；不要在随意的异步回调中首次注册组件生命周期钩子。

## 总结

Composables 的价值在于复用状态与生命周期逻辑，而不是增加组件间隐式依赖。明确响应式输入、输出所有权以及资源清理，可以让同一能力在多个 Vue 组件中稳定复用。

## 热搜标题与本站资料

以下热搜标题仅作当日资讯记录；本站链接为独立资料，并非该新闻的报道。

- 热搜：分享你拍的超级月亮；本站资料：<https://www.yxtiktok.com/blog/new-tiktok-agency-first-100-creator-leads-plan/>
- 热搜：北京天空现“超级月亮”；本站资料：<https://www.yxtiktok.com/blog/tiktok-game-streamer-audience-interaction/>
- 热搜：冯远征笑称“考北京人艺其实不难”；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-creator-data-privacy-access-control/>
- 热搜：女子登山上班往返约6小时已瘦12斤；本站资料：<https://www.yxtiktok.com/blog/build-tiktok-agency-creator-selection-criteria/>
- 热搜：华为Mate 90 Air售价4999元起；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-streamer-manager-performance-kpis/>
- 热搜：华为首款小折叠电脑发布；本站资料：<https://www.yxtiktok.com/blog/avoid-duplicate-tiktok-creator-collection-and-followup/>
- 热搜：中考要不要考历史 教材主编征集意见；本站资料：<https://www.yxtiktok.com/blog/tiktok-creator-weeks-performance-decline/>
- 热搜：上海市委书记陈吉宁被查；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-first-contact-message-to-creator/>
- 热搜：大众拟裁员10万人 工会威胁罢工；本站资料：<https://www.yxtiktok.com/blog/agency-invite-sent-creator-not-accepted-follow-up/>
- 热搜：受贿1.13亿余元！吴胜华被判无期；本站资料：<https://www.yxtiktok.com/blog/filter-overseas-tiktok-creators-by-language/>
- 热搜：动辄千元起步的户外徒步鞋火了；本站资料：<https://www.yxtiktok.com/blog/tiktok-part-time-vs-full-time-streamer-management/>
- 热搜：重庆男孩被坠楼男子砸中送医；本站资料：<https://www.yxtiktok.com/blog/early-signs-streamer-unhappy-with-agency/>
- 热搜：伪人AI广告 已经成了一种新的城市病；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-first-call-creator-needs-questions/>
- 热搜：换脸还没消停 “AI融脸”又来了；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-agency-differentiated-creator-offer/>
- 热搜：大爷抵脖子不放后小伙倒地 地铁回应；本站资料：<https://www.yxtiktok.com/blog/tiktok-pk-gifts-live-data-review/>
- 热搜：一校一主题 消谣育青年；本站资料：<https://www.yxtiktok.com/blog/screen-tiktok-creator-public-live-content-policy-risk/>
- 热搜：天还没亮 学生就集体站在操场晨读；本站资料：<https://www.yxtiktok.com/blog/what-is-tiktok-live-backstage-agency-functions/>
- 热搜：香奈儿包日租仅需25元？押金要一万六；本站资料：<https://www.yxtiktok.com/blog/why-agencies-should-not-rely-only-on-tiktok-top-rankings/>
- 热搜：车企老板娘女装卖了30亿；本站资料：<https://www.yxtiktok.com/blog/single-country-tiktok-agency-creator-pipeline/>
- 热搜：新娘提1341次分手新郎回1342次不行；本站资料：<https://www.yxtiktok.com/blog/benefits-of-joining-tiktok-live-agency/>
- 热搜：中国城市 重排全球座次；本站资料：<https://www.yxtiktok.com/blog/tiktok-creator-discovery-ranking-live-profile-workflow/>
- 热搜：央行将开展8000亿元MLF操作；本站资料：<https://www.yxtiktok.com/blog/why-tiktok-creator-invite-eligibility-changes/>
- 热搜：松鼠妈妈六次示范教小松鼠跳树成功；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-replay-review-method/>
- 热搜：培养更多高技能人才和大国工匠；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-creator-tiered-management/>
- 热搜：东北外卖15分钟冻成冰；本站资料：<https://www.yxtiktok.com/blog/avoid-spam-repeated-dm-tiktok-agency-recruitment/>
- 热搜：飞机与“超级月亮”浪漫同框；本站资料：<https://www.yxtiktok.com/blog/tiktok-new-creator-coaching-payback-period/>
- 热搜：黄奕用AI视频还原过去三十年人生；本站资料：<https://www.yxtiktok.com/blog/tiktok-dance-live-content-interaction-plan/>
- 热搜：特朗普连发30多张照片“回味”深圳；本站资料：<https://www.yxtiktok.com/blog/talk-to-new-tiktok-live-creator-needs/>
- 热搜：太平洋暖得“失控”了；本站资料：<https://www.yxtiktok.com/blog/growth-vs-established-tiktok-creators-recruitment-talk/>
- 热搜：Anthropic上市5天即破发；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-host-review-meeting-metrics/>
- 热搜：田秉毅任国家羽毛球队主教练；本站资料：<https://www.yxtiktok.com/blog/evaluate-tiktok-live-leaderboard-for-recruitment-list/>
- 热搜：旺仔牛奶更不好卖了；本站资料：<https://www.yxtiktok.com/blog/clean-invalid-stale-tiktok-creator-lead-records/>
- 热搜：几个姑娘在-25℃的黑龙江清凉上阵；本站资料：<https://www.yxtiktok.com/blog/why-tiktok-agency-deduplicate-creator-uid/>
- 热搜：特朗普致电黄仁勋：答应我一件事；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-feed-creator-recruitment/>
- 热搜：“民主党向黄仁勋和英伟达宣战”；本站资料：<https://www.yxtiktok.com/blog/find-tiktok-live-creators-streaming-now/>
- 热搜：中国人登月又近一步；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-recruitment-support-pitch/>
- 热搜：课堂上的“嘉豪、嘉欣” 已不敢举手；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-recruitment-communication-tips/>
- 热搜：国产可口可乐不再标生产日期；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-pk-interaction-planning/>
- 热搜：年产1500万根的“棍子” 卖出10亿元；本站资料：<https://www.yxtiktok.com/blog/tiktok-coins-gifts-diamonds-difference/>
- 热搜：加州给548名死刑犯减刑 特朗普震怒；本站资料：<https://www.yxtiktok.com/blog/tiktok-streamer-recruitment-channels-by-country/>
- 热搜：黄磊真空围裙；本站资料：<https://www.yxtiktok.com/blog/can-individual-apply-tiktok-live-creator-network/>
- 热搜：19楼落地窗底部装无限位外凸窗；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-recruitment-promise-contract-check/>
- 热搜：Mate X8 华为史上最强大折叠；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-low-live-hours-diagnosis/>
- 热搜：泰警方确认女子异地遇害被肢解抛尸；本站资料：<https://www.yxtiktok.com/blog/new-tiktok-agency-creator-pool-and-roles/>
- 热搜：余承东现场提词器曝光；本站资料：<https://www.yxtiktok.com/blog/tiktok-new-streamer-training-curriculum/>
- 热搜：睡眠大赛4人躺赢平分6000元；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-creator-live-schedule/>
- 热搜：注水长剧的“狼”来了；本站资料：<https://www.yxtiktok.com/blog/assess-third-party-tiktok-live-ranking-coverage/>
- 热搜：文吉儿从小就活跃在各大荧幕；本站资料：<https://www.yxtiktok.com/blog/schedule-cross-timezone-tiktok-creator-recruiting-team/>
- 热搜：外交部回应“黄仁勋重回英伟达”；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-recruitment-channels/>
- 热搜：小米Q3净利润48.84亿 同比减少56.8%；本站资料：<https://www.yxtiktok.com/blog/small-tiktok-live-agency-essential-team-roles/>

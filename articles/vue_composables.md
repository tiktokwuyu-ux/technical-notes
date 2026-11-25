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

## 本站 URL 数据样本

以下为本站公开页面地址样本，与上文技术示例及下方热搜无关。

```text
https://www.yxtiktok.com/blog/tiktok-singing-live-setlist-interaction/
https://www.yxtiktok.com/blog/schedule-cross-timezone-tiktok-creator-recruiting-team/
https://www.yxtiktok.com/blog/which-live-metrics-tiktok-agency-should-evaluate/
https://www.yxtiktok.com/blog/prevent-tiktok-agency-recruiter-duplicate-contact/
https://www.yxtiktok.com/blog/tiktok-creator-pipeline-status-management/
https://www.yxtiktok.com/blog/tiktok-live-awkward-silence-topic-switching/
https://www.yxtiktok.com/blog/creator-not-interested-agency-recruitment-status/
https://www.yxtiktok.com/blog/tiktok-agency-first-contact-message-to-creator/
https://www.yxtiktok.com/blog/tiktok-agency-recruitment-message-cooperation-terms/
https://www.yxtiktok.com/blog/multi-country-tiktok-agency-team-language-allocation/
https://www.yxtiktok.com/blog/tiktok-live-league-tiers-creator-discovery/
https://www.yxtiktok.com/blog/why-tiktok-live-ranking-differs-from-real-performance/
https://www.yxtiktok.com/blog/tiktok-backstage-creator-management-relationship/
https://www.yxtiktok.com/blog/small-tiktok-live-agency-priority-creator-types/
https://www.yxtiktok.com/blog/tiktok-live-entertainment-gaming-shopping-creators/
https://www.yxtiktok.com/blog/tiktok-agency-creator-incentive-communication/
https://www.yxtiktok.com/blog/tiktok-host-recruitment-cost-per-signed-creator/
https://www.yxtiktok.com/blog/new-tiktok-agency-first-month-action-plan/
https://www.yxtiktok.com/blog/tiktok-agency-first-call-creator-needs-questions/
https://www.yxtiktok.com/blog/manual-vs-software-tiktok-streamer-recruitment/
https://www.yxtiktok.com/blog/tiktok-live-pk-interaction-planning/
https://www.yxtiktok.com/blog/tiktok-streamer-performance-decline-recovery-plan/
https://www.yxtiktok.com/blog/normal-invite-only-creator-agency-choice/
https://www.yxtiktok.com/blog/agency-invite-sent-creator-not-accepted-follow-up/
https://www.yxtiktok.com/blog/solo-vs-team-tiktok-agency-recruitment-workflow/
https://www.yxtiktok.com/blog/assess-tiktok-creator-live-positioning-from-profile/
https://www.yxtiktok.com/blog/tiktok-new-creator-coaching-payback-period/
https://www.yxtiktok.com/blog/evaluate-tiktok-live-leaderboard-for-recruitment-list/
https://www.yxtiktok.com/blog/tiktok-agency-streamer-referral-recruitment/
https://www.yxtiktok.com/blog/tiktok-agency-recruiter-performance-kpis/
https://www.yxtiktok.com/blog/tiktok-game-streamer-audience-interaction/
https://www.yxtiktok.com/blog/tiktok-agency-daily-recruitment-plan/
https://www.yxtiktok.com/blog/avoid-spam-repeated-dm-tiktok-agency-recruitment/
https://www.yxtiktok.com/blog/tiktok-streamer-not-responding-agency/
https://www.yxtiktok.com/blog/why-invitable-tiktok-creator-may-not-join/
https://www.yxtiktok.com/blog/remote-tiktok-agency-team-shared-creator-data/
https://www.yxtiktok.com/blog/tiktok-agency-first-recruitment-dm-scripts/
https://www.yxtiktok.com/blog/build-tiktok-agency-creator-selection-criteria/
https://www.yxtiktok.com/blog/why-high-earning-tiktok-creators-may-not-fit-agency/
https://www.yxtiktok.com/blog/why-tiktok-live-ranking-platforms-disagree/
https://www.yxtiktok.com/blog/tiktok-agency-many-leads-low-signup-rate/
https://www.yxtiktok.com/blog/tiktok-diamonds-vs-creator-rewards/
https://www.yxtiktok.com/blog/tiktok-live-creator-network-violation-consequences/
https://www.yxtiktok.com/blog/filter-overseas-tiktok-creators-by-language/
https://www.yxtiktok.com/blog/tiktok-streamer-income-expectations-management/
https://www.yxtiktok.com/blog/tiktok-agency-recruitment-promise-contract-check/
https://www.yxtiktok.com/blog/tiktok-recruitment-channel-conversion-analysis/
https://www.yxtiktok.com/blog/select-tiktok-entertainment-live-creator-content-types/
https://www.yxtiktok.com/blog/reduce-duplicate-work-multiple-tiktok-agency-accounts/
https://www.yxtiktok.com/blog/tiktok-live-events-vs-campaigns/
```

## 热搜标题

以下仅为采集时的资讯标题记录。

- 同心合锦 织向未来
- 特朗普一直夸一直夸 完全停不下来
- “转人工转人工 给我转人工”
- 神舟二十四号发射成功
- 中外航天员实现首次“太空握手”
- 近四成低龄老年人愿意继续工作
- “数学回不到从前了”
- 央视揭秘走私孕妇血样大案
- 《再见老板》要来了？网友疯狂提名
- 詹姆斯：背后说我坏话 当面找我握手
- 《丑狐》凭什么让全网催更
- 护卫银发与童心 让网络温暖无虞
- 高以忱获刑十一年
- 女研究生一行3人青海自驾游失联
- 河南老中医收藏一套清朝手术刀
- 中方回应“美将对中国等国加征关税”
- 巴基斯坦航天员在天宫中文唠嗑
- 国台办回应“欧阳娜娜几度哽咽”
- “面具之下 谁都可以是赛伊德”
- 老电影翻新 一部能赚多少钱
- 张哲华 龙傲天和喜剧之后
- 地级市市长转任县委书记
- 大三男生做保姆 入职俩月胖了近30斤
- 儿子目睹父亲被杀患上严重精神疾病
- “最火性转马尔福”作者是中专生
- 2026年环球小姐冠军诞生
- 迪丽热巴“最佳红毯着装”
- 全国艾滋病治疗成功比例达95%以上
- 郭涛儿子石头：希望尽量赶上爸爸
- 《变形计》“最像姐妹的母女”
- 台湾屏东县海域发生4.8级地震
- 胡海泉“悟道”被网友质疑炒股亏了
- 宇宙级全家福集齐9宫格
- 中外航天员在中国空间站合影
- 女子穿楚服看楚展被误认穿“和服”
- 求婚遭打断要求免单是小题大做吗
- “世界超市”又爆单了
- 小区健身器材才是真宝藏
- 三种流感毒株叠加来袭
- 南方人快看 大风吹雪拉出“长毛”
- 厂家称直播灌装荔枝冰酒都是造假
- 前日本女乒名将石川佳纯宣布结婚
- 女子每天从保定通勤6小时到北京上班
- 神23神24航天员今天一起吃午饭
- 首位已婚已育环球小姐冠军诞生
- 山寨“塔斯汀”开出1300家门店
- 马来西亚总理：中国南海谈判很讲道理
- 巴基斯坦总理：感谢兄弟般的中国
- 巴驻华大使：好好学中文 万一上太空了
- 斗鱼及创始人陈少杰被执行1.9亿

<!-- publisher:github-format=2 -->

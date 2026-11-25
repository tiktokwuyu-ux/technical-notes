# Vue Router 4 实战：导航守卫、权限路由与离开页面确认

后台系统在切换页面时不仅要改变路由，还要验证会话、判断访问权限并保护未保存的编辑内容。Vue Router 4 提供多个导航守卫，可以将这些逻辑放在适当的阶段。本文讨论客户端路由控制，不把它当成服务端权限检查的替代品。

## 1. 定义路由及元信息契约

使用 meta 表达页面访问要求，而不是在各个组件中分别判断路径字符串。页面名称和访问规则集中定义。

```vue
<div>
  <nav aria-label="后台导航"><RouterLink :to="{ name: 'dashboard' }">工作台</RouterLink></nav>
</div>
```

```ts
const routes = [
  { path: '/login', name: 'login', component: LoginPage },
  { path: '/dashboard', name: 'dashboard', component: DashboardPage, meta: { requiresAuth: true } }
];
```

## 2. 创建全局前置守卫

beforeEach 适合跨页面通用的会话检查。Vue Router 4 可以通过返回 false 取消导航，或者返回新的路由位置进行重定向。

```vue
<div>
  <main><RouterView /></main>
</div>
```

```ts
router.beforeEach(async to => {
  if (!to.meta.requiresAuth) return true;
  await session.ensureInitialized();
  return session.authenticated || { name: 'login', query: { returnTo: to.fullPath } };
});
```

## 3. 合并会话初始化请求

用户可能快速点击多个路由。如果每次守卫都发起独立会话请求，会造成重复网络访问。会话服务可以共享尚未完成的初始化 Promise，并在失败后释放它。

```vue
<div>
  <p v-if="session.initializing" role="status">正在确认登录状态</p>
</div>
```

```ts
let initialization: Promise<void> | undefined;
function ensureInitialized() {
  if (session.ready) return Promise.resolve();
  return initialization ??= readSession().finally(() => { initialization = undefined; });
}
```

## 4. 区分未登录和权限不足

认证成功不等于可以访问每个模块。角色规则可用于界面导航过滤，但 API 仍必须检查真实权限。

```vue
<div>
  <section role="alert"><h2>没有访问权限</h2><p>请联系系统管理员确认权限。</p></section>
</div>
```

```ts
router.beforeEach(to => {
  const permission = to.meta.permission;
  if (typeof permission === 'string' && !session.can(permission)) return { name: 'forbidden' };
});
```

## 5. 安全处理登录后的返回路径

登录页接收 returnTo 时不能直接信任任意 URL。只接受本应用允许的站内路径，并通过路由解析和业务白名单验证。

```vue
<div>
  <form @submit.prevent="signIn"><button type="submit">登录并继续</button></form>
</div>
```

```ts
function resolveReturnTo(input: unknown) {
  if (typeof input !== 'string' || !input.startsWith('/') || input.startsWith('//') || input.includes('\\')) return { name: 'dashboard' };
  const resolved = router.resolve(input);
  return resolved.matched.length && resolved.name !== 'login' ? resolved.fullPath : { name: 'dashboard' };
}
```

## 6. 阻止离开尚未保存的页面

onBeforeRouteLeave 适合局部编辑页面。应只在确实有未保存更改时弹出确认，并在保存成功后更新基准状态。

```vue
<div>
  <section><textarea v-model="draft" aria-label="编辑备注" /><p v-if="dirty">尚未保存</p></section>
</div>
```

```ts
import { onBeforeRouteLeave } from 'vue-router';
onBeforeRouteLeave(async () => {
  if (!dirty.value) return true;
  return await confirmDiscardChanges();
});
```

## 7. 处理同一路由参数变化

从一个详情 ID 切换到另一个 ID 时，Vue Router 可能复用同一组件实例。仅依赖 onMounted 加载数据会展示旧记录。

```vue
<div>
  <article><h2>记录详情</h2><p>{{ record?.title }}</p></article>
</div>
```

```ts
watch(() => route.params.id, (id, _, onCleanup) => {
  const controller = new AbortController();
  onCleanup(() => controller.abort());
  void loadRecord(String(id), controller.signal);
}, { immediate: true });
```

## 8. 识别导航取消与真正错误

router.push 的 Promise 可以返回导航失败对象。正常取消不应被当成服务端错误弹出。

```vue
<div>
  <button type="button" @click="goToSettings">打开设置</button>
  <p v-if="navigationMessage" role="status">{{ navigationMessage }}</p>
</div>
```

```ts
import { isNavigationFailure, NavigationFailureType } from 'vue-router';
const failure = await router.push({ name: 'settings' });
if (isNavigationFailure(failure, NavigationFailureType.aborted)) navigationMessage.value = '已取消页面切换';
```

## 9. 在导航完成后执行展示更新

afterEach 不能再阻止导航，但可以用于更新页面标题或清理界面提示。接收 failure 参数，避免在失败时误报页面已经切换。

```vue
<div>
  <header><h1>{{ route.meta.title ?? '工作台' }}</h1></header>
</div>
```

```ts
router.afterEach((to, _from, failure) => {
  if (!failure && typeof to.meta.title === 'string') document.title = to.meta.title;
});
```

## 10. 验证守卫顺序与恢复行为

测试未登录访问、会话失效、权限不足、保存失败后离开及快速切换。用户关闭浏览器或刷新页面不是普通路由导航，需要单独考虑浏览器的离开确认机制。

```vue
<div>
  <ul aria-label="导航验收"><li>登录后恢复合法路径</li><li>未保存时可取消导航</li><li>权限不足时展示独立页面</li></ul>
</div>
```

## 总结

导航守卫应按照职责分层：全局会话、权限规则和局部编辑保护分别管理。结合明确的导航失败语义、可取消的数据读取以及服务端授权，才能让复杂后台导航保持一致。

## 本站 URL 数据样本

以下为本站公开页面地址样本，与上文技术示例及下方热搜无关。

```text
https://www.yxtiktok.com/blog/tiktok-agency-recruitment-channels/
https://www.yxtiktok.com/blog/tiktok-creator-no-reply-recruitment-follow-up/
https://www.yxtiktok.com/blog/assess-third-party-tiktok-live-ranking-coverage/
https://www.yxtiktok.com/blog/find-largest-dropoff-tiktok-agency-recruitment-funnel/
https://www.yxtiktok.com/blog/three-tier-tiktok-creator-lead-priority-system/
https://www.yxtiktok.com/blog/tiktok-pk-live-recruitment/
https://www.yxtiktok.com/blog/avoid-duplicate-tiktok-creator-collection-and-followup/
https://www.yxtiktok.com/blog/daily-tiktok-agency-recruitment-report-kpis/
https://www.yxtiktok.com/blog/top-vs-mid-tier-tiktok-live-creators-agency/
https://www.yxtiktok.com/blog/tiktok-new-vs-experienced-streamer-outreach/
https://www.yxtiktok.com/blog/tiktok-agency-recruitment-message-cooperation-terms/
https://www.yxtiktok.com/blog/why-recheck-creator-eligibility-before-contact/
https://www.yxtiktok.com/blog/tiktok-host-recruitment-cost-per-signed-creator/
https://www.yxtiktok.com/blog/tiktok-agency-recruitment-funnel-from-lead-to-join/
https://www.yxtiktok.com/blog/tiktok-live-awkward-silence-topic-switching/
https://www.yxtiktok.com/blog/evaluate-tiktok-live-leaderboard-for-recruitment-list/
https://www.yxtiktok.com/blog/why-invitable-tiktok-creator-may-not-join/
https://www.yxtiktok.com/blog/tiktok-followers-vs-live-engagement-recruiting/
https://www.yxtiktok.com/blog/tiktok-live-agency-differentiated-creator-offer/
https://www.yxtiktok.com/blog/tiktok-agency-first-recruitment-dm-scripts/
https://www.yxtiktok.com/blog/tiktok-live-creator-agency-contract-checklist/
https://www.yxtiktok.com/blog/new-tiktok-agency-first-month-action-plan/
https://www.yxtiktok.com/blog/remote-tiktok-agency-team-shared-creator-data/
https://www.yxtiktok.com/blog/tiktok-creator-network-cross-region-compliance/
https://www.yxtiktok.com/blog/tiktok-agency-recruitment-support-pitch/
https://www.yxtiktok.com/blog/clean-invalid-stale-tiktok-creator-lead-records/
https://www.yxtiktok.com/blog/small-tiktok-live-agency-essential-team-roles/
https://www.yxtiktok.com/blog/find-tiktok-live-creators-streaming-now/
https://www.yxtiktok.com/blog/tiktok-agency-daily-recruitment-plan/
https://www.yxtiktok.com/blog/calculate-tiktok-creator-recruitment-conversion-rate/
https://www.yxtiktok.com/blog/screen-tiktok-creator-public-live-content-policy-risk/
https://www.yxtiktok.com/blog/tiktok-streamer-income-expectations-management/
https://www.yxtiktok.com/blog/how-tiktok-agency-prove-identity-to-creator/
https://www.yxtiktok.com/blog/solo-vs-team-tiktok-agency-recruitment-workflow/
https://www.yxtiktok.com/blog/agency-invite-sent-creator-not-accepted-follow-up/
https://www.yxtiktok.com/blog/why-tiktok-creator-invite-eligibility-changes/
https://www.yxtiktok.com/blog/tiktok-creator-pipeline-status-management/
https://www.yxtiktok.com/blog/tiktok-agency-creator-incentive-communication/
https://www.yxtiktok.com/blog/tiktok-agency-streamer-onboarding-handoff/
https://www.yxtiktok.com/blog/tiktok-agency-recruitment-messages-lose-trust/
https://www.yxtiktok.com/blog/tiktok-agency-one-on-one-streamer-support/
https://www.yxtiktok.com/blog/assign-tiktok-creator-leads-country-language-stage/
https://www.yxtiktok.com/blog/tiktok-chat-streamer-improvisation-training/
https://www.yxtiktok.com/blog/tiktok-agency-streamer-manager-performance-kpis/
https://www.yxtiktok.com/blog/talk-to-new-tiktok-live-creator-needs/
https://www.yxtiktok.com/blog/tiktok-game-streamer-audience-interaction/
https://www.yxtiktok.com/blog/new-tiktok-agency-first-100-creator-leads-plan/
https://www.yxtiktok.com/blog/tiktok-streamer-not-responding-agency/
https://www.yxtiktok.com/blog/tiktok-agency-recruiter-performance-kpis/
https://www.yxtiktok.com/blog/tiktok-agency-streamer-training-feedback-system/
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

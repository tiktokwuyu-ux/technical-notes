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

## 热搜标题与本站资料

以下热搜标题仅作当日资讯记录；本站链接为独立资料，并非该新闻的报道。

- 热搜：外交部回应“黄仁勋重回英伟达”；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-diamonds-stability-vs-one-day-peak/>
- 热搜：Mate X8 华为史上最强大折叠；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-agency-differentiated-creator-offer/>
- 热搜：上海市委书记陈吉宁被查；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-recruitment-support-pitch/>
- 热搜：动辄千元起步的户外徒步鞋火了；本站资料：<https://www.yxtiktok.com/blog/tiktok-game-streamer-audience-interaction/>
- 热搜：冯远征笑称“考北京人艺其实不难”；本站资料：<https://www.yxtiktok.com/tutorials/tiktok-ios-non-jailbreak-dm-tutorial/>
- 热搜：飞机与“超级月亮”浪漫同框；本站资料：<https://www.yxtiktok.com/blog/reassess-long-unfollowed-tiktok-creator-leads/>
- 热搜：大众拟裁员10万人 工会威胁罢工；本站资料：<https://www.yxtiktok.com/blog/why-high-earning-tiktok-creators-may-not-fit-agency/>
- 热搜：年产1500万根的“棍子” 卖出10亿元；本站资料：<https://www.yxtiktok.com/blog/build-tiktok-agency-creator-selection-criteria/>
- 热搜：小米Q3净利润48.84亿 同比减少56.8%；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-streamer-maintenance-schedule/>
- 热搜：重庆男孩被坠楼男子砸中送医；本站资料：<https://www.yxtiktok.com/blog/how-to-check-tiktok-creator-agency-invite-eligibility/>
- 热搜：车企老板娘女装卖了30亿；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-creator-incentive-communication/>
- 热搜：特朗普连发30多张照片“回味”深圳；本站资料：<https://www.yxtiktok.com/blog/unified-tiktok-creator-recruitment-records/>
- 热搜：香奈儿包日租仅需25元？押金要一万六；本站资料：<https://www.yxtiktok.com/blog/tiktok-creator-discovery-ranking-live-profile-workflow/>
- 热搜：受贿1.13亿余元！吴胜华被判无期；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-low-live-hours-diagnosis/>
- 热搜：培养更多高技能人才和大国工匠；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-recruitment-funnel-from-lead-to-join/>
- 热搜：央行将开展8000亿元MLF操作；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-events-vs-campaigns/>
- 热搜：中国人登月又近一步；本站资料：<https://www.yxtiktok.com/blog/schedule-cross-timezone-tiktok-creator-recruiting-team/>
- 热搜：几个姑娘在-25℃的黑龙江清凉上阵；本站资料：<https://www.yxtiktok.com/blog/tiktok-creator-network-creator-relationship/>
- 热搜：文吉儿从小就活跃在各大荧幕；本站资料：<https://www.yxtiktok.com/blog/tiktok-pk-live-recruitment/>
- 热搜：华为首款小折叠电脑发布；本站资料：<https://www.yxtiktok.com/blog/small-tiktok-live-agency-essential-team-roles/>
- 热搜：松鼠妈妈六次示范教小松鼠跳树成功；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-recruitment-communication-tips/>
- 热搜：旺仔牛奶更不好卖了；本站资料：<https://www.yxtiktok.com/blog/record-tiktok-creator-invite-check-history/>
- 热搜：女子登山上班往返约6小时已瘦12斤；本站资料：<https://www.yxtiktok.com/blog/does-higher-tiktok-live-league-mean-better-recruit/>
- 热搜：睡眠大赛4人躺赢平分6000元；本站资料：<https://www.yxtiktok.com/blog/what-is-tiktok-live-backstage-agency-functions/>
- 热搜：东北外卖15分钟冻成冰；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-streamer-referral-recruitment/>
- 热搜：课堂上的“嘉豪、嘉欣” 已不敢举手；本站资料：<https://www.yxtiktok.com/blog/new-tiktok-agency-first-month-action-plan/>
- 热搜：一校一主题 消谣育青年；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-recruiter-offboarding-leads-handover/>
- 热搜：Anthropic上市5天即破发；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-streamer-training-feedback-system/>
- 热搜：新娘提1341次分手新郎回1342次不行；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-replay-review-method/>
- 热搜：华为Mate 90 Air售价4999元起；本站资料：<https://www.yxtiktok.com/blog/new-tiktok-agency-creator-pool-and-roles/>
- 热搜：中国城市 重排全球座次；本站资料：<https://www.yxtiktok.com/blog/tiktok-short-video-creator-recruitment/>
- 热搜：分享你拍的超级月亮；本站资料：<https://www.yxtiktok.com/blog/tiktok-streamer-recruitment-channels-by-country/>
- 热搜：19楼落地窗底部装无限位外凸窗；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-streamer-segmented-management/>
- 热搜：换脸还没消停 “AI融脸”又来了；本站资料：<https://www.yxtiktok.com/blog/single-country-tiktok-agency-creator-pipeline/>
- 热搜：“民主党向黄仁勋和英伟达宣战”；本站资料：<https://www.yxtiktok.com/blog/tiktok-singing-live-setlist-interaction/>
- 热搜：余承东现场提词器曝光；本站资料：<https://www.yxtiktok.com/blog/tiktok-country-ranking-find-live-creators/>
- 热搜：中考要不要考历史 教材主编征集意见；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-handle-creator-live-violation/>
- 热搜：加州给548名死刑犯减刑 特朗普震怒；本站资料：<https://www.yxtiktok.com/blog/avoid-duplicate-tiktok-creator-collection-and-followup/>
- 热搜：特朗普致电黄仁勋：答应我一件事；本站资料：<https://www.yxtiktok.com/blog/creator-already-in-other-network-new-invite/>
- 热搜：大爷抵脖子不放后小伙倒地 地铁回应；本站资料：<https://www.yxtiktok.com/blog/assess-third-party-tiktok-live-ranking-coverage/>
- 热搜：田秉毅任国家羽毛球队主教练；本站资料：<https://www.yxtiktok.com/blog/tiktok-host-recruitment-cost-per-signed-creator/>
- 热搜：北京天空现“超级月亮”；本站资料：<https://www.yxtiktok.com/blog/three-tier-tiktok-creator-lead-priority-system/>
- 热搜：黄奕用AI视频还原过去三十年人生；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-welcome-viewers-scripts/>
- 热搜：天还没亮 学生就集体站在操场晨读；本站资料：<https://www.yxtiktok.com/blog/tiktok-creator-network-multi-account-invite-risk/>
- 热搜：泰警方确认女子异地遇害被肢解抛尸；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-ranking-find-creators/>
- 热搜：太平洋暖得“失控”了；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-cross-country-invite-region-rules/>
- 热搜：国产可口可乐不再标生产日期；本站资料：<https://www.yxtiktok.com/blog/normal-invite-only-creator-agency-choice/>
- 热搜：黄磊真空围裙；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-streamer-feedback-follow-up/>
- 热搜：伪人AI广告 已经成了一种新的城市病；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-recruitment-message-cooperation-terms/>
- 热搜：注水长剧的“狼”来了；本站资料：<https://www.yxtiktok.com/blog/how-tiktok-agency-prove-identity-to-creator/>

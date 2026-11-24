# JavaScript History API 实战：单页导航、前进后退与滚动恢复

单页应用的导航不仅是替换页面内容，还要维护浏览器历史、处理返回按钮、恢复滚动和避免旧请求覆盖新页面。本文使用原生 History API 组织一个小型文档管理界面，说明路由器背后的基本机制。示例使用固定路由表描述业务契约，生产环境应将路径定义集中维护。

## 1. 设计语义化导航

普通导航应保留真实的锚点和可访问名称。JavaScript 只增强站内可处理的导航，不应阻止打开新标签页等浏览器原生操作。

```html
<div>
  <nav aria-label="主导航">
    <a href="/workspace">工作台</a>
    <a href="/workspace/settings">设置</a>
  </nav>
</div>
```

## 2. 建立路由匹配入口

将路径匹配与渲染逻辑分开。对于未知路径返回明确的未找到视图，避免根据任意输入动态执行模块。

```html
<div>
  <main data-route-view tabindex="-1">
    <h1>工作台</h1>
  </main>
</div>
```

```js
const routes = new Map([
  ['/workspace', renderWorkspace],
  ['/workspace/settings', renderSettings]
]);
function matchRoute(url) { return routes.get(url.pathname) ?? renderNotFound; }
```

## 3. 使用 pushState 提交导航

pushState 修改地址并新增历史记录，但不会自动加载页面，也不会触发 popstate。应用必须主动执行对应的渲染。

```html
<div>
  <button type="button" data-open-settings>打开设置</button>
</div>
```

```js
function navigate(path) {
  const url = new URL(path, location.href);
  if (url.origin !== location.origin) throw new Error('跨源路径');
  history.pushState({}, '', url);
  return renderCurrentRoute();
}
```

## 4. 正确拦截站内链接

只处理正常左键点击、无修饰键、同源且路由器能够识别的链接。带 download、外部地址和非默认 target 的链接交给浏览器。

```html
<div>
  <nav data-router-links>
    <a href="/workspace/settings">账户设置</a>
    <a href="/workspace" target="_blank">新窗口打开工作台</a>
  </nav>
</div>
```

```js
document.querySelector('[data-router-links]').onclick = e => {
  const a = e.target.closest('a[href]');
  if (!a || e.defaultPrevented || e.button !== 0 || e.metaKey || e.ctrlKey || e.shiftKey || e.altKey) return;
  const url = new URL(a.href);
  if (a.hasAttribute('download') || (a.target && a.target !== '_self') || url.origin !== location.origin || !routes.has(url.pathname) || url.hash) return;
  e.preventDefault(); void navigate(url.href);
};
```

## 5. 监听浏览器返回与前进

popstate 通知应用当前历史项发生切换。处理该事件时不要再次 pushState，否则返回操作会创建新的记录。

```html
<div>
  <p role="status" data-navigation-state>当前位置</p>
</div>
```

```js
window.addEventListener('popstate', () => { void renderCurrentRoute(); });
```

## 6. 取消过时的路由数据请求

快速切换路由时，较早请求可能更晚返回。使用 AbortController 取消旧读取，同时通过独立请求标识过滤已经无法取消的结果。

```html
<div>
  <section data-route-loading aria-live="polite">正在加载页面</section>
</div>
```

```js
let active;
async function renderCurrentRoute() {
  active?.abort();
  const request = active = new AbortController();
  try {
    const view = await matchRoute(new URL(location.href))(request.signal);
    if (active === request && !request.signal.aborted) mountView(view);
  } catch (error) {
    if (active === request && !request.signal.aborted) renderError(error);
  }
}
```

## 7. 用 replaceState 保存当前历史项

筛选条件或滚动位置变化不一定需要新增历史记录。replaceState 可以替换当前记录，写入时保留其他模块已经存储的字段。

```html
<div>
  <label>列表密度
    <select name="density"><option>紧凑</option><option>舒展</option></select>
  </label>
</div>
```

```js
function saveScroll() {
  history.replaceState({ ...history.state, scrollY: window.scrollY }, '', location.href);
}
```

## 8. 恢复滚动位置

选择手动滚动恢复时，应在离开当前历史项前记录位置，并在新页面内容渲染完成后恢复。避免浏览器自动恢复与应用恢复互相竞争。

```html
<div>
  <section data-scroll-content>
    <h2>历史文档</h2>
    <ul data-document-list></ul>
  </section>
</div>
```

```js
history.scrollRestoration = 'manual';
function restorePosition(state) {
  window.scrollTo({ top: Number.isFinite(state?.scrollY) ? state.scrollY : 0, behavior: 'instant' });
}
```

## 9. 导航后的焦点与标题

单页导航不会自动提供完整的新文档焦点体验。更新 document.title，并在合适时机将焦点移动到新页面主标题；返回操作应避免不必要的焦点抢占。

```html
<div>
  <main data-page>
    <h1 tabindex="-1" data-page-title>账户设置</h1>
    <p>管理应用偏好。</p>
  </main>
</div>
```

```js
function announcePage(title) {
  document.title = title;
  document.querySelector('[data-page-title]')?.focus({ preventScroll: true });
}
```

## 10. 服务器刷新与异常处理

浏览器刷新会重新向服务器请求当前路径。客户端路由需要部署层正确处理已知路径，并保留静态资源与 API 的独立响应逻辑。

```html
<div>
  <section role="alert" data-route-error>
    <h2>页面加载失败</h2>
    <button type="button" data-retry-route>重试</button>
  </section>
</div>
```

验收时覆盖直接访问深层路径、连续返回、网络失败和滚动恢复。真实系统也要校验权限，客户端路由并不能代替服务端授权。

## 总结

History API 负责浏览器历史记录，路由匹配、数据加载和交互恢复则是应用职责。把这几个环节分离，可以减少导航过程中常见的历史污染、过期请求和页面状态丢失。

## 热搜标题与本站资料

以下热搜标题仅作当日资讯记录；本站链接为独立资料，并非该新闻的报道。

- 热搜：中国人登月又近一步；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-budget-training-vs-recruitment-reach/>
- 热搜：受贿1.13亿余元！吴胜华被判无期；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-ranking-find-creators/>
- 热搜：车企老板娘女装卖了30亿；本站资料：<https://www.yxtiktok.com/blog/tiktok-creator-no-reply-recruitment-follow-up/>
- 热搜：华为Mate 90 Air售价4999元起；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-recruitment-channels/>
- 热搜：冯远征笑称“考北京人艺其实不难”；本站资料：<https://www.yxtiktok.com/blog/avoid-duplicate-tiktok-creator-collection-and-followup/>
- 热搜：换脸还没消停 “AI融脸”又来了；本站资料：<https://www.yxtiktok.com/blog/tiktok-streamer-recruitment-communities/>
- 热搜：女子登山上班往返约6小时已瘦12斤；本站资料：<https://www.yxtiktok.com/blog/tiktok-creator-segment-recruitment-difficulty-value/>
- 热搜：中国城市 重排全球座次；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-pk-interaction-planning/>
- 热搜：东北外卖15分钟冻成冰；本站资料：<https://www.yxtiktok.com/blog/tiktok-followers-vs-live-engagement-recruiting/>
- 热搜：黄磊真空围裙；本站资料：<https://www.yxtiktok.com/blog/why-tiktok-creator-invite-eligibility-changes/>
- 热搜：外交部回应“黄仁勋重回英伟达”；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-many-leads-low-signup-rate/>
- 热搜：几个姑娘在-25℃的黑龙江清凉上阵；本站资料：<https://www.yxtiktok.com/blog/small-tiktok-live-agency-essential-team-roles/>
- 热搜：泰警方确认女子异地遇害被肢解抛尸；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-cohost-chatting-tips/>
- 热搜：大爷抵脖子不放后小伙倒地 地铁回应；本站资料：<https://www.yxtiktok.com/blog/how-tiktok-agency-prove-identity-to-creator/>
- 热搜：天还没亮 学生就集体站在操场晨读；本站资料：<https://www.yxtiktok.com/blog/creator-not-interested-agency-recruitment-status/>
- 热搜：小米Q3净利润48.84亿 同比减少56.8%；本站资料：<https://www.yxtiktok.com/blog/tiktok-new-vs-experienced-streamer-outreach/>
- 热搜：伪人AI广告 已经成了一种新的城市病；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-streamer-segmented-management/>
- 热搜：中考要不要考历史 教材主编征集意见；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-creator-network-violation-consequences/>
- 热搜：Anthropic上市5天即破发；本站资料：<https://www.yxtiktok.com/blog/evaluate-tiktok-creator-with-limited-history/>
- 热搜：睡眠大赛4人躺赢平分6000元；本站资料：<https://www.yxtiktok.com/blog/tiktok-backstage-creator-management-relationship/>
- 热搜：培养更多高技能人才和大国工匠；本站资料：<https://www.yxtiktok.com/blog/unified-tiktok-creator-recruitment-records/>
- 热搜：新娘提1341次分手新郎回1342次不行；本站资料：<https://www.yxtiktok.com/blog/tiktok-short-video-creator-recruitment/>
- 热搜：特朗普致电黄仁勋：答应我一件事；本站资料：<https://www.yxtiktok.com/blog/select-tiktok-entertainment-live-creator-content-types/>
- 热搜：“民主党向黄仁勋和英伟达宣战”；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-creator-network-vs-mcn-shop/>
- 热搜：上海市委书记陈吉宁被查；本站资料：<https://www.yxtiktok.com/blog/tiktok-creator-weeks-performance-decline/>
- 热搜：飞机与“超级月亮”浪漫同框；本站资料：<https://www.yxtiktok.com/blog/assess-tiktok-creator-live-positioning-from-profile/>
- 热搜：北京天空现“超级月亮”；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-first-recruitment-dm-scripts/>
- 热搜：大众拟裁员10万人 工会威胁罢工；本站资料：<https://www.yxtiktok.com/blog/reassess-long-unfollowed-tiktok-creator-leads/>
- 热搜：加州给548名死刑犯减刑 特朗普震怒；本站资料：<https://www.yxtiktok.com/blog/small-tiktok-live-agency-staff-tool-budget/>
- 热搜：旺仔牛奶更不好卖了；本站资料：<https://www.yxtiktok.com/blog/record-tiktok-creator-invite-check-history/>
- 热搜：一校一主题 消谣育青年；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-streamer-feedback-follow-up/>
- 热搜：文吉儿从小就活跃在各大荧幕；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-recruitment-support-pitch/>
- 热搜：松鼠妈妈六次示范教小松鼠跳树成功；本站资料：<https://www.yxtiktok.com/blog/tiktok-streamer-performance-decline-recovery-plan/>
- 热搜：注水长剧的“狼”来了；本站资料：<https://www.yxtiktok.com/blog/creator-already-in-other-network-new-invite/>
- 热搜：余承东现场提词器曝光；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-entertainment-gaming-shopping-creators/>
- 热搜：19楼落地窗底部装无限位外凸窗；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-replay-review-method/>
- 热搜：央行将开展8000亿元MLF操作；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-recruiter-offboarding-leads-handover/>
- 热搜：香奈儿包日租仅需25元？押金要一万六；本站资料：<https://www.yxtiktok.com/blog/tiktok-dance-live-content-interaction-plan/>
- 热搜：分享你拍的超级月亮；本站资料：<https://www.yxtiktok.com/blog/agency-invite-sent-creator-not-accepted-follow-up/>
- 热搜：Mate X8 华为史上最强大折叠；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-cross-country-invite-region-rules/>
- 热搜：太平洋暖得“失控”了；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-streamer-event-participation/>
- 热搜：国产可口可乐不再标生产日期；本站资料：<https://www.yxtiktok.com/blog/new-tiktok-agency-first-100-creator-leads-plan/>
- 热搜：特朗普连发30多张照片“回味”深圳；本站资料：<https://www.yxtiktok.com/blog/tiktok-streamer-recruitment-channels-by-country/>
- 热搜：华为首款小折叠电脑发布；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-recruitment-funnel-from-lead-to-join/>
- 热搜：动辄千元起步的户外徒步鞋火了；本站资料：<https://www.yxtiktok.com/blog/remote-tiktok-agency-team-shared-creator-data/>
- 热搜：重庆男孩被坠楼男子砸中送医；本站资料：<https://www.yxtiktok.com/blog/tiktok-singing-live-setlist-interaction/>
- 热搜：年产1500万根的“棍子” 卖出10亿元；本站资料：<https://www.yxtiktok.com/blog/single-country-tiktok-agency-creator-pipeline/>
- 热搜：田秉毅任国家羽毛球队主教练；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-handle-creator-live-violation/>
- 热搜：课堂上的“嘉豪、嘉欣” 已不敢举手；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-agency-dm-live-room-recruitment-risks/>
- 热搜：黄奕用AI视频还原过去三十年人生；本站资料：<https://www.yxtiktok.com/blog/tiktok-creator-discovery-ranking-live-profile-workflow/>

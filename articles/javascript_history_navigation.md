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

## 本站 URL 数据样本

以下为本站公开页面地址样本，与上文技术示例及下方热搜无关。

```text
https://www.yxtiktok.com/blog/tiktok-live-fan-club-features/
https://www.yxtiktok.com/blog/solo-vs-team-tiktok-agency-recruitment-workflow/
https://www.yxtiktok.com/blog/tiktok-live-ranking-find-creators/
https://www.yxtiktok.com/blog/why-tiktok-creator-invite-eligibility-changes/
https://www.yxtiktok.com/blog/screen-tiktok-creator-public-live-content-policy-risk/
https://www.yxtiktok.com/blog/tiktok-live-welcome-viewers-scripts/
https://www.yxtiktok.com/blog/tiktok-creator-network-creator-relationship/
https://www.yxtiktok.com/blog/tiktok-live-league-tiers-creator-discovery/
https://www.yxtiktok.com/blog/tiktok-agency-low-live-hours-diagnosis/
https://www.yxtiktok.com/blog/tiktok-creator-discovery-ranking-live-profile-workflow/
https://www.yxtiktok.com/blog/tiktok-backstage-normal-vs-premium-invite/
https://www.yxtiktok.com/blog/tiktok-agency-recruitment-support-pitch/
https://www.yxtiktok.com/blog/small-tiktok-live-agency-essential-team-roles/
https://www.yxtiktok.com/blog/tiktok-agency-multilingual-recruitment-script-library/
https://www.yxtiktok.com/blog/unified-tiktok-creator-recruitment-records/
https://www.yxtiktok.com/blog/tiktok-singing-live-setlist-interaction/
https://www.yxtiktok.com/blog/tiktok-creator-network-multi-account-invite-risk/
https://www.yxtiktok.com/blog/how-to-check-tiktok-creator-agency-invite-eligibility/
https://www.yxtiktok.com/blog/growth-vs-established-tiktok-creators-recruitment-talk/
https://www.yxtiktok.com/blog/creator-not-interested-agency-recruitment-status/
https://www.yxtiktok.com/blog/can-individual-apply-tiktok-live-creator-network/
https://www.yxtiktok.com/blog/tiktok-live-events-vs-campaigns/
https://www.yxtiktok.com/blog/select-tiktok-entertainment-live-creator-content-types/
https://www.yxtiktok.com/blog/tiktok-agency-recruitment-promise-contract-check/
https://www.yxtiktok.com/blog/tiktok-agency-recruitment-funnel-from-lead-to-join/
https://www.yxtiktok.com/blog/why-high-earning-tiktok-creators-may-not-fit-agency/
https://www.yxtiktok.com/blog/tiktok-agency-streamer-event-participation/
https://www.yxtiktok.com/blog/reduce-duplicate-work-multiple-tiktok-agency-accounts/
https://www.yxtiktok.com/blog/why-tiktok-agency-needs-employee-audit-logs/
https://www.yxtiktok.com/blog/why-invitable-tiktok-creator-may-not-join/
https://www.yxtiktok.com/blog/tiktok-country-ranking-find-live-creators/
https://www.yxtiktok.com/blog/personalize-tiktok-creator-outreach-by-live-content/
https://www.yxtiktok.com/blog/tiktok-agency-recruiter-performance-kpis/
https://www.yxtiktok.com/blog/tiktok-streamer-rejects-agency-advice/
https://www.yxtiktok.com/blog/early-signs-streamer-unhappy-with-agency/
https://www.yxtiktok.com/blog/why-agencies-should-not-rely-only-on-tiktok-top-rankings/
https://www.yxtiktok.com/blog/tiktok-agency-recruitment-channels/
https://www.yxtiktok.com/blog/tiktok-live-pk-interaction-planning/
https://www.yxtiktok.com/blog/clean-invalid-stale-tiktok-creator-lead-records/
https://www.yxtiktok.com/blog/prevent-tiktok-agency-recruiter-duplicate-contact/
https://www.yxtiktok.com/blog/tiktok-agency-daily-recruitment-plan/
https://www.yxtiktok.com/blog/tiktok-host-recruitment-cost-per-signed-creator/
https://www.yxtiktok.com/blog/tiktok-agency-streamer-feedback-follow-up/
https://www.yxtiktok.com/blog/tiktok-streamer-performance-decline-recovery-plan/
https://www.yxtiktok.com/blog/does-higher-tiktok-live-league-mean-better-recruit/
https://www.yxtiktok.com/blog/tiktok-creator-pre-signing-info-minimization/
https://www.yxtiktok.com/blog/avoid-duplicate-tiktok-creator-collection-and-followup/
https://www.yxtiktok.com/blog/tiktok-live-awkward-silence-topic-switching/
https://www.yxtiktok.com/blog/tiktok-agency-streamer-referral-recruitment/
https://www.yxtiktok.com/blog/tiktok-streamer-not-responding-agency/
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

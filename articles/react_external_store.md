# React useSyncExternalStore 实战：订阅外部数据源与稳定快照

React 应用有时需要读取已有的事件总线、浏览器在线状态或非 React 状态容器。直接在 render 中读取可变对象，可能在并发渲染时看到不一致的值。useSyncExternalStore 提供标准订阅和快照接口。本文以连接状态与消息计数为例说明正确的集成方式。

## 1. 先区分内部状态与外部状态

如果数据完全属于单个 React 组件，useState 通常就足够了。只有状态由 React 之外的系统持有，且组件需要持续读取它时，才考虑这个 Hook。

```jsx
<div>
  <section><h2>连接中心</h2><p>显示通信状态与未读消息数</p></section>
</div>
```

## 2. 实现稳定的外部快照

getSnapshot 返回的是当前数据视图。没有数据变化时必须返回相同的对象引用，而不是每次调用都创建一个新对象。

```jsx
<div>
  <output aria-label="未读消息数">{snapshot.unread}</output>
</div>
```

```ts
type Snapshot = Readonly<{ online: boolean; unread: number }>;
let snapshot: Snapshot = Object.freeze({ online: true, unread: 0 });
const listeners = new Set<() => void>();
export const getSnapshot = () => snapshot;
export function update(next: Snapshot) {
  if (next.online === snapshot.online && next.unread === snapshot.unread) return;
  snapshot = Object.freeze({ ...next });
  listeners.forEach(listener => listener());
}
```

## 3. 订阅函数必须提供清理

subscribe 接收通知回调并返回取消订阅函数。定义在组件外部可保持函数身份稳定，减少不必要的重新订阅。

```jsx
<div>
  <p role="status">{snapshot.online ? '在线' : '离线'}</p>
</div>
```

```ts
export function subscribe(listener: () => void) {
  listeners.add(listener);
  return () => { listeners.delete(listener); };
}
```

## 4. 在 React 组件中读取快照

Hook 负责协调订阅与快照读取，组件不需要再维护一份通过 effect 同步的本地副本。

```jsx
<div>
  <section aria-label="消息摘要">
    <span>未读：{snapshot.unread}</span>
    <span>{snapshot.online ? '可用' : '断开'}</span>
  </section>
</div>
```

```tsx
import { useSyncExternalStore } from 'react';
const snapshot = useSyncExternalStore(subscribe, getSnapshot, getServerSnapshot);
```

## 5. 适配浏览器在线状态

navigator.onLine 表示浏览器判断的网络连接状态，不代表业务服务必然可达。应单独记录业务连接探测结果。

```jsx
<div>
  <p role="status">{online ? '设备有网络连接' : '设备离线'}</p>
</div>
```

```ts
function subscribeOnline(notify: () => void) {
  window.addEventListener('online', notify);
  window.addEventListener('offline', notify);
  return () => {
    window.removeEventListener('online', notify);
    window.removeEventListener('offline', notify);
  };
}
const readOnline = () => navigator.onLine;
```

## 6. 服务器渲染时的初始快照

getServerSnapshot 在服务端及客户端水合阶段提供初始数据。两端必须一致。若每个用户有不同的初始数据，需要通过请求作用域和序列化机制传递，不能共享全局私人状态。

```jsx
<div>
  <aside aria-label="初始连接信息"><span>连接状态将在客户端更新</span></aside>
</div>
```

```ts
const initialSnapshot = Object.freeze({ online: true, unread: 0 });
const getServerSnapshot = () => initialSnapshot;
```

## 7. 避免无意义的整体更新

当数据对象拥有大量字段时，一个新快照可能触发所有相关订阅者重新读取。拆分业务独立的状态容器，或使用具备选择器能力的成熟实现。

```jsx
<div>
  <section><h2>未读消息</h2><strong>{unread}</strong></section>
</div>
```

```tsx
const getUnread = () => getSnapshot().unread;
const getServerUnread = () => getServerSnapshot().unread;
const unread = useSyncExternalStore(subscribe, getUnread, getServerUnread);
```

## 8. 确保外部事件有明确的状态所有者

消息到达后由外部 store 更新，不要在每个订阅组件中分别增加计数。否则多组件挂载可能让同一事件被重复计算。

```jsx
<div>
  <button type="button" onClick={markAllRead}>全部标记已读</button>
</div>
```

```ts
function markAllRead() {
  update({ ...getSnapshot(), unread: 0 });
}
```

## 9. 外部事件与业务连接分离

WebSocket 或事件流适配器应独立处理重连、心跳和错误恢复，store 仅接收已规范化的数据。订阅组件并不负责创建每个连接实例。

```jsx
<div>
  <dl><dt>连接状态</dt><dd>{connectionState}</dd><dt>消息数</dt><dd>{snapshot.unread}</dd></dl>
</div>
```

外部事件可能乱序或重复，业务数据仍需稳定 ID、版本或服务端确认。Hook 本身不会自动处理这些协议问题。

## 10. 验证快照和清理行为

测试没有变化时快照引用相同，有变化时引用不同；订阅取消后回调不再执行。React 组件测试还应覆盖挂载与卸载。

```jsx
<div>
  <p aria-live="polite">{snapshot.unread === 0 ? '没有未读消息' : '存在新消息'}</p>
</div>
```

```ts
const before = getSnapshot();
update({ ...before });
expect(getSnapshot()).toBe(before);
const callback = vi.fn();
const stop = subscribe(callback);
stop();
update({ ...before, unread: before.unread + 1 });
expect(callback).not.toHaveBeenCalled();
```

## 总结

useSyncExternalStore 解决的是 React 与外部可变数据源之间的读取一致性。正确的实现依赖稳定快照、可清理的订阅和合理的状态边界，而不是简单地将每次外部变化复制到组件内部。

## 热搜标题与本站资料

以下热搜标题仅作当日资讯记录；本站链接为独立资料，并非该新闻的报道。

- 热搜：车企老板娘女装卖了30亿；本站资料：<https://www.yxtiktok.com/blog/early-signs-streamer-unhappy-with-agency/>
- 热搜：东北外卖15分钟冻成冰；本站资料：<https://www.yxtiktok.com/blog/tiktok-creator-no-reply-recruitment-follow-up/>
- 热搜：香奈儿包日租仅需25元？押金要一万六；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-invite-not-found-troubleshooting/>
- 热搜：19楼落地窗底部装无限位外凸窗；本站资料：<https://www.yxtiktok.com/blog/unified-tiktok-creator-recruitment-records/>
- 热搜：小米Q3净利润48.84亿 同比减少56.8%；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-daily-recruitment-plan/>
- 热搜：大众拟裁员10万人 工会威胁罢工；本站资料：<https://www.yxtiktok.com/blog/tiktok-entertainment-live-opening-retention/>
- 热搜：重庆男孩被坠楼男子砸中送医；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-streamer-onboarding-handoff/>
- 热搜：田秉毅任国家羽毛球队主教练；本站资料：<https://www.yxtiktok.com/blog/why-tiktok-agency-deduplicate-creator-uid/>
- 热搜：课堂上的“嘉豪、嘉欣” 已不敢举手；本站资料：<https://www.yxtiktok.com/blog/daily-tiktok-agency-recruitment-report-kpis/>
- 热搜：“民主党向黄仁勋和英伟达宣战”；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-recruitment-funnel-from-lead-to-join/>
- 热搜：北京天空现“超级月亮”；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-first-contact-message-to-creator/>
- 热搜：中考要不要考历史 教材主编征集意见；本站资料：<https://www.yxtiktok.com/blog/multi-country-tiktok-agency-team-language-allocation/>
- 热搜：培养更多高技能人才和大国工匠；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-awkward-silence-topic-switching/>
- 热搜：特朗普连发30多张照片“回味”深圳；本站资料：<https://www.yxtiktok.com/blog/manual-vs-software-tiktok-streamer-recruitment/>
- 热搜：中国城市 重排全球座次；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-follow-cta-without-pressure/>
- 热搜：特朗普致电黄仁勋：答应我一件事；本站资料：<https://www.yxtiktok.com/blog/avoid-spam-repeated-dm-tiktok-agency-recruitment/>
- 热搜：泰警方确认女子异地遇害被肢解抛尸；本站资料：<https://www.yxtiktok.com/blog/assign-tiktok-creator-leads-country-language-stage/>
- 热搜：注水长剧的“狼”来了；本站资料：<https://www.yxtiktok.com/blog/personalize-tiktok-creator-outreach-by-live-content/>
- 热搜：加州给548名死刑犯减刑 特朗普震怒；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-creator-live-schedule/>
- 热搜：央行将开展8000亿元MLF操作；本站资料：<https://www.yxtiktok.com/blog/tiktok-new-streamer-pre-live-checklist/>
- 热搜：一校一主题 消谣育青年；本站资料：<https://www.yxtiktok.com/blog/tiktok-creator-segment-recruitment-difficulty-value/>
- 热搜：冯远征笑称“考北京人艺其实不难”；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-league-tiers-creator-discovery/>
- 热搜：动辄千元起步的户外徒步鞋火了；本站资料：<https://www.yxtiktok.com/blog/find-tiktok-creators-for-small-agency-development/>
- 热搜：睡眠大赛4人躺赢平分6000元；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-host-review-meeting-metrics/>
- 热搜：换脸还没消停 “AI融脸”又来了；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-recruitment-messages-lose-trust/>
- 热搜：几个姑娘在-25℃的黑龙江清凉上阵；本站资料：<https://www.yxtiktok.com/blog/benefits-of-joining-tiktok-live-agency/>
- 热搜：余承东现场提词器曝光；本站资料：<https://www.yxtiktok.com/blog/evaluate-tiktok-live-leaderboard-for-recruitment-list/>
- 热搜：Mate X8 华为史上最强大折叠；本站资料：<https://www.yxtiktok.com/blog/why-tiktok-creator-invite-eligibility-changes/>
- 热搜：天还没亮 学生就集体站在操场晨读；本站资料：<https://www.yxtiktok.com/blog/tiktok-streamer-performance-decline-recovery-plan/>
- 热搜：华为首款小折叠电脑发布；本站资料：<https://www.yxtiktok.com/blog/tiktok-diamonds-vs-creator-rewards/>
- 热搜：女子登山上班往返约6小时已瘦12斤；本站资料：<https://www.yxtiktok.com/blog/tiktok-creator-discovery-to-joining-sop/>
- 热搜：旺仔牛奶更不好卖了；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-streamer-referral-recruitment/>
- 热搜：黄奕用AI视频还原过去三十年人生；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-match-pk-rules/>
- 热搜：Anthropic上市5天即破发；本站资料：<https://www.yxtiktok.com/blog/why-high-earning-tiktok-creators-may-not-fit-agency/>
- 热搜：松鼠妈妈六次示范教小松鼠跳树成功；本站资料：<https://www.yxtiktok.com/blog/compare-tiktok-live-rankings-across-countries/>
- 热搜：文吉儿从小就活跃在各大荧幕；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-fan-club-features/>
- 热搜：国产可口可乐不再标生产日期；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-streamer-segmented-management/>
- 热搜：黄磊真空围裙；本站资料：<https://www.yxtiktok.com/blog/tiktok-creator-network-multi-account-invite-risk/>
- 热搜：年产1500万根的“棍子” 卖出10亿元；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-budget-training-vs-recruitment-reach/>
- 热搜：上海市委书记陈吉宁被查；本站资料：<https://www.yxtiktok.com/blog/tiktok-dance-live-content-interaction-plan/>
- 热搜：华为Mate 90 Air售价4999元起；本站资料：<https://www.yxtiktok.com/blog/tiktok-short-video-creator-recruitment/>
- 热搜：太平洋暖得“失控”了；本站资料：<https://www.yxtiktok.com/blog/three-tier-tiktok-creator-lead-priority-system/>
- 热搜：外交部回应“黄仁勋重回英伟达”；本站资料：<https://www.yxtiktok.com/blog/tiktok-creator-weeks-performance-decline/>
- 热搜：中国人登月又近一步；本站资料：<https://www.yxtiktok.com/blog/why-tiktok-agency-needs-employee-audit-logs/>
- 热搜：飞机与“超级月亮”浪漫同框；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-events-vs-campaigns/>
- 热搜：伪人AI广告 已经成了一种新的城市病；本站资料：<https://www.yxtiktok.com/blog/screen-tiktok-creator-public-live-content-policy-risk/>
- 热搜：大爷抵脖子不放后小伙倒地 地铁回应；本站资料：<https://www.yxtiktok.com/blog/which-live-metrics-tiktok-agency-should-evaluate/>
- 热搜：新娘提1341次分手新郎回1342次不行；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-creator-incentive-communication/>
- 热搜：分享你拍的超级月亮；本站资料：<https://www.yxtiktok.com/blog/tiktok-creator-network-cross-region-compliance/>
- 热搜：受贿1.13亿余元！吴胜华被判无期；本站资料：<https://www.yxtiktok.com/blog/tiktok-streamer-recruitment-communities/>

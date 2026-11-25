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

## 本站 URL 数据样本

以下为本站公开页面地址样本，与上文技术示例及下方热搜无关。

```text
https://www.yxtiktok.com/blog/how-tiktok-agency-prove-identity-to-creator/
https://www.yxtiktok.com/blog/tiktok-chat-streamer-improvisation-training/
https://www.yxtiktok.com/blog/tiktok-live-diamonds-stability-vs-one-day-peak/
https://www.yxtiktok.com/blog/screen-tiktok-creator-public-live-content-policy-risk/
https://www.yxtiktok.com/blog/tiktok-agency-recruitment-messages-lose-trust/
https://www.yxtiktok.com/blog/tiktok-recruitment-channel-conversion-analysis/
https://www.yxtiktok.com/blog/tiktok-singing-live-setlist-interaction/
https://www.yxtiktok.com/blog/tiktok-host-recruitment-cost-per-signed-creator/
https://www.yxtiktok.com/blog/tiktok-creator-network-cross-region-compliance/
https://www.yxtiktok.com/blog/tiktok-live-match-pk-rules/
https://www.yxtiktok.com/blog/which-live-metrics-tiktok-agency-should-evaluate/
https://www.yxtiktok.com/blog/tiktok-agency-many-leads-low-signup-rate/
https://www.yxtiktok.com/blog/tiktok-agency-creator-live-schedule/
https://www.yxtiktok.com/blog/why-high-earning-tiktok-creators-may-not-fit-agency/
https://www.yxtiktok.com/blog/tiktok-live-agency-revenue-sources/
https://www.yxtiktok.com/blog/how-to-check-tiktok-creator-agency-invite-eligibility/
https://www.yxtiktok.com/blog/tiktok-agency-streamer-feedback-follow-up/
https://www.yxtiktok.com/blog/why-tiktok-agency-needs-employee-audit-logs/
https://www.yxtiktok.com/blog/tiktok-agency-recruiter-offboarding-leads-handover/
https://www.yxtiktok.com/blog/high-gift-high-growth-stable-tiktok-creator-types/
https://www.yxtiktok.com/blog/evaluate-tiktok-live-leaderboard-for-recruitment-list/
https://www.yxtiktok.com/blog/tiktok-new-creator-coaching-payback-period/
https://www.yxtiktok.com/blog/three-tier-tiktok-creator-lead-priority-system/
https://www.yxtiktok.com/blog/avoid-spam-repeated-dm-tiktok-agency-recruitment/
https://www.yxtiktok.com/blog/tiktok-part-time-vs-full-time-streamer-management/
https://www.yxtiktok.com/blog/tiktok-creator-network-creator-relationship/
https://www.yxtiktok.com/blog/tiktok-dm-requests-privacy-creator-contact/
https://www.yxtiktok.com/blog/tiktok-streamer-not-responding-agency/
https://www.yxtiktok.com/blog/why-agencies-should-not-rely-only-on-tiktok-top-rankings/
https://www.yxtiktok.com/blog/top-vs-mid-tier-tiktok-live-creators-agency/
https://www.yxtiktok.com/blog/cost-structure-recruit-established-vs-new-live-creators/
https://www.yxtiktok.com/blog/unified-tiktok-creator-recruitment-records/
https://www.yxtiktok.com/blog/creator-already-in-other-network-new-invite/
https://www.yxtiktok.com/blog/tiktok-agency-streamer-training-feedback-system/
https://www.yxtiktok.com/blog/small-tiktok-live-agency-staff-tool-budget/
https://www.yxtiktok.com/blog/tiktok-agency-streamer-maintenance-schedule/
https://www.yxtiktok.com/blog/assess-tiktok-creator-live-positioning-from-profile/
https://www.yxtiktok.com/blog/small-tiktok-live-agency-priority-creator-types/
https://www.yxtiktok.com/blog/tiktok-pk-live-recruitment/
https://www.yxtiktok.com/blog/new-tiktok-agency-first-100-creator-leads-plan/
https://www.yxtiktok.com/blog/build-tiktok-agency-creator-selection-criteria/
https://www.yxtiktok.com/blog/single-country-tiktok-agency-creator-pipeline/
https://www.yxtiktok.com/blog/tiktok-agency-recruitment-channels/
https://www.yxtiktok.com/blog/tiktok-coins-gifts-diamonds-difference/
https://www.yxtiktok.com/blog/tiktok-live-agency-dm-live-room-recruitment-risks/
https://www.yxtiktok.com/blog/prevent-tiktok-agency-recruiter-duplicate-contact/
https://www.yxtiktok.com/blog/tiktok-agency-invite-not-found-troubleshooting/
https://www.yxtiktok.com/blog/tiktok-dance-live-content-interaction-plan/
https://www.yxtiktok.com/tutorials/tiktok-ios-non-jailbreak-dm-tutorial/
https://www.yxtiktok.com/blog/tiktok-live-host-review-meeting-metrics/
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

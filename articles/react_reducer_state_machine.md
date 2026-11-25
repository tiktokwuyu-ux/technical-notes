# React useReducer 实战：复杂组件状态转换与可测试的业务流程

当一个上传面板同时维护文件、进度、错误、取消和重试时，多个 useState 很容易产生互相矛盾的值。useReducer 可以把状态转换集中为纯函数，使事件与界面状态之间的关系更清楚。本文以导入任务为例，采用 TypeScript 判别联合描述业务状态。

## 1. 确定状态而不是布尔变量数量

不要分别维护 isLoading、isSuccess 和 isError 等可能同时为真的变量。导入任务可以处于 idle、running、success 或 error 中的一个状态。

```jsx
<div>
  <section aria-label="导入任务">
    <h2>批量导入</h2>
    <p>选择数据文件后开始处理</p>
  </section>
</div>
```

```ts
type State =
  | { status: 'idle' }
  | { status: 'running'; id: string; progress: number }
  | { status: 'success'; count: number }
  | { status: 'error'; message: string };
```

## 2. 为状态转换建立事件协议

事件应表达发生了什么，而非直接描述需要设置哪几个字段。异步进度事件携带任务 ID，使 reducer 能判断它是否仍属于当前操作。

```jsx
<div>
  <button type="button" onClick={startImport}>开始导入</button>
</div>
```

```ts
type Action =
  | { type: 'start'; id: string }
  | { type: 'progress'; id: string; value: number }
  | { type: 'success'; id: string; count: number }
  | { type: 'error'; id: string; message: string }
  | { type: 'reset' };
```

## 3. 编写纯 reducer

reducer 不发送请求、不生成随机值，也不操作 DOM。对旧任务的响应直接返回原状态，可以防止过期事件破坏新流程。

```jsx
<div>
  <output aria-live="polite">{state.status}</output>
</div>
```

```ts
function reducer(state: State, action: Action): State {
  if (action.type === 'reset') return { status: 'idle' };
  if (action.type === 'start') return { status: 'running', id: action.id, progress: 0 };
  if (state.status !== 'running' || state.id !== action.id) return state;
  switch (action.type) {
    case 'progress': return { ...state, progress: Math.max(state.progress, Math.min(100, action.value)) };
    case 'success': return { status: 'success', count: action.count };
    case 'error': return { status: 'error', message: action.message };
  }
}
```

## 4. 把状态接入组件

使用 useReducer 初始化状态，视图从 state.status 派生。事件的处理时机和异步请求依旧由组件或业务服务决定。

```jsx
<div>
  <button disabled={state.status === 'running'} onClick={startImport}>
    {state.status === 'running' ? '处理中' : '开始'}
  </button>
</div>
```

```tsx
const [state, dispatch] = useReducer(reducer, { status: 'idle' } satisfies State);
```

## 5. 在事件处理器中启动异步操作

请求发生于用户明确点击开始时，通常不必依赖 effect 监听状态再启动。业务服务接收进度回调和取消信号。

```jsx
<div>
  <label>选择文件 <input type="file" onChange={selectFile} /></label>
  <button onClick={startImport}>导入文件</button>
</div>
```

```tsx
async function runImport(file: File, signal: AbortSignal) {
  const id = crypto.randomUUID();
  dispatch({ type: 'start', id });
  try {
    const count = await importFile(file, { signal, onProgress: value => dispatch({ type: 'progress', id, value }) });
    dispatch({ type: 'success', id, count });
  } catch (error) {
    if (!signal.aborted) dispatch({ type: 'error', id, message: '导入失败' });
  }
}
```

## 6. 让进度只在运行状态可见

判别联合能够帮助 TypeScript 缩小类型范围，避免在空闲状态读取不存在的进度属性。

```jsx
<div>
  {state.status === 'running' && (
    <progress value={state.progress} max={100} aria-label="导入进度" />
  )}
</div>
```

进度的单位要与服务端或本地任务统一。网络上传完成不一定表示服务端已经解析并写入数据。

## 7. 取消任务并区分本地与远程状态

取消读取后可重置本地视图，但远程任务如果已经开始，可能需要独立取消接口。不要将 abort 当成业务回滚。

```jsx
<div>
  {state.status === 'running' && <button onClick={cancelImport}>取消</button>}
</div>
```

```tsx
function cancelImport() {
  controllerRef.current?.abort();
  dispatch({ type: 'reset' });
}
```

## 8. 错误重试应创建新任务身份

失败状态保留错误信息；重试使用上次选择的文件并生成新的任务标识。是否可以重复提交由接口幂等契约决定。

```jsx
<div>
  {state.status === 'error' && (
    <section role="alert"><p>{state.message}</p><button onClick={retry}>重试</button></section>
  )}
</div>
```

重试不能复用可能已经过期的 AbortSignal。对于写入请求，服务端应有独立的任务身份或幂等键。

## 9. 独立测试 reducer

纯函数适合做表驱动测试。重点验证非法转换、不同任务的响应和边界进度，而不仅是正常成功流程。

```jsx
<div>
  <dl><dt>任务状态</dt><dd>{state.status}</dd></dl>
</div>
```

```ts
const running: State = { status: 'running', id: 'current', progress: 40 };
expect(reducer(running, { type: 'success', id: 'old', count: 10 })).toBe(running);
expect(reducer(running, { type: 'progress', id: 'current', value: 80 })).toEqual({ ...running, progress: 80 });
```

## 10. 明确组件与服务的边界

一个 reducer 适合一个具有内聚性的业务流程，不必把文件读取、列表查询和全局权限都揉进同一状态机。组件卸载时还要取消请求并释放资源。

```jsx
<div>
  {state.status === 'success' && <p role="status">已导入 {state.count} 条记录</p>}
</div>
```

测试界面时验证状态变化、错误信息和取消后的重新开始。React 开发模式可能额外调用纯 reducer 来帮助发现副作用，所以不要在 reducer 里执行一次性动作。

## 总结

useReducer 的核心价值是显式表达状态转换规则。保持 reducer 纯净，将网络操作放到事件或业务服务中，再使用任务身份处理竞态，复杂交互就能具备清晰的数据流与可测试性。

## 本站 URL 数据样本

以下为本站公开页面地址样本，与上文技术示例及下方热搜无关。

```text
https://www.yxtiktok.com/blog/tiktok-singing-live-setlist-interaction/
https://www.yxtiktok.com/blog/tiktok-agency-recruitment-channels/
https://www.yxtiktok.com/blog/tiktok-agency-multilingual-recruitment-script-library/
https://www.yxtiktok.com/blog/tiktok-creator-pre-signing-info-minimization/
https://www.yxtiktok.com/blog/tiktok-agency-one-on-one-streamer-support/
https://www.yxtiktok.com/blog/tiktok-live-league-tiers-creator-discovery/
https://www.yxtiktok.com/blog/tiktok-diamonds-vs-creator-rewards/
https://www.yxtiktok.com/blog/why-agencies-should-not-rely-only-on-tiktok-top-rankings/
https://www.yxtiktok.com/blog/tiktok-agency-recruitment-funnel-from-lead-to-join/
https://www.yxtiktok.com/blog/daily-tiktok-agency-recruitment-report-kpis/
https://www.yxtiktok.com/blog/tiktok-creator-network-creator-relationship/
https://www.yxtiktok.com/blog/why-tiktok-live-ranking-platforms-disagree/
https://www.yxtiktok.com/blog/tiktok-live-creator-network-vs-mcn-shop/
https://www.yxtiktok.com/blog/multi-country-tiktok-agency-team-language-allocation/
https://www.yxtiktok.com/blog/tiktok-agency-streamer-training-feedback-system/
https://www.yxtiktok.com/blog/tiktok-dance-live-content-interaction-plan/
https://www.yxtiktok.com/blog/tiktok-live-creator-network-violation-consequences/
https://www.yxtiktok.com/blog/how-to-check-tiktok-creator-agency-invite-eligibility/
https://www.yxtiktok.com/tutorials/tiktok-live-ranklist-tutorial/
https://www.yxtiktok.com/blog/select-tiktok-entertainment-live-creator-content-types/
https://www.yxtiktok.com/blog/tiktok-creator-discovery-ranking-live-profile-workflow/
https://www.yxtiktok.com/blog/tiktok-live-agency-revenue-sources/
https://www.yxtiktok.com/blog/tiktok-creator-discovery-to-joining-sop/
https://www.yxtiktok.com/blog/tiktok-creator-no-reply-recruitment-follow-up/
https://www.yxtiktok.com/blog/tiktok-creator-network-cross-region-compliance/
https://www.yxtiktok.com/blog/benefits-of-joining-tiktok-live-agency/
https://www.yxtiktok.com/blog/tiktok-agency-recruitment-communication-tips/
https://www.yxtiktok.com/blog/tiktok-live-welcome-viewers-scripts/
https://www.yxtiktok.com/blog/tiktok-agency-recruitment-messages-lose-trust/
https://www.yxtiktok.com/blog/tiktok-part-time-vs-full-time-streamer-management/
https://www.yxtiktok.com/blog/what-is-tiktok-live-backstage-agency-functions/
https://www.yxtiktok.com/blog/why-tiktok-agency-needs-employee-audit-logs/
https://www.yxtiktok.com/blog/remote-tiktok-agency-team-shared-creator-data/
https://www.yxtiktok.com/blog/early-signs-streamer-unhappy-with-agency/
https://www.yxtiktok.com/blog/small-tiktok-live-agency-essential-team-roles/
https://www.yxtiktok.com/blog/tiktok-live-agency-differentiated-creator-offer/
https://www.yxtiktok.com/blog/avoid-duplicate-tiktok-creator-collection-and-followup/
https://www.yxtiktok.com/blog/tiktok-agency-recruitment-message-cooperation-terms/
https://www.yxtiktok.com/blog/tiktok-agency-recruiter-performance-kpis/
https://www.yxtiktok.com/blog/tiktok-live-creator-agency-contract-checklist/
https://www.yxtiktok.com/blog/tiktok-live-cohost-chatting-tips/
https://www.yxtiktok.com/blog/tiktok-live-awkward-silence-topic-switching/
https://www.yxtiktok.com/blog/tiktok-game-streamer-audience-interaction/
https://www.yxtiktok.com/blog/tiktok-new-streamer-pre-live-checklist/
https://www.yxtiktok.com/blog/tiktok-agency-many-leads-low-signup-rate/
https://www.yxtiktok.com/blog/tiktok-new-vs-experienced-streamer-outreach/
https://www.yxtiktok.com/blog/can-individual-apply-tiktok-live-creator-network/
https://www.yxtiktok.com/blog/tiktok-agency-recruiter-offboarding-leads-handover/
https://www.yxtiktok.com/blog/tiktok-agency-first-call-creator-needs-questions/
https://www.yxtiktok.com/blog/tiktok-chat-streamer-improvisation-training/
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

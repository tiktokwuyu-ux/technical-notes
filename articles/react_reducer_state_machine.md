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

## 热搜标题与本站资料

以下热搜标题仅作当日资讯记录；本站链接为独立资料，并非该新闻的报道。

- 热搜：几个姑娘在-25℃的黑龙江清凉上阵；本站资料：<https://www.yxtiktok.com/blog/which-live-metrics-tiktok-agency-should-evaluate/>
- 热搜：天还没亮 学生就集体站在操场晨读；本站资料：<https://www.yxtiktok.com/blog/tiktok-backstage-normal-vs-premium-invite/>
- 热搜：中国人登月又近一步；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-first-contact-message-to-creator/>
- 热搜：车企老板娘女装卖了30亿；本站资料：<https://www.yxtiktok.com/blog/tiktok-creator-no-reply-recruitment-follow-up/>
- 热搜：香奈儿包日租仅需25元？押金要一万六；本站资料：<https://www.yxtiktok.com/blog/assess-tiktok-creator-live-positioning-from-profile/>
- 热搜：一校一主题 消谣育青年；本站资料：<https://www.yxtiktok.com/blog/tiktok-creator-network-multi-account-invite-risk/>
- 热搜：黄奕用AI视频还原过去三十年人生；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-creator-incentive-communication/>
- 热搜：华为首款小折叠电脑发布；本站资料：<https://www.yxtiktok.com/blog/tiktok-country-ranking-find-live-creators/>
- 热搜：“民主党向黄仁勋和英伟达宣战”；本站资料：<https://www.yxtiktok.com/blog/why-agencies-should-not-rely-only-on-tiktok-top-rankings/>
- 热搜：大爷抵脖子不放后小伙倒地 地铁回应；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-diamonds-stability-vs-one-day-peak/>
- 热搜：小米Q3净利润48.84亿 同比减少56.8%；本站资料：<https://www.yxtiktok.com/blog/why-tiktok-creator-invite-eligibility-changes/>
- 热搜：注水长剧的“狼”来了；本站资料：<https://www.yxtiktok.com/blog/three-tier-tiktok-creator-lead-priority-system/>
- 热搜：国产可口可乐不再标生产日期；本站资料：<https://www.yxtiktok.com/blog/evaluate-tiktok-creator-with-limited-history/>
- 热搜：北京天空现“超级月亮”；本站资料：<https://www.yxtiktok.com/blog/filter-overseas-tiktok-creators-by-language/>
- 热搜：特朗普连发30多张照片“回味”深圳；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-creator-network-violation-consequences/>
- 热搜：年产1500万根的“棍子” 卖出10亿元；本站资料：<https://www.yxtiktok.com/blog/tiktok-chat-streamer-improvisation-training/>
- 热搜：睡眠大赛4人躺赢平分6000元；本站资料：<https://www.yxtiktok.com/blog/find-largest-dropoff-tiktok-agency-recruitment-funnel/>
- 热搜：中国城市 重排全球座次；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-replay-review-method/>
- 热搜：黄磊真空围裙；本站资料：<https://www.yxtiktok.com/blog/tiktok-entertainment-live-opening-retention/>
- 热搜：Anthropic上市5天即破发；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-feed-creator-recruitment/>
- 热搜：换脸还没消停 “AI融脸”又来了；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-ranking-find-creators/>
- 热搜：文吉儿从小就活跃在各大荧幕；本站资料：<https://www.yxtiktok.com/blog/early-signs-streamer-unhappy-with-agency/>
- 热搜：中考要不要考历史 教材主编征集意见；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-recruitment-support-pitch/>
- 热搜：东北外卖15分钟冻成冰；本站资料：<https://www.yxtiktok.com/blog/tiktok-creator-network-cross-region-compliance/>
- 热搜：女子登山上班往返约6小时已瘦12斤；本站资料：<https://www.yxtiktok.com/blog/single-country-tiktok-agency-creator-pipeline/>
- 热搜：特朗普致电黄仁勋：答应我一件事；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-recruitment-funnel-from-lead-to-join/>
- 热搜：外交部回应“黄仁勋重回英伟达”；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-turnover-revenue-gross-net-profit/>
- 热搜：伪人AI广告 已经成了一种新的城市病；本站资料：<https://www.yxtiktok.com/blog/multi-country-tiktok-agency-team-language-allocation/>
- 热搜：央行将开展8000亿元MLF操作；本站资料：<https://www.yxtiktok.com/blog/avoid-duplicate-tiktok-creator-collection-and-followup/>
- 热搜：分享你拍的超级月亮；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-host-review-meeting-metrics/>
- 热搜：泰警方确认女子异地遇害被肢解抛尸；本站资料：<https://www.yxtiktok.com/blog/why-recheck-creator-eligibility-before-contact/>
- 热搜：飞机与“超级月亮”浪漫同框；本站资料：<https://www.yxtiktok.com/blog/high-gift-high-growth-stable-tiktok-creator-types/>
- 热搜：旺仔牛奶更不好卖了；本站资料：<https://www.yxtiktok.com/blog/small-tiktok-live-agency-staff-tool-budget/>
- 热搜：培养更多高技能人才和大国工匠；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-streamer-referral-recruitment/>
- 热搜：新娘提1341次分手新郎回1342次不行；本站资料：<https://www.yxtiktok.com/blog/how-to-check-tiktok-creator-agency-invite-eligibility/>
- 热搜：课堂上的“嘉豪、嘉欣” 已不敢举手；本站资料：<https://www.yxtiktok.com/blog/tiktok-new-creator-coaching-payback-period/>
- 热搜：上海市委书记陈吉宁被查；本站资料：<https://www.yxtiktok.com/blog/why-tiktok-live-ranking-differs-from-real-performance/>
- 热搜：受贿1.13亿余元！吴胜华被判无期；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-creator-network-vs-mcn-shop/>
- 热搜：加州给548名死刑犯减刑 特朗普震怒；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-events-vs-campaigns/>
- 热搜：余承东现场提词器曝光；本站资料：<https://www.yxtiktok.com/blog/why-tiktok-live-ranking-platforms-disagree/>
- 热搜：大众拟裁员10万人 工会威胁罢工；本站资料：<https://www.yxtiktok.com/blog/why-tiktok-agency-needs-employee-audit-logs/>
- 热搜：重庆男孩被坠楼男子砸中送医；本站资料：<https://www.yxtiktok.com/blog/can-individual-apply-tiktok-live-creator-network/>
- 热搜：田秉毅任国家羽毛球队主教练；本站资料：<https://www.yxtiktok.com/blog/assess-third-party-tiktok-live-ranking-coverage/>
- 热搜：太平洋暖得“失控”了；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-handle-creator-live-violation/>
- 热搜：冯远征笑称“考北京人艺其实不难”；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-fan-club-features/>
- 热搜：松鼠妈妈六次示范教小松鼠跳树成功；本站资料：<https://www.yxtiktok.com/blog/record-tiktok-creator-invite-check-history/>
- 热搜：华为Mate 90 Air售价4999元起；本站资料：<https://www.yxtiktok.com/blog/tiktok-followers-vs-live-engagement-recruiting/>
- 热搜：动辄千元起步的户外徒步鞋火了；本站资料：<https://www.yxtiktok.com/blog/tiktok-recruitment-channel-conversion-analysis/>
- 热搜：19楼落地窗底部装无限位外凸窗；本站资料：<https://www.yxtiktok.com/blog/select-tiktok-entertainment-live-creator-content-types/>
- 热搜：Mate X8 华为史上最强大折叠；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-recruiter-performance-kpis/>

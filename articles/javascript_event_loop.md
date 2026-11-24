# JavaScript 事件循环实战：微任务、渲染时机与长任务拆分

前端卡顿并不总是网络问题。一个请求已经返回，按钮却迟迟不能响应，往往与主线程上的任务调度有关。本文从浏览器事件循环出发，设计一个可以逐步观察任务执行顺序的实验页面，再讨论如何拆分耗时计算。各节示例为独立实验或可组合的局部代码，不依赖特定框架。

## 1. 先建立可观察的实验面板

控制台适合确认执行顺序，页面状态则能观察实际反馈。使用一个输出区域记录事件，不要把日志的 DOM 写入本身当作性能基准。

```html
<div>
  <button type="button" data-run>运行实验</button>
  <pre data-trace aria-live="polite"></pre>
</div>
```

```js
const output = document.querySelector('[data-trace]');
const log = text => { output.textContent += `${text}\n`; };
```

## 2. 同步调用为何先完成

事件处理函数进入调用栈后，会执行完当前同步代码才交出控制权。即使定时器的延迟为零，它的回调也只能等待后续任务机会。

```html
<div>
  <button type="button" data-sync>比较同步与定时器</button>
  <output data-sync-result></output>
</div>
```

```js
document.querySelector('[data-sync]').onclick = () => {
  log('A');
  setTimeout(() => log('C'), 0);
  log('B');
};
// 输出 A、B、C
```

## 3. 微任务如何参与调度

Promise 回调和 queueMicrotask 都可以排入微任务队列。当前任务结束后的微任务检查点会处理已经排队的微任务，通常早于下一次定时器回调。

```html
<div>
  <button type="button" data-micro>比较微任务</button>
  <pre data-micro-result></pre>
</div>
```

```js
document.querySelector('[data-micro]').onclick = () => {
  setTimeout(() => log('timer'), 0);
  Promise.resolve().then(() => log('promise'));
  queueMicrotask(() => log('microtask'));
  log('sync');
};
```

## 4. await 并不意味着新线程

await 暂停的是当前 async 函数的后续执行，而不是自动把同步计算送到其他线程。等待已完成的 Promise 也会让后续代码通过异步调度恢复。

```html
<div>
  <button type="button" data-await>观察 await</button>
  <output data-await-state>尚未运行</output>
</div>
```

```js
async function demo() {
  log('before');
  await Promise.resolve();
  log('after');
}
demo();
log('outside');
```

## 5. 为什么状态已经修改却没有绘制

设置文本或类名会改变 DOM 状态，但浏览器未必立刻完成绘制。如果后面紧跟耗时同步循环，用户可能看不到刚刚设置的处理中提示。

```html
<div>
  <button type="button" data-heavy>执行计算</button>
  <p data-heavy-state role="status">空闲</p>
</div>
```

应把耗时计算拆分或移出主线程，而不是在更新提示后直接执行长循环。仅等待 `Promise.resolve()` 不能保证获得新的绘制机会。

## 6. requestAnimationFrame 的用途

动画帧回调通常在浏览器准备下一次绘制时运行，适合合并视觉更新。它不是后台计算接口，回调太重同样会阻塞该帧。

```html
<div>
  <progress data-frame-progress max="100" value="0"></progress>
  <output data-frame-text>0%</output>
</div>
```

```js
function paint(value) {
  requestAnimationFrame(() => {
    document.querySelector('[data-frame-progress]').value = value;
    document.querySelector('[data-frame-text]').textContent = `${value}%`;
  });
}
```

## 7. 按时间预算拆分计算

分批处理比固定每批条数更容易适应不同设备。用 performance.now 测量一批的时间，超出预算后通过新的任务继续执行。下面的预算由调用方提供。

```html
<div>
  <p>分批计算</p>
  <progress data-batch-progress max="1" value="0"></progress>
</div>
```

```js
async function processBatch(items, consume, budgetMs, onProgress) {
  let i = 0;
  while (i < items.length) {
    const start = performance.now();
    do { consume(items[i++]); }
    while (i < items.length && performance.now() - start < budgetMs);
    onProgress(i, items.length);
    if (i < items.length) await new Promise(r => setTimeout(r, 0));
  }
}
```

## 8. 支持用户主动停止

分批循环需要取消机制，否则切换页面后旧计算仍可能继续。将 AbortSignal 传给业务函数，并在每批和必要的内部步骤检查。

```html
<div>
  <button type="button" data-cancel>取消计算</button>
  <p data-cancel-state aria-live="polite"></p>
</div>
```

```js
const controller = new AbortController();
document.querySelector('[data-cancel]').onclick = () => controller.abort();
function checkCancellation() {
  controller.signal.throwIfAborted();
}
```

## 9. 识别微任务饥饿

如果每个微任务都继续创建下一个微任务，浏览器可能长期无法进入下一任务及渲染阶段。批量操作不应该使用无界的递归 queueMicrotask 链。

```html
<div>
  <section aria-label="调度诊断">
    <p>记录每批处理时间和待处理数量</p>
    <output data-queue-size></output>
  </section>
</div>
```

性能面板中重点检查长任务和主线程空档；不要把定时器延迟当成执行耗时。背景标签页的定时器节流也会影响实验。

## 10. 建立可重复的性能测试

使用相同数据规模分别测试同步循环和分批循环。记录总耗时、最长单批耗时及用户输入响应，避免只追求吞吐量而忽略交互体验。

```html
<div>
  <dl>
    <dt>总耗时</dt><dd data-total-ms></dd>
    <dt>最长批次</dt><dd data-max-ms></dd>
    <dt>是否取消</dt><dd data-aborted></dd>
  </dl>
</div>
```

可在测试中注入时间预算和数据处理函数。对于真正 CPU 密集且难以拆分的算法，后台 Worker 是更适合的下一步。

## 总结

事件循环优化的目标不是让每个任务都变成异步，而是控制单次占用主线程的时间。区分微任务、后续任务和渲染机会，再配合测量及取消能力，才能实现稳定的交互反馈。

## 热搜标题与本站资料

以下热搜标题仅作当日资讯记录；本站链接为独立资料，并非该新闻的报道。

- 热搜：田秉毅任国家羽毛球队主教练；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-many-leads-low-signup-rate/>
- 热搜：余承东现场提词器曝光；本站资料：<https://www.yxtiktok.com/blog/select-tiktok-entertainment-live-creator-content-types/>
- 热搜：太平洋暖得“失控”了；本站资料：<https://www.yxtiktok.com/blog/small-tiktok-live-agency-essential-team-roles/>
- 热搜：冯远征笑称“考北京人艺其实不难”；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-recruitment-promise-contract-check/>
- 热搜：旺仔牛奶更不好卖了；本站资料：<https://www.yxtiktok.com/blog/tiktok-streamer-rejects-agency-advice/>
- 热搜：中国城市 重排全球座次；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-recruitment-message-cooperation-terms/>
- 热搜：松鼠妈妈六次示范教小松鼠跳树成功；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-recruitment-support-pitch/>
- 热搜：课堂上的“嘉豪、嘉欣” 已不敢举手；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-creator-network-vs-mcn-shop/>
- 热搜：新娘提1341次分手新郎回1342次不行；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-pk-interaction-planning/>
- 热搜：中国人登月又近一步；本站资料：<https://www.yxtiktok.com/blog/tiktok-pk-gifts-live-data-review/>
- 热搜：华为首款小折叠电脑发布；本站资料：<https://www.yxtiktok.com/blog/tiktok-creator-discovery-ranking-live-profile-workflow/>
- 热搜：注水长剧的“狼”来了；本站资料：<https://www.yxtiktok.com/blog/how-to-check-tiktok-creator-agency-invite-eligibility/>
- 热搜：中考要不要考历史 教材主编征集意见；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-creator-tiered-management/>
- 热搜：天还没亮 学生就集体站在操场晨读；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-agency-dm-live-room-recruitment-risks/>
- 热搜：加州给548名死刑犯减刑 特朗普震怒；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-streamer-maintenance-schedule/>
- 热搜：小米Q3净利润48.84亿 同比减少56.8%；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-one-on-one-streamer-support/>
- 热搜：伪人AI广告 已经成了一种新的城市病；本站资料：<https://www.yxtiktok.com/blog/tiktok-part-time-vs-full-time-streamer-management/>
- 热搜：Mate X8 华为史上最强大折叠；本站资料：<https://www.yxtiktok.com/blog/tiktok-short-video-creator-recruitment/>
- 热搜：车企老板娘女装卖了30亿；本站资料：<https://www.yxtiktok.com/blog/clean-invalid-stale-tiktok-creator-lead-records/>
- 热搜：“民主党向黄仁勋和英伟达宣战”；本站资料：<https://www.yxtiktok.com/blog/why-tiktok-agency-needs-employee-audit-logs/>
- 热搜：北京天空现“超级月亮”；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-first-call-creator-needs-questions/>
- 热搜：黄磊真空围裙；本站资料：<https://www.yxtiktok.com/blog/tiktok-followers-vs-live-engagement-recruiting/>
- 热搜：飞机与“超级月亮”浪漫同框；本站资料：<https://www.yxtiktok.com/blog/does-higher-tiktok-live-league-mean-better-recruit/>
- 热搜：东北外卖15分钟冻成冰；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-fan-club-features/>
- 热搜：香奈儿包日租仅需25元？押金要一万六；本站资料：<https://www.yxtiktok.com/blog/can-individual-apply-tiktok-live-creator-network/>
- 热搜：特朗普连发30多张照片“回味”深圳；本站资料：<https://www.yxtiktok.com/blog/find-tiktok-creators-for-small-agency-development/>
- 热搜：动辄千元起步的户外徒步鞋火了；本站资料：<https://www.yxtiktok.com/blog/tiktok-new-vs-experienced-streamer-outreach/>
- 热搜：外交部回应“黄仁勋重回英伟达”；本站资料：<https://www.yxtiktok.com/blog/tiktok-host-recruitment-cost-per-signed-creator/>
- 热搜：特朗普致电黄仁勋：答应我一件事；本站资料：<https://www.yxtiktok.com/blog/why-tiktok-live-ranking-differs-from-real-performance/>
- 热搜：培养更多高技能人才和大国工匠；本站资料：<https://www.yxtiktok.com/blog/agency-owner-recruiters-software-role-permissions/>
- 热搜：国产可口可乐不再标生产日期；本站资料：<https://www.yxtiktok.com/blog/avoid-duplicate-tiktok-creator-collection-and-followup/>
- 热搜：大爷抵脖子不放后小伙倒地 地铁回应；本站资料：<https://www.yxtiktok.com/blog/tiktok-new-streamer-pre-live-checklist/>
- 热搜：央行将开展8000亿元MLF操作；本站资料：<https://www.yxtiktok.com/tutorials/tiktok-live-ranklist-tutorial/>
- 热搜：睡眠大赛4人躺赢平分6000元；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-streamer-training-feedback-system/>
- 热搜：上海市委书记陈吉宁被查；本站资料：<https://www.yxtiktok.com/blog/tiktok-creator-no-reply-recruitment-follow-up/>
- 热搜：重庆男孩被坠楼男子砸中送医；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-streamer-feedback-follow-up/>
- 热搜：换脸还没消停 “AI融脸”又来了；本站资料：<https://www.yxtiktok.com/blog/which-live-metrics-tiktok-agency-should-evaluate/>
- 热搜：年产1500万根的“棍子” 卖出10亿元；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-events-vs-campaigns/>
- 热搜：文吉儿从小就活跃在各大荧幕；本站资料：<https://www.yxtiktok.com/blog/assign-tiktok-creator-leads-country-language-stage/>
- 热搜：大众拟裁员10万人 工会威胁罢工；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-first-recruitment-dm-scripts/>
- 热搜：华为Mate 90 Air售价4999元起；本站资料：<https://www.yxtiktok.com/blog/build-tiktok-agency-creator-selection-criteria/>
- 热搜：一校一主题 消谣育青年；本站资料：<https://www.yxtiktok.com/blog/tiktok-streamer-recruitment-channels-by-country/>
- 热搜：女子登山上班往返约6小时已瘦12斤；本站资料：<https://www.yxtiktok.com/blog/early-signs-streamer-unhappy-with-agency/>
- 热搜：几个姑娘在-25℃的黑龙江清凉上阵；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-diamonds-stability-vs-one-day-peak/>
- 热搜：Anthropic上市5天即破发；本站资料：<https://www.yxtiktok.com/blog/manual-vs-software-tiktok-streamer-recruitment/>
- 热搜：受贿1.13亿余元！吴胜华被判无期；本站资料：<https://www.yxtiktok.com/blog/tiktok-creator-pre-signing-info-minimization/>
- 热搜：分享你拍的超级月亮；本站资料：<https://www.yxtiktok.com/blog/benefits-of-joining-tiktok-live-agency/>
- 热搜：泰警方确认女子异地遇害被肢解抛尸；本站资料：<https://www.yxtiktok.com/blog/tiktok-streamer-recruitment-communities/>
- 热搜：19楼落地窗底部装无限位外凸窗；本站资料：<https://www.yxtiktok.com/blog/tiktok-new-streamer-training-curriculum/>
- 热搜：黄奕用AI视频还原过去三十年人生；本站资料：<https://www.yxtiktok.com/blog/tiktok-new-creator-coaching-payback-period/>

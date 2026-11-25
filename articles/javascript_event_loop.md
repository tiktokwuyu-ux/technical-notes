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

## 本站 URL 数据样本

以下为本站公开页面地址样本，与上文技术示例及下方热搜无关。

```text
https://www.yxtiktok.com/blog/tiktok-creator-discovery-ranking-live-profile-workflow/
https://www.yxtiktok.com/blog/tiktok-agency-multilingual-recruitment-script-library/
https://www.yxtiktok.com/blog/assess-tiktok-creator-live-positioning-from-profile/
https://www.yxtiktok.com/blog/early-signs-streamer-unhappy-with-agency/
https://www.yxtiktok.com/blog/creator-not-interested-agency-recruitment-status/
https://www.yxtiktok.com/blog/how-to-check-tiktok-creator-agency-invite-eligibility/
https://www.yxtiktok.com/blog/tiktok-agency-one-on-one-streamer-support/
https://www.yxtiktok.com/blog/tiktok-agency-many-leads-low-signup-rate/
https://www.yxtiktok.com/blog/reduce-duplicate-work-multiple-tiktok-agency-accounts/
https://www.yxtiktok.com/blog/tiktok-agency-creator-incentive-communication/
https://www.yxtiktok.com/blog/tiktok-agency-recruitment-promise-contract-check/
https://www.yxtiktok.com/blog/select-tiktok-entertainment-live-creator-content-types/
https://www.yxtiktok.com/blog/tiktok-live-ranking-find-creators/
https://www.yxtiktok.com/blog/tiktok-agency-invite-not-found-troubleshooting/
https://www.yxtiktok.com/blog/small-tiktok-live-agency-essential-team-roles/
https://www.yxtiktok.com/blog/growth-vs-established-tiktok-creators-recruitment-talk/
https://www.yxtiktok.com/blog/tiktok-creator-pre-signing-info-minimization/
https://www.yxtiktok.com/blog/tiktok-agency-streamer-maintenance-schedule/
https://www.yxtiktok.com/blog/remote-tiktok-agency-team-shared-creator-data/
https://www.yxtiktok.com/blog/high-gift-high-growth-stable-tiktok-creator-types/
https://www.yxtiktok.com/blog/tiktok-streamer-live-schedule-reminder-scripts/
https://www.yxtiktok.com/blog/solo-vs-team-tiktok-agency-recruitment-workflow/
https://www.yxtiktok.com/blog/tiktok-creator-network-cross-region-compliance/
https://www.yxtiktok.com/blog/record-tiktok-creator-invite-check-history/
https://www.yxtiktok.com/blog/tiktok-live-entertainment-gaming-shopping-creators/
https://www.yxtiktok.com/blog/tiktok-agency-cross-country-invite-region-rules/
https://www.yxtiktok.com/blog/tiktok-agency-streamer-segmented-management/
https://www.yxtiktok.com/blog/build-tiktok-agency-creator-selection-criteria/
https://www.yxtiktok.com/blog/tiktok-streamer-rejects-agency-advice/
https://www.yxtiktok.com/blog/tiktok-agency-recruiter-offboarding-leads-handover/
https://www.yxtiktok.com/blog/tiktok-agency-creator-live-schedule/
https://www.yxtiktok.com/blog/calculate-tiktok-creator-recruitment-conversion-rate/
https://www.yxtiktok.com/blog/which-live-metrics-tiktok-agency-should-evaluate/
https://www.yxtiktok.com/blog/tiktok-live-league-tiers-creator-discovery/
https://www.yxtiktok.com/blog/avoid-spam-repeated-dm-tiktok-agency-recruitment/
https://www.yxtiktok.com/blog/tiktok-new-vs-experienced-streamer-outreach/
https://www.yxtiktok.com/blog/normal-invite-only-creator-agency-choice/
https://www.yxtiktok.com/blog/tiktok-creator-network-multi-account-invite-risk/
https://www.yxtiktok.com/blog/tiktok-creator-segment-recruitment-difficulty-value/
https://www.yxtiktok.com/blog/evaluate-tiktok-live-leaderboard-for-recruitment-list/
https://www.yxtiktok.com/blog/tiktok-agency-creator-tiered-management/
https://www.yxtiktok.com/blog/tiktok-dance-live-content-interaction-plan/
https://www.yxtiktok.com/blog/why-recheck-creator-eligibility-before-contact/
https://www.yxtiktok.com/blog/screen-tiktok-creator-public-live-content-policy-risk/
https://www.yxtiktok.com/blog/why-tiktok-live-ranking-platforms-disagree/
https://www.yxtiktok.com/blog/prioritize-tiktok-creators-by-live-activity-eligibility/
https://www.yxtiktok.com/blog/benefits-of-joining-tiktok-live-agency/
https://www.yxtiktok.com/blog/tiktok-agency-streamer-onboarding-handoff/
https://www.yxtiktok.com/blog/tiktok-agency-first-contact-message-to-creator/
https://www.yxtiktok.com/blog/tiktok-short-video-creator-recruitment/
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

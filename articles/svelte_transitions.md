# Svelte 动画实战：transition、animate 与列表重排交互

在任务列表或消息面板中，元素突然出现、消失或移动，容易让用户失去对当前操作的感知。Svelte 提供 transition 与 animate 指令，可以在数据变化时为界面提供明确的视觉反馈。本文使用 Svelte 5 的模板语法说明进入、退出、重排和无障碍动画设计。

## 1. 先确认动画服务于什么交互

动画应帮助用户理解状态变化，而不是延迟业务结果。例如通知被关闭时淡出，排序后记录平滑移动。状态本身仍由业务数据决定。

```svelte
<div>
  <section><h2>消息面板</h2><button onclick={() => visible = !visible}>切换消息</button></section>
</div>
```

```svelte
<script>
  let visible = $state(true);
</script>
```

## 2. 使用 transition 实现进入和退出

transition 指令可以同时管理元素进入与离开。元素在条件块中被移除时，框架会协调退出动画及 DOM 移除。

```svelte
<div>
  {#if visible}<p transition:fade={{ duration }}>已保存最新设置</p>{/if}
</div>
```

```svelte
<script>
  import { fade } from 'svelte/transition';
  let duration = $state(180);
</script>
```

## 3. 分别配置进入与退出效果

in 和 out 可以定义不同的动画。例如面板进入时小幅移动，退出时仅淡出，使离开操作更轻量。

```svelte
<div>
  {#if expanded}
    <section in:fly={{ y: 8, duration }} out:fade={{ duration }}><h3>详细信息</h3><p>更多业务数据</p></section>
  {/if}
</div>
```

`fly` 与 `fade` 均来自 `svelte/transition`。实际持续时间应由统一的动效规范管理。

## 4. 理解局部与全局过渡

默认过渡只在直接相关的控制流变化时触发。嵌套区域整体被移除时，如果确实需要内部节点执行过渡，可以使用 global 修饰符。

```svelte
<div>
  {#if showPanel}
    <section>{#if showMessage}<p transition:fade|global>操作通知</p>{/if}</section>
  {/if}
</div>
```

全局过渡不适合无条件用于所有子组件，否则页面切换时可能同时触发过多动画。

## 5. 使用稳定业务 ID 管理列表

列表重排需要保持每个记录的身份。 keyed each 使用稳定 ID，而不是当前下标。这样排序时节点能与对应数据保持一致。

```svelte
<div>
  <ul>{#each messages as message (message.id)}<li>{message.title}</li>{/each}</ul>
</div>
```

```js
let messages = $state([]);
function sortMessages() {
  messages = [...messages].sort((a, b) => a.title.localeCompare(b.title));
}
```

## 6. 通过 animate:flip 表达位置变化

flip 适合已存在列表项的重排。使用 animate 指令的元素应是 keyed each 块的直接且唯一子元素；新增删除则通过 transition 处理。

```svelte
<div>
  <ul>{#each messages as message (message.id)}<li animate:flip={{ duration }} transition:fade={{ duration }}>{message.title}</li>{/each}</ul>
</div>
```

```svelte
<script>
  import { flip } from 'svelte/animate';
  import { fade } from 'svelte/transition';
</script>
```

## 7. 避免在动画期间重复提交

业务删除与动画退出是两件事。用户触发删除后应立即记录对应操作状态，动画不承担服务端确认职责。

```svelte
<div>
  <button disabled={deleting} onclick={removeMessage}>{deleting ? '正在删除' : '删除通知'}</button>
</div>
```

如果删除失败，重新展示原记录并提示原因。不要在 outroend 中才启动关键业务写入，否则动画配置会意外影响业务执行。

## 8. 尊重减少动态效果的系统偏好

对移动、缩放和长距离变换敏感的用户，应减少非必要动效。可以在客户端读取 prefers-reduced-motion，并将可选动画持续时间设为零。

```svelte
<div>
  {#if visible}<section transition:fade={{ duration: reducedMotion ? 0 : duration }}>当前任务状态</section>{/if}
</div>
```

```svelte
<script>
  import { onMount } from 'svelte';
  let reducedMotion = $state(true);
  onMount(() => {
    const media = matchMedia('(prefers-reduced-motion: reduce)');
    const sync = () => { reducedMotion = media.matches; };
    sync(); media.addEventListener('change', sync);
    return () => media.removeEventListener('change', sync);
  });
</script>
```

## 9. 观察真实的动画性能

大量列表项同时重排会增加布局测量开销。应限制一次变化的范围，并优先使用 transform 与 opacity 等适合动画的属性，而不是持续读取和修改布局。

```svelte
<div>
  <section><h3>消息排序</h3><button onclick={sortMessages}>按标题排序</button><output>{messages.length} 条消息</output></section>
</div>
```

在低性能设备与真实数据量下比较动画前后的主线程负载。对数千条记录，应考虑分页或虚拟列表，而非为每条记录增加过渡。

## 10. 覆盖快速反向操作和焦点恢复

用户可能在退出动画尚未结束时重新展开。测试快速切换、同时删除多个条目及键盘焦点所在项被移除。

```svelte
<div>
  <p role="status" aria-live="polite">{statusMessage}</p><button onclick={restoreFocus}>返回列表</button>
</div>
```

状态通知不依赖动画结束才能出现。焦点恢复应基于实际元素是否仍在文档中，并提供合理的备用目标。

## 总结

Svelte 的过渡与重排动画适合表达已有状态变化。使用稳定的列表身份、独立业务操作和可关闭的动效配置，能让动画改善理解与反馈，而不是成为新的状态管理负担。

## 本站 URL 数据样本

以下为本站公开页面地址样本，与上文技术示例及下方热搜无关。

```text
https://www.yxtiktok.com/blog/new-tiktok-agency-creator-pool-and-roles/
https://www.yxtiktok.com/blog/multi-country-tiktok-agency-team-language-allocation/
https://www.yxtiktok.com/blog/tiktok-creator-network-cross-region-compliance/
https://www.yxtiktok.com/blog/tiktok-creator-pre-signing-info-minimization/
https://www.yxtiktok.com/blog/tiktok-live-entertainment-gaming-shopping-creators/
https://www.yxtiktok.com/blog/tiktok-live-diamonds-stability-vs-one-day-peak/
https://www.yxtiktok.com/blog/tiktok-live-agency-differentiated-creator-offer/
https://www.yxtiktok.com/blog/tiktok-creator-weeks-performance-decline/
https://www.yxtiktok.com/blog/tiktok-creator-segment-recruitment-difficulty-value/
https://www.yxtiktok.com/blog/tiktok-agency-recruitment-support-pitch/
https://www.yxtiktok.com/blog/calculate-tiktok-creator-recruitment-conversion-rate/
https://www.yxtiktok.com/blog/tiktok-backstage-normal-vs-premium-invite/
https://www.yxtiktok.com/blog/benefits-of-joining-tiktok-live-agency/
https://www.yxtiktok.com/blog/tiktok-live-follow-cta-without-pressure/
https://www.yxtiktok.com/blog/avoid-duplicate-tiktok-creator-collection-and-followup/
https://www.yxtiktok.com/blog/tiktok-agency-recruitment-promise-contract-check/
https://www.yxtiktok.com/tutorials/tiktok-live-ranklist-tutorial/
https://www.yxtiktok.com/blog/why-tiktok-live-ranking-differs-from-real-performance/
https://www.yxtiktok.com/blog/tiktok-live-events-vs-campaigns/
https://www.yxtiktok.com/blog/tiktok-streamer-income-expectations-management/
https://www.yxtiktok.com/blog/talk-to-new-tiktok-live-creator-needs/
https://www.yxtiktok.com/blog/unified-tiktok-creator-recruitment-records/
https://www.yxtiktok.com/blog/tiktok-creator-pipeline-status-management/
https://www.yxtiktok.com/blog/tiktok-live-welcome-viewers-scripts/
https://www.yxtiktok.com/blog/tiktok-live-pk-interaction-planning/
https://www.yxtiktok.com/blog/why-tiktok-agency-needs-employee-audit-logs/
https://www.yxtiktok.com/blog/tiktok-live-awkward-silence-topic-switching/
https://www.yxtiktok.com/blog/early-signs-streamer-unhappy-with-agency/
https://www.yxtiktok.com/blog/new-tiktok-agency-first-100-creator-leads-plan/
https://www.yxtiktok.com/blog/tiktok-agency-handle-creator-live-violation/
https://www.yxtiktok.com/blog/tiktok-live-agency-revenue-sources/
https://www.yxtiktok.com/blog/tiktok-live-replay-review-method/
https://www.yxtiktok.com/blog/tiktok-live-ranking-find-creators/
https://www.yxtiktok.com/blog/how-tiktok-agency-prove-identity-to-creator/
https://www.yxtiktok.com/blog/tiktok-agency-creator-live-schedule/
https://www.yxtiktok.com/blog/tiktok-agency-first-call-creator-needs-questions/
https://www.yxtiktok.com/blog/assess-third-party-tiktok-live-ranking-coverage/
https://www.yxtiktok.com/blog/tiktok-agency-many-leads-low-signup-rate/
https://www.yxtiktok.com/blog/tiktok-agency-cross-country-invite-region-rules/
https://www.yxtiktok.com/blog/tiktok-live-creator-agency-contract-checklist/
https://www.yxtiktok.com/blog/prevent-tiktok-agency-recruiter-duplicate-contact/
https://www.yxtiktok.com/blog/avoid-spam-repeated-dm-tiktok-agency-recruitment/
https://www.yxtiktok.com/blog/why-tiktok-live-ranking-platforms-disagree/
https://www.yxtiktok.com/blog/personalize-tiktok-creator-outreach-by-live-content/
https://www.yxtiktok.com/blog/schedule-cross-timezone-tiktok-creator-recruiting-team/
https://www.yxtiktok.com/blog/cost-structure-recruit-established-vs-new-live-creators/
https://www.yxtiktok.com/blog/tiktok-agency-creator-incentive-communication/
https://www.yxtiktok.com/blog/tiktok-chat-streamer-improvisation-training/
https://www.yxtiktok.com/blog/tiktok-streamer-performance-decline-recovery-plan/
https://www.yxtiktok.com/blog/why-recheck-creator-eligibility-before-contact/
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

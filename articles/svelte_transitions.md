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

## 热搜标题与本站资料

以下热搜标题仅作当日资讯记录；本站链接为独立资料，并非该新闻的报道。

- 热搜：中国城市 重排全球座次；本站资料：<https://www.yxtiktok.com/blog/how-to-check-tiktok-creator-agency-invite-eligibility/>
- 热搜：中国人登月又近一步；本站资料：<https://www.yxtiktok.com/blog/why-tiktok-live-ranking-differs-from-real-performance/>
- 热搜：华为首款小折叠电脑发布；本站资料：<https://www.yxtiktok.com/blog/tiktok-followers-vs-live-engagement-recruiting/>
- 热搜：伪人AI广告 已经成了一种新的城市病；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-streamer-onboarding-handoff/>
- 热搜：华为Mate 90 Air售价4999元起；本站资料：<https://www.yxtiktok.com/blog/new-tiktok-agency-creator-pool-and-roles/>
- 热搜：Anthropic上市5天即破发；本站资料：<https://www.yxtiktok.com/blog/tiktok-creator-pre-signing-info-minimization/>
- 热搜：外交部回应“黄仁勋重回英伟达”；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-league-tiers-creator-discovery/>
- 热搜：特朗普连发30多张照片“回味”深圳；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-streamer-referral-recruitment/>
- 热搜：田秉毅任国家羽毛球队主教练；本站资料：<https://www.yxtiktok.com/blog/single-country-tiktok-agency-creator-pipeline/>
- 热搜：19楼落地窗底部装无限位外凸窗；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-daily-recruitment-plan/>
- 热搜：中考要不要考历史 教材主编征集意见；本站资料：<https://www.yxtiktok.com/blog/small-tiktok-live-agency-staff-tool-budget/>
- 热搜：课堂上的“嘉豪、嘉欣” 已不敢举手；本站资料：<https://www.yxtiktok.com/blog/personalize-tiktok-creator-outreach-by-live-content/>
- 热搜：上海市委书记陈吉宁被查；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-creator-agency-contract-checklist/>
- 热搜：大爷抵脖子不放后小伙倒地 地铁回应；本站资料：<https://www.yxtiktok.com/blog/tiktok-creator-discovery-ranking-live-profile-workflow/>
- 热搜：“民主党向黄仁勋和英伟达宣战”；本站资料：<https://www.yxtiktok.com/blog/why-tiktok-creator-invite-eligibility-changes/>
- 热搜：北京天空现“超级月亮”；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-agency-revenue-sources/>
- 热搜：Mate X8 华为史上最强大折叠；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-first-recruitment-dm-scripts/>
- 热搜：受贿1.13亿余元！吴胜华被判无期；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-pk-interaction-planning/>
- 热搜：余承东现场提词器曝光；本站资料：<https://www.yxtiktok.com/blog/small-tiktok-live-agency-priority-creator-types/>
- 热搜：睡眠大赛4人躺赢平分6000元；本站资料：<https://www.yxtiktok.com/blog/find-tiktok-live-creators-streaming-now/>
- 热搜：香奈儿包日租仅需25元？押金要一万六；本站资料：<https://www.yxtiktok.com/blog/tiktok-streamer-recruitment-channels-by-country/>
- 热搜：特朗普致电黄仁勋：答应我一件事；本站资料：<https://www.yxtiktok.com/blog/tiktok-diamonds-vs-creator-rewards/>
- 热搜：动辄千元起步的户外徒步鞋火了；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-agency-dm-live-room-recruitment-risks/>
- 热搜：黄磊真空围裙；本站资料：<https://www.yxtiktok.com/blog/daily-tiktok-agency-recruitment-report-kpis/>
- 热搜：国产可口可乐不再标生产日期；本站资料：<https://www.yxtiktok.com/blog/cost-structure-recruit-established-vs-new-live-creators/>
- 热搜：太平洋暖得“失控”了；本站资料：<https://www.yxtiktok.com/blog/reduce-duplicate-work-multiple-tiktok-agency-accounts/>
- 热搜：东北外卖15分钟冻成冰；本站资料：<https://www.yxtiktok.com/blog/normal-invite-only-creator-agency-choice/>
- 热搜：重庆男孩被坠楼男子砸中送医；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-events-vs-campaigns/>
- 热搜：松鼠妈妈六次示范教小松鼠跳树成功；本站资料：<https://www.yxtiktok.com/blog/why-high-earning-tiktok-creators-may-not-fit-agency/>
- 热搜：换脸还没消停 “AI融脸”又来了；本站资料：<https://www.yxtiktok.com/blog/new-tiktok-agency-first-100-creator-leads-plan/>
- 热搜：加州给548名死刑犯减刑 特朗普震怒；本站资料：<https://www.yxtiktok.com/blog/tiktok-entertainment-live-opening-retention/>
- 热搜：分享你拍的超级月亮；本站资料：<https://www.yxtiktok.com/blog/schedule-cross-timezone-tiktok-creator-recruiting-team/>
- 热搜：飞机与“超级月亮”浪漫同框；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-recruitment-channels/>
- 热搜：女子登山上班往返约6小时已瘦12斤；本站资料：<https://www.yxtiktok.com/blog/tiktok-creator-weeks-performance-decline/>
- 热搜：文吉儿从小就活跃在各大荧幕；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-creator-live-schedule/>
- 热搜：车企老板娘女装卖了30亿；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-recruitment-message-cooperation-terms/>
- 热搜：培养更多高技能人才和大国工匠；本站资料：<https://www.yxtiktok.com/blog/tiktok-creator-pipeline-status-management/>
- 热搜：新娘提1341次分手新郎回1342次不行；本站资料：<https://www.yxtiktok.com/blog/why-agencies-should-not-rely-only-on-tiktok-top-rankings/>
- 热搜：泰警方确认女子异地遇害被肢解抛尸；本站资料：<https://www.yxtiktok.com/blog/tiktok-streamer-performance-decline-recovery-plan/>
- 热搜：几个姑娘在-25℃的黑龙江清凉上阵；本站资料：<https://www.yxtiktok.com/blog/creator-not-interested-agency-recruitment-status/>
- 热搜：一校一主题 消谣育青年；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-diamonds-stability-vs-one-day-peak/>
- 热搜：天还没亮 学生就集体站在操场晨读；本站资料：<https://www.yxtiktok.com/blog/small-tiktok-live-agency-essential-team-roles/>
- 热搜：小米Q3净利润48.84亿 同比减少56.8%；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-streamer-event-participation/>
- 热搜：央行将开展8000亿元MLF操作；本站资料：<https://www.yxtiktok.com/blog/agency-owner-recruiters-software-role-permissions/>
- 热搜：大众拟裁员10万人 工会威胁罢工；本站资料：<https://www.yxtiktok.com/blog/tiktok-creator-network-cross-region-compliance/>
- 热搜：旺仔牛奶更不好卖了；本站资料：<https://www.yxtiktok.com/blog/select-tiktok-entertainment-live-creator-content-types/>
- 热搜：黄奕用AI视频还原过去三十年人生；本站资料：<https://www.yxtiktok.com/blog/tiktok-streamer-recruitment-communities/>
- 热搜：年产1500万根的“棍子” 卖出10亿元；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-awkward-silence-topic-switching/>
- 热搜：冯远征笑称“考北京人艺其实不难”；本站资料：<https://www.yxtiktok.com/blog/top-vs-mid-tier-tiktok-live-creators-agency/>
- 热搜：注水长剧的“狼”来了；本站资料：<https://www.yxtiktok.com/blog/evaluate-tiktok-live-leaderboard-for-recruitment-list/>

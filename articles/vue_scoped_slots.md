# Vue 3 插槽实战：作用域插槽、可复用表格与组件 API 设计

业务后台中的表格、卡片和列表常拥有相同结构，但每个页面的字段展示和操作按钮不同。若组件为每一种业务细节增加布尔属性，通用接口很快变得复杂。Vue 插槽可以把结构责任交给容器，把局部展示交给使用者。本文围绕一个数据表格讲解插槽契约。

## 1. 先建立固定的语义结构

表格组件应保留 table、thead 和 tbody 的正确层级。插槽用于替换具体内容，而不是把整张表的语义交给无法预测的外部代码。

```vue
<div>
  <table><thead><tr><th scope="col">名称</th><th scope="col">状态</th></tr></thead><tbody><slot /></tbody></table>
</div>
```

## 2. 默认插槽负责通用内容

默认插槽适合容器中的主体区域。没有传入内容时，可通过后备内容展示空状态或通用提示。

```vue
<div>
  <section class="table-panel"><slot><p>暂无内容</p></slot></section>
</div>
```

## 3. 命名插槽区分组件区域

将工具栏、表格正文和分页区分为具名区域，允许使用者仅替换必要位置。

```vue
<div>
  <section class="table-shell">
    <header><slot name="toolbar" /></header>
    <slot />
    <footer><slot name="footer" /></footer>
  </section>
</div>
```

## 4. 作用域插槽传递行数据

插槽内容在父组件的作用域中执行。子组件通过 slot props 将当前记录传给使用者，而不是假设父组件能直接访问子组件局部变量。

```vue
<div>
  <table><tbody><tr v-for="row in rows" :key="row.id">
    <td><slot name="name" :row="row">{{ row.name }}</slot></td>
    <td><slot name="status" :row="row">{{ row.status }}</slot></td>
  </tr></tbody></table>
</div>
```

## 5. 在父组件定制字段展示

同一表格可以在订单页面渲染状态标签，在用户页面展示普通文本。保持 slot props 的数据结构稳定，使使用者不用了解组件内部 DOM 查询方式。

```vue
<div>
  <DataTable :rows="orders">
    <template #status="{ row }"><StatusBadge :status="row.status" /></template>
    <template #name="{ row }"><strong>{{ row.name }}</strong></template>
  </DataTable>
</div>
```

## 6. 使用 defineSlots 描述类型契约

Vue 3.3 及以上支持 defineSlots 宏，为 TypeScript 工具提供插槽类型信息。该宏主要服务于类型检查，不是运行时数据验证。

```vue
<div>
  <slot name="actions" :row="row" :disabled="readonly" />
</div>
```

```vue
<script setup lang="ts">
type Row = { id: string; name: string; status: string };
defineSlots<{
  name(props: { row: Row }): any;
  status(props: { row: Row }): any;
  actions(props: { row: Row; disabled: boolean }): any;
}>();
</script>
```

## 7. 动态列名与动态插槽

当列结构来自配置时，可使用动态插槽名称。但列 key 应来自受控配置，不能直接把任意外部字段当成组件扩展接口。

```vue
<div>
  <table><tbody><tr v-for="row in rows" :key="row.id">
    <td v-for="column in columns" :key="column.key">
      <slot :name="`cell-${column.key}`" :row="row">{{ row[column.key] }}</slot>
    </td>
  </tr></tbody></table>
</div>
```

## 8. 组件负责空状态和错误状态

空列表和请求失败属于不同业务状态。允许使用者自定义文案，但容器应明确决定当前应该展示哪一种状态。

```vue
<div>
  <section>
    <slot v-if="error" name="error" :error="error"><p role="alert">读取失败</p></slot>
    <slot v-else-if="rows.length === 0" name="empty"><p>暂无记录</p></slot>
    <slot v-else />
  </section>
</div>
```

## 9. 操作回调保持所有权清晰

使用者可以通过作用域插槽渲染编辑按钮，但删除、保存等业务操作应由拥有数据的一方发起。不要同时在父子两层修改同一个集合。

```vue
<div>
  <DataTable :rows="rows">
    <template #actions="{ row, disabled }"><button :disabled="disabled" @click="editRow(row.id)">编辑</button></template>
  </DataTable>
</div>
```

## 10. 用真实使用方式测试插槽

除了检查默认渲染，还应挂载带自定义插槽的组件，验证行数据、空状态和事件是否与契约一致。

```vue
<div>
  <DataTable :rows="[]"><template #empty><p role="status">当前没有匹配数据</p></template></DataTable>
</div>
```

若表格采用虚拟滚动，额外检查插槽中的组件身份和焦点行为。不要通过随机 key 强制重建每个单元格来修复状态问题。

## 总结

插槽是 Vue 组件公开 API 的一部分。固定结构、明确命名、类型化 slot props 与业务操作所有权共同决定组件的可维护性。适度开放展示扩展点，比为每种业务情况持续增加组件属性更有效。

## 热搜标题与本站资料

以下热搜标题仅作当日资讯记录；本站链接为独立资料，并非该新闻的报道。

- 热搜：伪人AI广告 已经成了一种新的城市病；本站资料：<https://www.yxtiktok.com/blog/evaluate-tiktok-live-leaderboard-for-recruitment-list/>
- 热搜：特朗普致电黄仁勋：答应我一件事；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-daily-recruitment-plan/>
- 热搜：太平洋暖得“失控”了；本站资料：<https://www.yxtiktok.com/blog/creator-not-interested-agency-recruitment-status/>
- 热搜：东北外卖15分钟冻成冰；本站资料：<https://www.yxtiktok.com/blog/tiktok-coins-gifts-diamonds-difference/>
- 热搜：大爷抵脖子不放后小伙倒地 地铁回应；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-streamer-segmented-management/>
- 热搜：睡眠大赛4人躺赢平分6000元；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-pk-interaction-planning/>
- 热搜：动辄千元起步的户外徒步鞋火了；本站资料：<https://www.yxtiktok.com/blog/remote-tiktok-agency-team-shared-creator-data/>
- 热搜：松鼠妈妈六次示范教小松鼠跳树成功；本站资料：<https://www.yxtiktok.com/blog/prioritize-tiktok-creators-by-live-activity-eligibility/>
- 热搜：外交部回应“黄仁勋重回英伟达”；本站资料：<https://www.yxtiktok.com/blog/tiktok-streamer-performance-decline-recovery-plan/>
- 热搜：分享你拍的超级月亮；本站资料：<https://www.yxtiktok.com/blog/tiktok-creator-pre-signing-info-minimization/>
- 热搜：余承东现场提词器曝光；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-creator-incentive-communication/>
- 热搜：课堂上的“嘉豪、嘉欣” 已不敢举手；本站资料：<https://www.yxtiktok.com/blog/new-tiktok-agency-creator-pool-and-roles/>
- 热搜：天还没亮 学生就集体站在操场晨读；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-many-leads-low-signup-rate/>
- 热搜：一校一主题 消谣育青年；本站资料：<https://www.yxtiktok.com/blog/tiktok-creator-discovery-ranking-live-profile-workflow/>
- 热搜：中国人登月又近一步；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-awkward-silence-topic-switching/>
- 热搜：19楼落地窗底部装无限位外凸窗；本站资料：<https://www.yxtiktok.com/blog/tiktok-streamer-recruitment-communities/>
- 热搜：文吉儿从小就活跃在各大荧幕；本站资料：<https://www.yxtiktok.com/blog/calculate-tiktok-creator-recruitment-conversion-rate/>
- 热搜：央行将开展8000亿元MLF操作；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-first-recruitment-dm-scripts/>
- 热搜：车企老板娘女装卖了30亿；本站资料：<https://www.yxtiktok.com/blog/evaluate-tiktok-creator-with-limited-history/>
- 热搜：香奈儿包日租仅需25元？押金要一万六；本站资料：<https://www.yxtiktok.com/blog/why-tiktok-live-ranking-differs-from-real-performance/>
- 热搜：特朗普连发30多张照片“回味”深圳；本站资料：<https://www.yxtiktok.com/blog/tiktok-streamer-rejects-agency-advice/>
- 热搜：几个姑娘在-25℃的黑龙江清凉上阵；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-agency-differentiated-creator-offer/>
- 热搜：泰警方确认女子异地遇害被肢解抛尸；本站资料：<https://www.yxtiktok.com/blog/tiktok-followers-vs-live-engagement-recruiting/>
- 热搜：飞机与“超级月亮”浪漫同框；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-cohost-chatting-tips/>
- 热搜：中国城市 重排全球座次；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-low-live-hours-diagnosis/>
- 热搜：田秉毅任国家羽毛球队主教练；本站资料：<https://www.yxtiktok.com/blog/record-tiktok-creator-invite-check-history/>
- 热搜：Mate X8 华为史上最强大折叠；本站资料：<https://www.yxtiktok.com/blog/reduce-duplicate-work-multiple-tiktok-agency-accounts/>
- 热搜：旺仔牛奶更不好卖了；本站资料：<https://www.yxtiktok.com/blog/tiktok-creator-pipeline-status-management/>
- 热搜：年产1500万根的“棍子” 卖出10亿元；本站资料：<https://www.yxtiktok.com/blog/tiktok-streamer-live-schedule-reminder-scripts/>
- 热搜：Anthropic上市5天即破发；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-fan-club-features/>
- 热搜：受贿1.13亿余元！吴胜华被判无期；本站资料：<https://www.yxtiktok.com/blog/multi-country-tiktok-agency-team-language-allocation/>
- 热搜：新娘提1341次分手新郎回1342次不行；本站资料：<https://www.yxtiktok.com/blog/tiktok-country-ranking-find-live-creators/>
- 热搜：冯远征笑称“考北京人艺其实不难”；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-creator-agency-contract-checklist/>
- 热搜：培养更多高技能人才和大国工匠；本站资料：<https://www.yxtiktok.com/blog/schedule-cross-timezone-tiktok-creator-recruiting-team/>
- 热搜：“民主党向黄仁勋和英伟达宣战”；本站资料：<https://www.yxtiktok.com/blog/why-invitable-tiktok-creator-may-not-join/>
- 热搜：上海市委书记陈吉宁被查；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-multilingual-recruitment-script-library/>
- 热搜：加州给548名死刑犯减刑 特朗普震怒；本站资料：<https://www.yxtiktok.com/blog/tiktok-game-streamer-audience-interaction/>
- 热搜：华为首款小折叠电脑发布；本站资料：<https://www.yxtiktok.com/blog/small-tiktok-live-agency-staff-tool-budget/>
- 热搜：华为Mate 90 Air售价4999元起；本站资料：<https://www.yxtiktok.com/blog/tiktok-entertainment-live-opening-retention/>
- 热搜：国产可口可乐不再标生产日期；本站资料：<https://www.yxtiktok.com/blog/assess-third-party-tiktok-live-ranking-coverage/>
- 热搜：黄奕用AI视频还原过去三十年人生；本站资料：<https://www.yxtiktok.com/blog/how-tiktok-agency-prove-identity-to-creator/>
- 热搜：小米Q3净利润48.84亿 同比减少56.8%；本站资料：<https://www.yxtiktok.com/blog/creator-already-in-other-network-new-invite/>
- 热搜：换脸还没消停 “AI融脸”又来了；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-recruiter-performance-kpis/>
- 热搜：中考要不要考历史 教材主编征集意见；本站资料：<https://www.yxtiktok.com/blog/tiktok-diamonds-vs-creator-rewards/>
- 热搜：注水长剧的“狼”来了；本站资料：<https://www.yxtiktok.com/blog/why-tiktok-agency-deduplicate-creator-uid/>
- 热搜：黄磊真空围裙；本站资料：<https://www.yxtiktok.com/blog/early-signs-streamer-unhappy-with-agency/>
- 热搜：北京天空现“超级月亮”；本站资料：<https://www.yxtiktok.com/blog/small-tiktok-live-agency-essential-team-roles/>
- 热搜：大众拟裁员10万人 工会威胁罢工；本站资料：<https://www.yxtiktok.com/blog/manual-vs-software-tiktok-streamer-recruitment/>
- 热搜：重庆男孩被坠楼男子砸中送医；本站资料：<https://www.yxtiktok.com/blog/tiktok-chat-streamer-improvisation-training/>
- 热搜：女子登山上班往返约6小时已瘦12斤；本站资料：<https://www.yxtiktok.com/blog/clean-invalid-stale-tiktok-creator-lead-records/>

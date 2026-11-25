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

## 本站 URL 数据样本

以下为本站公开页面地址样本，与上文技术示例及下方热搜无关。

```text
https://www.yxtiktok.com/blog/tiktok-host-recruitment-cost-per-signed-creator/
https://www.yxtiktok.com/blog/talk-to-new-tiktok-live-creator-needs/
https://www.yxtiktok.com/blog/tiktok-agency-streamer-maintenance-schedule/
https://www.yxtiktok.com/tutorials/tiktok-ios-non-jailbreak-dm-tutorial/
https://www.yxtiktok.com/blog/reassess-long-unfollowed-tiktok-creator-leads/
https://www.yxtiktok.com/blog/tiktok-agency-recruiter-performance-kpis/
https://www.yxtiktok.com/blog/tiktok-agency-recruitment-message-cooperation-terms/
https://www.yxtiktok.com/blog/daily-tiktok-agency-recruitment-report-kpis/
https://www.yxtiktok.com/blog/does-higher-tiktok-live-league-mean-better-recruit/
https://www.yxtiktok.com/blog/remote-tiktok-agency-team-shared-creator-data/
https://www.yxtiktok.com/blog/tiktok-agency-creator-live-schedule/
https://www.yxtiktok.com/blog/tiktok-live-host-review-meeting-metrics/
https://www.yxtiktok.com/blog/tiktok-live-replay-review-method/
https://www.yxtiktok.com/blog/tiktok-agency-streamer-event-participation/
https://www.yxtiktok.com/blog/single-country-tiktok-agency-creator-pipeline/
https://www.yxtiktok.com/blog/tiktok-backstage-normal-vs-premium-invite/
https://www.yxtiktok.com/blog/early-signs-streamer-unhappy-with-agency/
https://www.yxtiktok.com/blog/tiktok-agency-recruitment-communication-tips/
https://www.yxtiktok.com/blog/tiktok-streamer-recruitment-communities/
https://www.yxtiktok.com/blog/tiktok-backstage-creator-management-relationship/
https://www.yxtiktok.com/blog/why-recheck-creator-eligibility-before-contact/
https://www.yxtiktok.com/blog/evaluate-tiktok-creator-with-limited-history/
https://www.yxtiktok.com/blog/tiktok-live-welcome-viewers-scripts/
https://www.yxtiktok.com/blog/tiktok-streamer-recruitment-channels-by-country/
https://www.yxtiktok.com/blog/new-tiktok-agency-creator-pool-and-roles/
https://www.yxtiktok.com/blog/prioritize-tiktok-creators-by-live-activity-eligibility/
https://www.yxtiktok.com/blog/tiktok-agency-recruitment-support-pitch/
https://www.yxtiktok.com/blog/tiktok-followers-vs-live-engagement-recruiting/
https://www.yxtiktok.com/blog/tiktok-agency-first-call-creator-needs-questions/
https://www.yxtiktok.com/blog/tiktok-singing-live-setlist-interaction/
https://www.yxtiktok.com/blog/tiktok-live-entertainment-gaming-shopping-creators/
https://www.yxtiktok.com/blog/tiktok-agency-first-recruitment-dm-scripts/
https://www.yxtiktok.com/blog/tiktok-live-match-pk-rules/
https://www.yxtiktok.com/blog/small-tiktok-live-agency-staff-tool-budget/
https://www.yxtiktok.com/blog/tiktok-new-streamer-training-curriculum/
https://www.yxtiktok.com/blog/select-tiktok-entertainment-live-creator-content-types/
https://www.yxtiktok.com/blog/find-tiktok-live-creators-streaming-now/
https://www.yxtiktok.com/blog/tiktok-creator-network-creator-relationship/
https://www.yxtiktok.com/blog/tiktok-agency-invite-not-found-troubleshooting/
https://www.yxtiktok.com/blog/small-tiktok-live-agency-priority-creator-types/
https://www.yxtiktok.com/blog/tiktok-agency-streamer-segmented-management/
https://www.yxtiktok.com/blog/tiktok-live-pk-interaction-planning/
https://www.yxtiktok.com/blog/tiktok-agency-cross-country-invite-region-rules/
https://www.yxtiktok.com/blog/multi-country-tiktok-agency-team-language-allocation/
https://www.yxtiktok.com/blog/top-vs-mid-tier-tiktok-live-creators-agency/
https://www.yxtiktok.com/blog/tiktok-live-creator-agency-contract-checklist/
https://www.yxtiktok.com/blog/tiktok-agency-streamer-manager-performance-kpis/
https://www.yxtiktok.com/blog/tiktok-agency-streamer-referral-recruitment/
https://www.yxtiktok.com/blog/tiktok-new-creator-coaching-payback-period/
https://www.yxtiktok.com/blog/reduce-duplicate-work-multiple-tiktok-agency-accounts/
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

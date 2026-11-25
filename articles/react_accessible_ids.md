# React useId 与无障碍组件实战：表单关联、提示信息与组件复用

可复用输入组件经常出现在多个弹窗或设置面板中。如果每个组件都使用相同的 id，label、错误提示及辅助技术关联就可能指向错误元素。React useId 能生成适合组件实例的标识。本文讨论如何把它用于可复用的表单与信息面板，并说明它与列表 key 的区别。

## 1. 明确需要稳定标识的地方

HTML id 主要用于文档内唯一标识及元素关联。表单标签、帮助说明和错误信息是典型场景；不是所有节点都需要 id。

```jsx
<div>
  <label htmlFor={inputId}>显示名称</label>
  <input id={inputId} name="displayName" />
</div>
```

## 2. 用 useId 生成组件实例标识

useId 应在组件顶层调用。用户复制两份字段组件时，每个实例都拥有独立标识，不需要全局自增变量。

```jsx
<div>
  <label htmlFor={id}>{label}</label>
  <input id={id} name={name} />
</div>
```

```tsx
import { useId } from 'react';
function Field({ label, name }: { label: string; name: string }) {
  const id = useId();
  return <label htmlFor={id}>{label}<input id={id} name={name} /></label>;
}
```

## 3. 使用同一前缀派生关联 ID

一个字段的帮助文字和错误信息可从同一个 useId 派生，避免为每个关联节点单独建立状态。

```jsx
<div>
  <label htmlFor={`${id}-input`}>邮箱</label>
  <input id={`${id}-input`} aria-describedby={`${id}-hint`} type="email" />
  <p id={`${id}-hint`}>用于接收业务通知</p>
</div>
```

## 4. 条件错误信息需要正确关联

只在错误元素存在时将其 ID 加入 aria-describedby。aria-invalid 描述字段是否无效，不应把所有初始必填字段立即标记为错误。

```jsx
<div>
  <input id={inputId} aria-invalid={Boolean(error)} aria-describedby={error ? `${hintId} ${errorId}` : hintId} />
  <p id={hintId}>请输入有效地址</p>
  {error && <p id={errorId} role="alert">{error}</p>}
</div>
```

## 5. 为面板标题建立可访问名称

有意义的区域可以通过 aria-labelledby 指向可见标题。不要把每个普通 div 都升级成 landmark，否则页面导航会变得拥挤。

```jsx
<div>
  <section aria-labelledby={titleId}>
    <h2 id={titleId}>通知设置</h2>
    <p>管理消息接收偏好</p>
  </section>
</div>
```

```tsx
const titleId = useId();
```

## 6. 折叠控件使用 aria-controls

展开按钮通过 aria-expanded 反映实际状态，aria-controls 指向内容区域。使用原生 button 获得键盘交互，不需要模拟点击行为。

```jsx
<div>
  <button aria-expanded={expanded} aria-controls={panelId} onClick={() => setExpanded(v => !v)}>高级设置</button>
  <section id={panelId} hidden={!expanded}>高级选项</section>
</div>
```

```tsx
const panelId = useId();
const [expanded, setExpanded] = useState(false);
```

## 7. useId 不是列表 key

列表 key 应来自业务数据稳定身份。数组重排时，数据身份必须跟随记录，而不是跟随当前渲染位置。不要在 map 循环中调用 Hook。

```jsx
<div>
  <ul>{fields.map(field => <li key={field.id}><Field label={field.label} name={field.name} /></li>)}</ul>
</div>
```

## 8. 避免客户端随机值造成水合差异

随机 ID 在服务端与客户端可能不同。useId 为相同组件树提供适合水合的标识；多个独立根需要使用匹配的 identifierPrefix 配置。

```jsx
<div>
  <section><Field label="联系人" name="contact" /><Field label="部门" name="department" /></section>
</div>
```

服务器与客户端组件树仍需一致；useId 不会修复条件渲染结构不同造成的水合问题。

## 9. 测试应面向用户可访问名称

测试不必断言 React 生成的具体 ID 字符串。通过 label 查询输入框、通过 role 查询按钮，并验证描述文本和错误状态。

```jsx
<div>
  <Field label="用户名称" name="username" />
</div>
```

```tsx
render(<Field label="用户名称" name="username" />);
expect(screen.getByLabelText('用户名称')).toBeInTheDocument();
```

## 10. 允许业务方显式传入标识

组件库可允许外部传入 id，用于与既有页面协议集成。但仍应无条件调用 useId，再选择传入值或生成值，避免违反 Hook 调用规则。

```jsx
<div>
  <label htmlFor={resolvedId}>{label}</label>
  <input id={resolvedId} name={name} />
</div>
```

```tsx
const generatedId = useId();
const resolvedId = providedId ?? generatedId;
```

## 总结

useId 的重点是正确建立可访问的元素关联，而不是给每个节点增加随机标识。配合语义化 HTML、稳定业务 key 和面向用户的测试，可复用组件才能在多实例、服务端渲染和复杂表单中保持可靠。

## 本站 URL 数据样本

以下为本站公开页面地址样本，与上文技术示例及下方热搜无关。

```text
https://www.yxtiktok.com/blog/tiktok-agency-creator-data-privacy-access-control/
https://www.yxtiktok.com/blog/tiktok-agency-handle-creator-live-violation/
https://www.yxtiktok.com/blog/tiktok-live-replay-review-method/
https://www.yxtiktok.com/blog/tiktok-live-agency-differentiated-creator-offer/
https://www.yxtiktok.com/blog/tiktok-host-recruitment-cost-per-signed-creator/
https://www.yxtiktok.com/blog/small-tiktok-live-agency-essential-team-roles/
https://www.yxtiktok.com/blog/select-tiktok-entertainment-live-creator-content-types/
https://www.yxtiktok.com/blog/tiktok-creator-discovery-ranking-live-profile-workflow/
https://www.yxtiktok.com/blog/tiktok-agency-recruitment-promise-contract-check/
https://www.yxtiktok.com/blog/single-country-tiktok-agency-creator-pipeline/
https://www.yxtiktok.com/blog/tiktok-dm-requests-privacy-creator-contact/
https://www.yxtiktok.com/blog/tiktok-live-ranking-find-creators/
https://www.yxtiktok.com/blog/record-tiktok-creator-invite-check-history/
https://www.yxtiktok.com/blog/find-tiktok-creators-for-small-agency-development/
https://www.yxtiktok.com/blog/reassess-long-unfollowed-tiktok-creator-leads/
https://www.yxtiktok.com/blog/tiktok-streamer-live-schedule-reminder-scripts/
https://www.yxtiktok.com/blog/tiktok-agency-cross-country-invite-region-rules/
https://www.yxtiktok.com/blog/creator-not-interested-agency-recruitment-status/
https://www.yxtiktok.com/blog/filter-overseas-tiktok-creators-by-language/
https://www.yxtiktok.com/blog/find-tiktok-live-creators-streaming-now/
https://www.yxtiktok.com/blog/what-is-tiktok-live-backstage-agency-functions/
https://www.yxtiktok.com/blog/tiktok-agency-recruiter-offboarding-leads-handover/
https://www.yxtiktok.com/blog/tiktok-live-welcome-viewers-scripts/
https://www.yxtiktok.com/blog/how-tiktok-agency-prove-identity-to-creator/
https://www.yxtiktok.com/blog/early-signs-streamer-unhappy-with-agency/
https://www.yxtiktok.com/blog/clean-invalid-stale-tiktok-creator-lead-records/
https://www.yxtiktok.com/blog/assess-tiktok-creator-live-positioning-from-profile/
https://www.yxtiktok.com/blog/tiktok-live-league-tiers-creator-discovery/
https://www.yxtiktok.com/blog/tiktok-agency-first-contact-message-to-creator/
https://www.yxtiktok.com/blog/reduce-duplicate-work-multiple-tiktok-agency-accounts/
https://www.yxtiktok.com/blog/small-tiktok-live-agency-staff-tool-budget/
https://www.yxtiktok.com/blog/tiktok-part-time-vs-full-time-streamer-management/
https://www.yxtiktok.com/blog/why-tiktok-creator-invite-eligibility-changes/
https://www.yxtiktok.com/blog/tiktok-agency-daily-recruitment-plan/
https://www.yxtiktok.com/blog/assess-third-party-tiktok-live-ranking-coverage/
https://www.yxtiktok.com/blog/schedule-cross-timezone-tiktok-creator-recruiting-team/
https://www.yxtiktok.com/blog/why-high-earning-tiktok-creators-may-not-fit-agency/
https://www.yxtiktok.com/blog/tiktok-streamer-recruitment-communities/
https://www.yxtiktok.com/blog/tiktok-agency-streamer-manager-performance-kpis/
https://www.yxtiktok.com/blog/daily-tiktok-agency-recruitment-report-kpis/
https://www.yxtiktok.com/blog/tiktok-streamer-recruitment-channels-by-country/
https://www.yxtiktok.com/blog/tiktok-live-host-review-meeting-metrics/
https://www.yxtiktok.com/blog/top-vs-mid-tier-tiktok-live-creators-agency/
https://www.yxtiktok.com/blog/tiktok-live-creator-agency-contract-checklist/
https://www.yxtiktok.com/blog/prioritize-tiktok-creators-by-live-activity-eligibility/
https://www.yxtiktok.com/blog/tiktok-agency-first-recruitment-dm-scripts/
https://www.yxtiktok.com/blog/tiktok-creator-network-creator-relationship/
https://www.yxtiktok.com/blog/tiktok-live-creator-network-violation-consequences/
https://www.yxtiktok.com/blog/tiktok-entertainment-live-opening-retention/
https://www.yxtiktok.com/blog/tiktok-streamer-flexible-live-scheduling/
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

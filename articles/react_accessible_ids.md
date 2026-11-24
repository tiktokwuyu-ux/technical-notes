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

## 热搜标题与本站资料

以下热搜标题仅作当日资讯记录；本站链接为独立资料，并非该新闻的报道。

- 热搜：加州给548名死刑犯减刑 特朗普震怒；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-creator-network-violation-consequences/>
- 热搜：伪人AI广告 已经成了一种新的城市病；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-creator-data-privacy-access-control/>
- 热搜：华为Mate 90 Air售价4999元起；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-one-on-one-streamer-support/>
- 热搜：中考要不要考历史 教材主编征集意见；本站资料：<https://www.yxtiktok.com/blog/small-tiktok-live-agency-essential-team-roles/>
- 热搜：车企老板娘女装卖了30亿；本站资料：<https://www.yxtiktok.com/blog/why-tiktok-agency-deduplicate-creator-uid/>
- 热搜：换脸还没消停 “AI融脸”又来了；本站资料：<https://www.yxtiktok.com/blog/tiktok-country-ranking-find-live-creators/>
- 热搜：天还没亮 学生就集体站在操场晨读；本站资料：<https://www.yxtiktok.com/blog/tiktok-creator-discovery-to-joining-sop/>
- 热搜：特朗普连发30多张照片“回味”深圳；本站资料：<https://www.yxtiktok.com/blog/tiktok-creator-network-multi-account-invite-risk/>
- 热搜：松鼠妈妈六次示范教小松鼠跳树成功；本站资料：<https://www.yxtiktok.com/blog/assess-tiktok-creator-live-positioning-from-profile/>
- 热搜：19楼落地窗底部装无限位外凸窗；本站资料：<https://www.yxtiktok.com/blog/tiktok-coins-gifts-diamonds-difference/>
- 热搜：Mate X8 华为史上最强大折叠；本站资料：<https://www.yxtiktok.com/blog/tiktok-creator-weeks-performance-decline/>
- 热搜：小米Q3净利润48.84亿 同比减少56.8%；本站资料：<https://www.yxtiktok.com/blog/agency-invite-sent-creator-not-accepted-follow-up/>
- 热搜：北京天空现“超级月亮”；本站资料：<https://www.yxtiktok.com/blog/tiktok-host-recruitment-cost-per-signed-creator/>
- 热搜：“民主党向黄仁勋和英伟达宣战”；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-agency-dm-live-room-recruitment-risks/>
- 热搜：飞机与“超级月亮”浪漫同框；本站资料：<https://www.yxtiktok.com/blog/why-high-earning-tiktok-creators-may-not-fit-agency/>
- 热搜：香奈儿包日租仅需25元？押金要一万六；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-agency-differentiated-creator-offer/>
- 热搜：女子登山上班往返约6小时已瘦12斤；本站资料：<https://www.yxtiktok.com/blog/tiktok-new-creator-coaching-payback-period/>
- 热搜：睡眠大赛4人躺赢平分6000元；本站资料：<https://www.yxtiktok.com/blog/single-country-tiktok-agency-creator-pipeline/>
- 热搜：新娘提1341次分手新郎回1342次不行；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-recruiter-offboarding-leads-handover/>
- 热搜：课堂上的“嘉豪、嘉欣” 已不敢举手；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-host-review-meeting-metrics/>
- 热搜：黄奕用AI视频还原过去三十年人生；本站资料：<https://www.yxtiktok.com/blog/new-tiktok-agency-first-100-creator-leads-plan/>
- 热搜：几个姑娘在-25℃的黑龙江清凉上阵；本站资料：<https://www.yxtiktok.com/blog/find-tiktok-creators-for-small-agency-development/>
- 热搜：大众拟裁员10万人 工会威胁罢工；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-streamer-maintenance-schedule/>
- 热搜：东北外卖15分钟冻成冰；本站资料：<https://www.yxtiktok.com/blog/tiktok-streamer-recruitment-communities/>
- 热搜：国产可口可乐不再标生产日期；本站资料：<https://www.yxtiktok.com/blog/evaluate-tiktok-creator-with-limited-history/>
- 热搜：外交部回应“黄仁勋重回英伟达”；本站资料：<https://www.yxtiktok.com/blog/how-to-check-tiktok-creator-agency-invite-eligibility/>
- 热搜：中国城市 重排全球座次；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-fan-club-features/>
- 热搜：央行将开展8000亿元MLF操作；本站资料：<https://www.yxtiktok.com/blog/select-tiktok-entertainment-live-creator-content-types/>
- 热搜：大爷抵脖子不放后小伙倒地 地铁回应；本站资料：<https://www.yxtiktok.com/blog/assess-third-party-tiktok-live-ranking-coverage/>
- 热搜：华为首款小折叠电脑发布；本站资料：<https://www.yxtiktok.com/blog/tiktok-short-video-creator-recruitment/>
- 热搜：重庆男孩被坠楼男子砸中送医；本站资料：<https://www.yxtiktok.com/blog/tiktok-streamer-rejects-agency-advice/>
- 热搜：动辄千元起步的户外徒步鞋火了；本站资料：<https://www.yxtiktok.com/blog/tiktok-entertainment-live-opening-retention/>
- 热搜：特朗普致电黄仁勋：答应我一件事；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-first-contact-message-to-creator/>
- 热搜：余承东现场提词器曝光；本站资料：<https://www.yxtiktok.com/blog/why-invitable-tiktok-creator-may-not-join/>
- 热搜：冯远征笑称“考北京人艺其实不难”；本站资料：<https://www.yxtiktok.com/blog/which-live-metrics-tiktok-agency-should-evaluate/>
- 热搜：上海市委书记陈吉宁被查；本站资料：<https://www.yxtiktok.com/blog/tiktok-creator-pre-signing-info-minimization/>
- 热搜：旺仔牛奶更不好卖了；本站资料：<https://www.yxtiktok.com/blog/avoid-spam-repeated-dm-tiktok-agency-recruitment/>
- 热搜：泰警方确认女子异地遇害被肢解抛尸；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-first-call-creator-needs-questions/>
- 热搜：太平洋暖得“失控”了；本站资料：<https://www.yxtiktok.com/blog/three-tier-tiktok-creator-lead-priority-system/>
- 热搜：培养更多高技能人才和大国工匠；本站资料：<https://www.yxtiktok.com/blog/tiktok-recruitment-channel-conversion-analysis/>
- 热搜：田秉毅任国家羽毛球队主教练；本站资料：<https://www.yxtiktok.com/blog/why-recheck-creator-eligibility-before-contact/>
- 热搜：分享你拍的超级月亮；本站资料：<https://www.yxtiktok.com/blog/record-tiktok-creator-invite-check-history/>
- 热搜：中国人登月又近一步；本站资料：<https://www.yxtiktok.com/blog/tiktok-dance-live-content-interaction-plan/>
- 热搜：Anthropic上市5天即破发；本站资料：<https://www.yxtiktok.com/blog/tiktok-chat-streamer-improvisation-training/>
- 热搜：注水长剧的“狼”来了；本站资料：<https://www.yxtiktok.com/blog/unified-tiktok-creator-recruitment-records/>
- 热搜：受贿1.13亿余元！吴胜华被判无期；本站资料：<https://www.yxtiktok.com/blog/tiktok-pk-gifts-live-data-review/>
- 热搜：黄磊真空围裙；本站资料：<https://www.yxtiktok.com/blog/talk-to-new-tiktok-live-creator-needs/>
- 热搜：文吉儿从小就活跃在各大荧幕；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-streamer-feedback-follow-up/>
- 热搜：年产1500万根的“棍子” 卖出10亿元；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-many-leads-low-signup-rate/>
- 热搜：一校一主题 消谣育青年；本站资料：<https://www.yxtiktok.com/blog/prioritize-tiktok-creators-by-live-activity-eligibility/>

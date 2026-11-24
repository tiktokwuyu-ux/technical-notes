# Next.js Route Handlers 实战：接口边界、参数校验与统一错误响应

在 Next.js 项目中，前端有时需要面向浏览器或外部客户端提供 HTTP 接口。Route Handlers 使用 Web 标准 Request 和 Response 对象处理请求。本文以工作记录接口为例，讨论路由定义、请求校验、权限、错误契约和可测试性。

## 1. 明确 HTTP 接口的职责

Route Handler 适合作为外部请求边界。内部数据访问函数不必为了在服务器使用而绕行同一应用的 HTTP 接口。

```jsx
<div>
  <section><h2>工作记录</h2><p>通过统一接口读取和创建记录</p></section>
</div>
```

```text
app/api/records/route.ts
lib/records/service.ts
lib/auth/session.ts
```

## 2. 定义 GET 请求

GET 接口读取数据时仍需验证会话。返回字段应经过业务层筛选，不应直接序列化数据库中的全部列。

```jsx
<div>
  <ul>{records.map(item => <li key={item.id}>{item.title}</li>)}</ul>
</div>
```

```ts
export async function GET(request: Request) {
  const user = await authenticate(request);
  if (!user) return Response.json({ code: 'UNAUTHORIZED' }, { status: 401 });
  const records = await listVisibleRecords(user.id);
  return Response.json({ items: records });
}
```

## 3. 解析并验证查询参数

URLSearchParams 返回字符串或 null，不能直接作为数据库分页数量使用。限定整数范围，避免异常输入扩大查询负载。

```jsx
<div>
  <label>每页数量 <select value={limit} onChange={changeLimit}><option value="20">20</option><option value="50">50</option></select></label>
</div>
```

```ts
function parseLimit(url: URL) {
  const raw = url.searchParams.get('limit') ?? '20';
  const limit = Number(raw);
  if (!/^\d+$/.test(raw) || !Number.isInteger(limit) || limit < 1 || limit > 100) return null;
  return limit;
}
```

## 4. 正确处理 POST 请求体

request.json 可能因为无效 JSON 抛出异常。应先检查请求类型，再验证解析后的未知数据，不要直接断言为业务类型。

```jsx
<div>
  <form onSubmit={createRecord}><label>标题 <input name="title" required maxLength={120} /></label><button>创建</button></form>
</div>
```

```ts
let payload: unknown;
try { payload = await request.json(); }
catch { return Response.json({ code: 'INVALID_JSON' }, { status: 400 }); }
if (!isValidRecordInput(payload)) return Response.json({ code: 'INVALID_INPUT' }, { status: 422 });
```

## 5. 区分认证与资源授权

已登录用户不一定有权修改指定记录。资源查询必须包含用户或租户范围，权限判断不能依赖请求体中的 ownerId。

```jsx
<div>
  <section><p>当前工作区：{workspaceName}</p><button disabled={!canCreate}>新增记录</button></section>
</div>
```

服务端使用认证会话确定用户身份，再由权限服务校验资源范围。Cookie 认证的写入接口还应按部署环境处理 CSRF 风险。

## 6. 建立稳定的错误响应契约

前端需要稳定错误码而不是数据库异常字符串。将预期业务错误映射到合适 HTTP 状态，未知异常记录在服务端并返回一般性错误。

```jsx
<div>
  {error && <p role="alert">{errorMessage(error.code)}</p>}
</div>
```

```ts
function conflict() {
  return Response.json({ code: 'RECORD_CONFLICT', message: '记录状态已变化' }, { status: 409 });
}
```

## 7. 为写入操作设计幂等边界

重复点击和网络重试可能产生多次请求。对于创建记录，可以使用服务端唯一约束或经校验的请求键，确保同一业务意图不重复创建。

```jsx
<div>
  <button type="submit" disabled={submitting}>{submitting ? '提交中' : '创建记录'}</button>
</div>
```

前端按钮禁用只是交互保护。服务器需在事务内验证幂等键与用户、请求内容的绑定，并明确冲突时返回的结果。

## 8. 明确响应缓存策略

含有用户数据的读取接口通常不应进入公共缓存。框架版本、路由配置和代理层都可能影响缓存，因此应明确声明所需行为。

```jsx
<div>
  <p>数据更新时间：{updatedAt}</p><button onClick={reload}>刷新</button>
</div>
```

```ts
return Response.json({ items }, { headers: { 'Cache-Control': 'private, no-store' } });
```

## 9. 让前端区分网络与业务错误

fetch 在收到 HTTP 错误状态时通常仍会返回 Response。前端应根据状态和约定的响应体进行错误处理，避免把所有失败都描述为网络断开。

```jsx
<div>
  <section role="status">{loading ? '正在读取' : '读取结束'}</section>
</div>
```

```ts
const response = await fetch('/api/records');
const result = await response.json();
if (!response.ok) throw new ApiError(response.status, result.code);
```

## 10. 以 HTTP 边界进行验收

验证无凭据、权限不足、无效 JSON、非法分页、重复提交和正常数据。还要检查接口错误不会暴露 SQL、路径或内部凭据。

```jsx
<div>
  <dl><dt>请求状态</dt><dd>{status}</dd><dt>错误码</dt><dd>{code ?? '无'}</dd></dl>
</div>
```

测试时可以直接构造 Request 调用 handler，但还需要部署环境的集成测试验证代理、Cookie 和运行时配置。

## 总结

Route Handlers 的重点是清晰的 HTTP 契约。将身份校验、参数验证、数据访问和错误转换分离，并为写入操作建立服务端幂等保障，能够使 Next.js 接口在多种客户端场景下保持稳定。

## 热搜标题与本站资料

以下热搜标题仅作当日资讯记录；本站链接为独立资料，并非该新闻的报道。

- 热搜：大众拟裁员10万人 工会威胁罢工；本站资料：<https://www.yxtiktok.com/blog/calculate-tiktok-creator-recruitment-conversion-rate/>
- 热搜：冯远征笑称“考北京人艺其实不难”；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-league-tiers-creator-discovery/>
- 热搜：东北外卖15分钟冻成冰；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-cross-country-invite-region-rules/>
- 热搜：女子登山上班往返约6小时已瘦12斤；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-multilingual-recruitment-script-library/>
- 热搜：重庆男孩被坠楼男子砸中送医；本站资料：<https://www.yxtiktok.com/blog/can-individual-apply-tiktok-live-creator-network/>
- 热搜：大爷抵脖子不放后小伙倒地 地铁回应；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-welcome-viewers-scripts/>
- 热搜：换脸还没消停 “AI融脸”又来了；本站资料：<https://www.yxtiktok.com/blog/tiktok-creator-weeks-performance-decline/>
- 热搜：培养更多高技能人才和大国工匠；本站资料：<https://www.yxtiktok.com/blog/reassess-long-unfollowed-tiktok-creator-leads/>
- 热搜：Mate X8 华为史上最强大折叠；本站资料：<https://www.yxtiktok.com/blog/why-recheck-creator-eligibility-before-contact/>
- 热搜：加州给548名死刑犯减刑 特朗普震怒；本站资料：<https://www.yxtiktok.com/blog/tiktok-short-video-creator-recruitment/>
- 热搜：天还没亮 学生就集体站在操场晨读；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-awkward-silence-topic-switching/>
- 热搜：几个姑娘在-25℃的黑龙江清凉上阵；本站资料：<https://www.yxtiktok.com/tutorials/tiktok-live-ranklist-tutorial/>
- 热搜：特朗普连发30多张照片“回味”深圳；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-cohost-chatting-tips/>
- 热搜：央行将开展8000亿元MLF操作；本站资料：<https://www.yxtiktok.com/blog/compare-tiktok-live-rankings-across-countries/>
- 热搜：动辄千元起步的户外徒步鞋火了；本站资料：<https://www.yxtiktok.com/blog/evaluate-tiktok-creator-with-limited-history/>
- 热搜：分享你拍的超级月亮；本站资料：<https://www.yxtiktok.com/blog/tiktok-part-time-vs-full-time-streamer-management/>
- 热搜：国产可口可乐不再标生产日期；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-streamer-event-participation/>
- 热搜：旺仔牛奶更不好卖了；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-creator-live-schedule/>
- 热搜：中国人登月又近一步；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-pk-interaction-planning/>
- 热搜：太平洋暖得“失控”了；本站资料：<https://www.yxtiktok.com/blog/reduce-duplicate-work-multiple-tiktok-agency-accounts/>
- 热搜：车企老板娘女装卖了30亿；本站资料：<https://www.yxtiktok.com/blog/find-tiktok-creators-for-small-agency-development/>
- 热搜：田秉毅任国家羽毛球队主教练；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-streamer-feedback-follow-up/>
- 热搜：黄磊真空围裙；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-first-contact-message-to-creator/>
- 热搜：新娘提1341次分手新郎回1342次不行；本站资料：<https://www.yxtiktok.com/blog/agency-owner-recruiters-software-role-permissions/>
- 热搜：飞机与“超级月亮”浪漫同框；本站资料：<https://www.yxtiktok.com/blog/tiktok-singing-live-setlist-interaction/>
- 热搜：小米Q3净利润48.84亿 同比减少56.8%；本站资料：<https://www.yxtiktok.com/blog/why-tiktok-agency-deduplicate-creator-uid/>
- 热搜：华为Mate 90 Air售价4999元起；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-events-vs-campaigns/>
- 热搜：香奈儿包日租仅需25元？押金要一万六；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-creator-incentive-communication/>
- 热搜：“民主党向黄仁勋和英伟达宣战”；本站资料：<https://www.yxtiktok.com/blog/tiktok-coins-gifts-diamonds-difference/>
- 热搜：北京天空现“超级月亮”；本站资料：<https://www.yxtiktok.com/blog/select-tiktok-entertainment-live-creator-content-types/>
- 热搜：注水长剧的“狼”来了；本站资料：<https://www.yxtiktok.com/blog/three-tier-tiktok-creator-lead-priority-system/>
- 热搜：外交部回应“黄仁勋重回英伟达”；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-many-leads-low-signup-rate/>
- 热搜：余承东现场提词器曝光；本站资料：<https://www.yxtiktok.com/blog/which-live-metrics-tiktok-agency-should-evaluate/>
- 热搜：黄奕用AI视频还原过去三十年人生；本站资料：<https://www.yxtiktok.com/blog/why-tiktok-live-ranking-differs-from-real-performance/>
- 热搜：19楼落地窗底部装无限位外凸窗；本站资料：<https://www.yxtiktok.com/blog/tiktok-creator-discovery-to-joining-sop/>
- 热搜：特朗普致电黄仁勋：答应我一件事；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-recruitment-funnel-from-lead-to-join/>
- 热搜：受贿1.13亿余元！吴胜华被判无期；本站资料：<https://www.yxtiktok.com/blog/why-tiktok-live-ranking-platforms-disagree/>
- 热搜：一校一主题 消谣育青年；本站资料：<https://www.yxtiktok.com/blog/prioritize-tiktok-creators-by-live-activity-eligibility/>
- 热搜：上海市委书记陈吉宁被查；本站资料：<https://www.yxtiktok.com/blog/find-tiktok-live-creators-streaming-now/>
- 热搜：伪人AI广告 已经成了一种新的城市病；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-creator-network-vs-mcn-shop/>
- 热搜：松鼠妈妈六次示范教小松鼠跳树成功；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-recruiter-performance-kpis/>
- 热搜：Anthropic上市5天即破发；本站资料：<https://www.yxtiktok.com/blog/high-gift-high-growth-stable-tiktok-creator-types/>
- 热搜：华为首款小折叠电脑发布；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-recruiter-offboarding-leads-handover/>
- 热搜：中考要不要考历史 教材主编征集意见；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-creator-tiered-management/>
- 热搜：文吉儿从小就活跃在各大荧幕；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-low-live-hours-diagnosis/>
- 热搜：年产1500万根的“棍子” 卖出10亿元；本站资料：<https://www.yxtiktok.com/blog/tiktok-recruitment-channel-conversion-analysis/>
- 热搜：睡眠大赛4人躺赢平分6000元；本站资料：<https://www.yxtiktok.com/blog/avoid-spam-repeated-dm-tiktok-agency-recruitment/>
- 热搜：课堂上的“嘉豪、嘉欣” 已不敢举手；本站资料：<https://www.yxtiktok.com/blog/tiktok-host-recruitment-cost-per-signed-creator/>
- 热搜：中国城市 重排全球座次；本站资料：<https://www.yxtiktok.com/blog/manual-vs-software-tiktok-streamer-recruitment/>
- 热搜：泰警方确认女子异地遇害被肢解抛尸；本站资料：<https://www.yxtiktok.com/blog/why-agencies-should-not-rely-only-on-tiktok-top-rankings/>

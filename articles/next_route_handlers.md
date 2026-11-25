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

## 本站 URL 数据样本

以下为本站公开页面地址样本，与上文技术示例及下方热搜无关。

```text
https://www.yxtiktok.com/blog/small-tiktok-live-agency-priority-creator-types/
https://www.yxtiktok.com/blog/avoid-duplicate-tiktok-creator-collection-and-followup/
https://www.yxtiktok.com/blog/remote-tiktok-agency-team-shared-creator-data/
https://www.yxtiktok.com/blog/small-tiktok-live-agency-essential-team-roles/
https://www.yxtiktok.com/blog/tiktok-agency-recruitment-channels/
https://www.yxtiktok.com/blog/tiktok-short-video-creator-recruitment/
https://www.yxtiktok.com/blog/creator-already-in-other-network-new-invite/
https://www.yxtiktok.com/blog/reassess-long-unfollowed-tiktok-creator-leads/
https://www.yxtiktok.com/blog/tiktok-agency-many-leads-low-signup-rate/
https://www.yxtiktok.com/blog/tiktok-creator-network-creator-relationship/
https://www.yxtiktok.com/blog/tiktok-creator-pre-signing-info-minimization/
https://www.yxtiktok.com/blog/new-tiktok-agency-first-100-creator-leads-plan/
https://www.yxtiktok.com/blog/tiktok-live-league-tiers-creator-discovery/
https://www.yxtiktok.com/blog/tiktok-coins-gifts-diamonds-difference/
https://www.yxtiktok.com/blog/growth-vs-established-tiktok-creators-recruitment-talk/
https://www.yxtiktok.com/blog/tiktok-new-streamer-pre-live-checklist/
https://www.yxtiktok.com/blog/tiktok-agency-streamer-onboarding-handoff/
https://www.yxtiktok.com/blog/benefits-of-joining-tiktok-live-agency/
https://www.yxtiktok.com/blog/tiktok-creator-pipeline-status-management/
https://www.yxtiktok.com/blog/unified-tiktok-creator-recruitment-records/
https://www.yxtiktok.com/blog/prevent-tiktok-agency-recruiter-duplicate-contact/
https://www.yxtiktok.com/blog/what-is-tiktok-live-backstage-agency-functions/
https://www.yxtiktok.com/blog/select-tiktok-entertainment-live-creator-content-types/
https://www.yxtiktok.com/blog/tiktok-live-match-pk-rules/
https://www.yxtiktok.com/blog/normal-invite-only-creator-agency-choice/
https://www.yxtiktok.com/blog/tiktok-agency-low-live-hours-diagnosis/
https://www.yxtiktok.com/blog/does-higher-tiktok-live-league-mean-better-recruit/
https://www.yxtiktok.com/blog/tiktok-agency-creator-tiered-management/
https://www.yxtiktok.com/blog/tiktok-streamer-flexible-live-scheduling/
https://www.yxtiktok.com/tutorials/tiktok-ios-non-jailbreak-dm-tutorial/
https://www.yxtiktok.com/blog/assess-tiktok-creator-live-positioning-from-profile/
https://www.yxtiktok.com/blog/tiktok-singing-live-setlist-interaction/
https://www.yxtiktok.com/blog/tiktok-creator-weeks-performance-decline/
https://www.yxtiktok.com/blog/reduce-duplicate-work-multiple-tiktok-agency-accounts/
https://www.yxtiktok.com/blog/tiktok-live-creator-agency-contract-checklist/
https://www.yxtiktok.com/blog/small-tiktok-live-agency-staff-tool-budget/
https://www.yxtiktok.com/blog/tiktok-agency-creator-incentive-communication/
https://www.yxtiktok.com/blog/tiktok-dm-requests-privacy-creator-contact/
https://www.yxtiktok.com/blog/tiktok-backstage-normal-vs-premium-invite/
https://www.yxtiktok.com/blog/agency-owner-recruiters-software-role-permissions/
https://www.yxtiktok.com/blog/tiktok-agency-recruitment-message-cooperation-terms/
https://www.yxtiktok.com/blog/tiktok-streamer-performance-decline-recovery-plan/
https://www.yxtiktok.com/blog/why-tiktok-agency-deduplicate-creator-uid/
https://www.yxtiktok.com/blog/high-gift-high-growth-stable-tiktok-creator-types/
https://www.yxtiktok.com/blog/which-live-metrics-tiktok-agency-should-evaluate/
https://www.yxtiktok.com/blog/why-high-earning-tiktok-creators-may-not-fit-agency/
https://www.yxtiktok.com/blog/tiktok-agency-recruitment-support-pitch/
https://www.yxtiktok.com/blog/tiktok-agency-turnover-revenue-gross-net-profit/
https://www.yxtiktok.com/blog/tiktok-recruitment-channel-conversion-analysis/
https://www.yxtiktok.com/blog/filter-overseas-tiktok-creators-by-language/
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

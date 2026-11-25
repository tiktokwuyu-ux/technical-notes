# Angular 依赖注入实战：服务作用域、InjectionToken 与可替换架构

大型 Angular 应用中，一个组件可能依赖 API、权限、配置和日志服务。如果组件直接创建这些对象，测试和环境切换会变得困难。Angular 依赖注入让实例创建与使用分离。本文围绕工作台通知服务介绍提供者、注入令牌及实例生命周期。

## 1. 把业务依赖与展示分离

通知组件只需要读取记录和发送已读操作，不需要知道具体是 HTTP 接口还是测试数据源。

```html
<div>
  <section><h2>通知中心</h2><p>显示工作区消息</p></section>
</div>
```

```ts
export interface NoticeRepository {
  list(): Promise<Notice[]>;
  markRead(id: string): Promise<void>;
}
```

## 2. 为接口创建运行时令牌

TypeScript interface 会在编译后消失，不能直接充当 Angular 注入键。使用 InjectionToken 为接口建立运行时身份。

```html
<div>
  <p role="status">{{ status }}</p>
</div>
```

```ts
import { InjectionToken } from '@angular/core';
export const NOTICE_REPOSITORY = new InjectionToken<NoticeRepository>('NoticeRepository');
```

## 3. 通过 useClass 配置实现

提供者可以把令牌映射到具体实现。组件依赖抽象契约，实际实例由注入器管理。

```html
<div>
  <ul>@for (notice of notices; track notice.id) { <li>{{ notice.title }}</li> }</ul>
</div>
```

```ts
@Injectable()
export class HttpNoticeRepository implements NoticeRepository {
  // 在此实现 list 和 markRead，封装 HTTP 调用。
}
// 应用 providers 中配置：
{ provide: NOTICE_REPOSITORY, useClass: HttpNoticeRepository }
```

## 4. 在组件中使用 inject

inject 可以在 Angular 创建对象时的注入上下文中使用，例如字段初始化器。它不能任意放在之后触发的普通回调中调用。

```html
<div>
  <button type="button" (click)="refresh()">刷新通知</button>
</div>
```

```ts
private readonly repository = inject(NOTICE_REPOSITORY);
async refresh() { this.notices = await this.repository.list(); }
```

## 5. 选择应用级或组件级作用域

应用级服务适合共享会话和通用能力。组件级 providers 可以为每个独立编辑面板创建单独实例，并随该组件的生命周期结束。

```html
<div>
  <section><app-notice-panel /><app-notice-panel /></section>
</div>
```

```ts
@Component({ selector: 'app-notice-panel', providers: [NoticeSelectionService], templateUrl: './notice-panel.html' })
export class NoticePanelComponent {}
```

## 6. 通过 useValue 注入只读配置

接口地址或功能策略可以作为明确的配置对象提供，不必在每个组件内部重复写字符串。注入并不会自动使对象不可变。

```html
<div>
  <p>当前通知刷新策略：{{ config.refreshLabel }}</p>
</div>
```

```ts
export const NOTICE_CONFIG = new InjectionToken<Readonly<NoticeConfig>>('NoticeConfig');
const provider = { provide: NOTICE_CONFIG, useValue: Object.freeze(environment.noticeConfig) };
```

## 7. 利用 useFactory 组合依赖

工厂提供者适合根据注入的配置创建实例。保持工厂简单，不要把昂贵初始化或不可控网络写入藏在实例构造阶段。

```html
<div>
  <section><h3>消息来源</h3><p>{{ sourceLabel }}</p></section>
</div>
```

```ts
const provider = {
  provide: NOTICE_REPOSITORY,
  useFactory: () => createNoticeRepository(inject(NOTICE_CONFIG), inject(HttpClient))
};
```

## 8. 使用 useExisting 避免重复实例

如果两个令牌表示同一服务实例，可以使用 useExisting 建立别名。useClass 则可能创建另一个实例，从而导致状态重复。

```html
<div>
  <output>{{ unreadCount }}</output>
</div>
```

```ts
const providers = [HttpNoticeRepository, { provide: NOTICE_REPOSITORY, useExisting: HttpNoticeRepository }];
```

## 9. 管理服务内部订阅生命周期

组件作用域服务可能拥有 Observable 订阅或外部事件。可以利用 DestroyRef 或 takeUntilDestroyed，使资源随对应注入上下文清理。

```html
<div>
  <p aria-live="polite">{{ connectionStatus }}</p>
</div>
```

```ts
private readonly destroyRef = inject(DestroyRef);
connect() {
  this.events.pipe(takeUntilDestroyed(this.destroyRef)).subscribe(event => this.apply(event));
}
```

## 10. 利用替代提供者测试业务

测试不需要启动真实网络服务。将接口令牌替换为 fake 实现，验证组件如何处理正常结果、空数据和失败。

```html
<div>
  <section><p>{{ notices.length }} 条通知</p></section>
</div>
```

```ts
TestBed.configureTestingModule({ providers: [{ provide: NOTICE_REPOSITORY, useValue: fakeRepository }] });
expect(TestBed.inject(NOTICE_REPOSITORY)).toBe(fakeRepository);
```

## 总结

依赖注入的主要价值是明确实例所有权和可替换边界。使用类型化令牌、合理的提供者作用域及生命周期清理，可以让 Angular 服务在复杂项目中保持一致且便于测试。

## 本站 URL 数据样本

以下为本站公开页面地址样本，与上文技术示例及下方热搜无关。

```text
https://www.yxtiktok.com/blog/tiktok-agency-recruiter-performance-kpis/
https://www.yxtiktok.com/blog/high-gift-high-growth-stable-tiktok-creator-types/
https://www.yxtiktok.com/blog/tiktok-creator-network-cross-region-compliance/
https://www.yxtiktok.com/blog/tiktok-agency-handle-creator-live-violation/
https://www.yxtiktok.com/blog/evaluate-tiktok-creator-with-limited-history/
https://www.yxtiktok.com/blog/agency-owner-recruiters-software-role-permissions/
https://www.yxtiktok.com/blog/growth-vs-established-tiktok-creators-recruitment-talk/
https://www.yxtiktok.com/blog/tiktok-followers-vs-live-engagement-recruiting/
https://www.yxtiktok.com/blog/creator-not-interested-agency-recruitment-status/
https://www.yxtiktok.com/blog/tiktok-live-pk-interaction-planning/
https://www.yxtiktok.com/blog/why-invitable-tiktok-creator-may-not-join/
https://www.yxtiktok.com/blog/tiktok-creator-discovery-ranking-live-profile-workflow/
https://www.yxtiktok.com/blog/tiktok-creator-weeks-performance-decline/
https://www.yxtiktok.com/blog/tiktok-streamer-performance-decline-recovery-plan/
https://www.yxtiktok.com/blog/small-tiktok-live-agency-priority-creator-types/
https://www.yxtiktok.com/blog/tiktok-agency-recruitment-support-pitch/
https://www.yxtiktok.com/blog/how-tiktok-agency-prove-identity-to-creator/
https://www.yxtiktok.com/blog/tiktok-new-vs-experienced-streamer-outreach/
https://www.yxtiktok.com/blog/tiktok-agency-streamer-event-participation/
https://www.yxtiktok.com/blog/prevent-tiktok-agency-recruiter-duplicate-contact/
https://www.yxtiktok.com/blog/tiktok-live-follow-cta-without-pressure/
https://www.yxtiktok.com/blog/daily-tiktok-agency-recruitment-report-kpis/
https://www.yxtiktok.com/blog/tiktok-singing-live-setlist-interaction/
https://www.yxtiktok.com/blog/tiktok-new-creator-coaching-payback-period/
https://www.yxtiktok.com/blog/talk-to-new-tiktok-live-creator-needs/
https://www.yxtiktok.com/blog/tiktok-creator-network-multi-account-invite-risk/
https://www.yxtiktok.com/blog/tiktok-agency-streamer-onboarding-handoff/
https://www.yxtiktok.com/blog/avoid-spam-repeated-dm-tiktok-agency-recruitment/
https://www.yxtiktok.com/blog/tiktok-live-cohost-chatting-tips/
https://www.yxtiktok.com/blog/cost-structure-recruit-established-vs-new-live-creators/
https://www.yxtiktok.com/blog/tiktok-live-agency-revenue-sources/
https://www.yxtiktok.com/blog/tiktok-dance-live-content-interaction-plan/
https://www.yxtiktok.com/blog/tiktok-agency-creator-live-schedule/
https://www.yxtiktok.com/blog/tiktok-dm-requests-privacy-creator-contact/
https://www.yxtiktok.com/blog/tiktok-entertainment-live-opening-retention/
https://www.yxtiktok.com/blog/tiktok-agency-recruiter-offboarding-leads-handover/
https://www.yxtiktok.com/blog/can-individual-apply-tiktok-live-creator-network/
https://www.yxtiktok.com/blog/clean-invalid-stale-tiktok-creator-lead-records/
https://www.yxtiktok.com/blog/small-tiktok-live-agency-essential-team-roles/
https://www.yxtiktok.com/blog/remote-tiktok-agency-team-shared-creator-data/
https://www.yxtiktok.com/blog/tiktok-backstage-creator-management-relationship/
https://www.yxtiktok.com/blog/tiktok-live-host-review-meeting-metrics/
https://www.yxtiktok.com/blog/tiktok-streamer-recruitment-channels-by-country/
https://www.yxtiktok.com/blog/screen-tiktok-creator-public-live-content-policy-risk/
https://www.yxtiktok.com/blog/three-tier-tiktok-creator-lead-priority-system/
https://www.yxtiktok.com/blog/tiktok-backstage-normal-vs-premium-invite/
https://www.yxtiktok.com/blog/solo-vs-team-tiktok-agency-recruitment-workflow/
https://www.yxtiktok.com/blog/how-to-check-tiktok-creator-agency-invite-eligibility/
https://www.yxtiktok.com/blog/reassess-long-unfollowed-tiktok-creator-leads/
https://www.yxtiktok.com/blog/tiktok-agency-recruitment-funnel-from-lead-to-join/
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

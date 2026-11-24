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

## 热搜标题与本站资料

以下热搜标题仅作当日资讯记录；本站链接为独立资料，并非该新闻的报道。

- 热搜：松鼠妈妈六次示范教小松鼠跳树成功；本站资料：<https://www.yxtiktok.com/blog/growth-vs-established-tiktok-creators-recruitment-talk/>
- 热搜：伪人AI广告 已经成了一种新的城市病；本站资料：<https://www.yxtiktok.com/blog/prioritize-tiktok-creators-by-live-activity-eligibility/>
- 热搜：分享你拍的超级月亮；本站资料：<https://www.yxtiktok.com/blog/cost-structure-recruit-established-vs-new-live-creators/>
- 热搜：注水长剧的“狼”来了；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-streamer-feedback-follow-up/>
- 热搜：余承东现场提词器曝光；本站资料：<https://www.yxtiktok.com/blog/normal-invite-only-creator-agency-choice/>
- 热搜：泰警方确认女子异地遇害被肢解抛尸；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-creator-agency-contract-checklist/>
- 热搜：年产1500万根的“棍子” 卖出10亿元；本站资料：<https://www.yxtiktok.com/blog/tiktok-dance-live-content-interaction-plan/>
- 热搜：华为首款小折叠电脑发布；本站资料：<https://www.yxtiktok.com/tutorials/tiktok-ios-non-jailbreak-dm-tutorial/>
- 热搜：东北外卖15分钟冻成冰；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-agency-differentiated-creator-offer/>
- 热搜：太平洋暖得“失控”了；本站资料：<https://www.yxtiktok.com/blog/talk-to-new-tiktok-live-creator-needs/>
- 热搜：新娘提1341次分手新郎回1342次不行；本站资料：<https://www.yxtiktok.com/blog/tiktok-new-vs-experienced-streamer-outreach/>
- 热搜：飞机与“超级月亮”浪漫同框；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-streamer-event-participation/>
- 热搜：加州给548名死刑犯减刑 特朗普震怒；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-recruiter-offboarding-leads-handover/>
- 热搜：华为Mate 90 Air售价4999元起；本站资料：<https://www.yxtiktok.com/blog/tiktok-streamer-not-responding-agency/>
- 热搜：文吉儿从小就活跃在各大荧幕；本站资料：<https://www.yxtiktok.com/blog/tiktok-streamer-live-schedule-reminder-scripts/>
- 热搜：国产可口可乐不再标生产日期；本站资料：<https://www.yxtiktok.com/blog/tiktok-creator-network-creator-relationship/>
- 热搜：车企老板娘女装卖了30亿；本站资料：<https://www.yxtiktok.com/blog/high-gift-high-growth-stable-tiktok-creator-types/>
- 热搜：动辄千元起步的户外徒步鞋火了；本站资料：<https://www.yxtiktok.com/blog/avoid-duplicate-tiktok-creator-collection-and-followup/>
- 热搜：“民主党向黄仁勋和英伟达宣战”；本站资料：<https://www.yxtiktok.com/blog/what-is-tiktok-live-backstage-agency-functions/>
- 热搜：外交部回应“黄仁勋重回英伟达”；本站资料：<https://www.yxtiktok.com/blog/tiktok-dm-requests-privacy-creator-contact/>
- 热搜：冯远征笑称“考北京人艺其实不难”；本站资料：<https://www.yxtiktok.com/blog/tiktok-coins-gifts-diamonds-difference/>
- 热搜：Mate X8 华为史上最强大折叠；本站资料：<https://www.yxtiktok.com/blog/find-tiktok-live-creators-streaming-now/>
- 热搜：黄奕用AI视频还原过去三十年人生；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-recruitment-funnel-from-lead-to-join/>
- 热搜：上海市委书记陈吉宁被查；本站资料：<https://www.yxtiktok.com/blog/tiktok-country-ranking-find-live-creators/>
- 热搜：中国人登月又近一步；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-creator-network-vs-mcn-shop/>
- 热搜：Anthropic上市5天即破发；本站资料：<https://www.yxtiktok.com/blog/compare-tiktok-live-rankings-across-countries/>
- 热搜：培养更多高技能人才和大国工匠；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-recruitment-communication-tips/>
- 热搜：特朗普连发30多张照片“回味”深圳；本站资料：<https://www.yxtiktok.com/blog/evaluate-tiktok-creator-with-limited-history/>
- 热搜：睡眠大赛4人躺赢平分6000元；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-multilingual-recruitment-script-library/>
- 热搜：田秉毅任国家羽毛球队主教练；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-pk-interaction-planning/>
- 热搜：旺仔牛奶更不好卖了；本站资料：<https://www.yxtiktok.com/blog/small-tiktok-live-agency-priority-creator-types/>
- 热搜：19楼落地窗底部装无限位外凸窗；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-diamonds-stability-vs-one-day-peak/>
- 热搜：几个姑娘在-25℃的黑龙江清凉上阵；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-feed-creator-recruitment/>
- 热搜：大爷抵脖子不放后小伙倒地 地铁回应；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-cohost-chatting-tips/>
- 热搜：课堂上的“嘉豪、嘉欣” 已不敢举手；本站资料：<https://www.yxtiktok.com/blog/why-recheck-creator-eligibility-before-contact/>
- 热搜：中国城市 重排全球座次；本站资料：<https://www.yxtiktok.com/blog/why-agencies-should-not-rely-only-on-tiktok-top-rankings/>
- 热搜：天还没亮 学生就集体站在操场晨读；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-budget-training-vs-recruitment-reach/>
- 热搜：香奈儿包日租仅需25元？押金要一万六；本站资料：<https://www.yxtiktok.com/blog/evaluate-tiktok-live-leaderboard-for-recruitment-list/>
- 热搜：一校一主题 消谣育青年；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-first-recruitment-dm-scripts/>
- 热搜：换脸还没消停 “AI融脸”又来了；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-creator-network-violation-consequences/>
- 热搜：北京天空现“超级月亮”；本站资料：<https://www.yxtiktok.com/blog/tiktok-short-video-creator-recruitment/>
- 热搜：重庆男孩被坠楼男子砸中送医；本站资料：<https://www.yxtiktok.com/blog/build-tiktok-agency-creator-selection-criteria/>
- 热搜：央行将开展8000亿元MLF操作；本站资料：<https://www.yxtiktok.com/blog/tiktok-recruitment-channel-conversion-analysis/>
- 热搜：受贿1.13亿余元！吴胜华被判无期；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-low-live-hours-diagnosis/>
- 热搜：女子登山上班往返约6小时已瘦12斤；本站资料：<https://www.yxtiktok.com/blog/tiktok-streamer-rejects-agency-advice/>
- 热搜：中考要不要考历史 教材主编征集意见；本站资料：<https://www.yxtiktok.com/blog/tiktok-new-streamer-training-curriculum/>
- 热搜：特朗普致电黄仁勋：答应我一件事；本站资料：<https://www.yxtiktok.com/blog/manual-vs-software-tiktok-streamer-recruitment/>
- 热搜：大众拟裁员10万人 工会威胁罢工；本站资料：<https://www.yxtiktok.com/blog/tiktok-streamer-performance-decline-recovery-plan/>
- 热搜：小米Q3净利润48.84亿 同比减少56.8%；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-recruitment-messages-lose-trust/>
- 热搜：黄磊真空围裙；本站资料：<https://www.yxtiktok.com/blog/tiktok-creator-pre-signing-info-minimization/>

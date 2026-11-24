# Angular Reactive Forms 实战：动态字段、交叉校验与复杂表单

业务资料表单可能包含多个联系人、动态地址和依赖其他字段的校验。如果每个输入都由手写 DOM 监听器管理，状态会迅速分散。Angular Reactive Forms 将表单结构表示为可操作的控件模型。本文采用类型化表单设计联系人管理模块。

## 1. 建立表单模型

Reactive Forms 的核心是控件模型而不是 HTML。先明确字段类型、默认值和约束，再绑定模板。

```html
<div>
  <form [formGroup]="form"><label>名称 <input formControlName="name"></label></form>
</div>
```

```ts
readonly form = new FormGroup({
  name: new FormControl('', { nonNullable: true, validators: [Validators.required] }),
  contacts: new FormArray<FormGroup<ContactControls>>([])
});
```

## 2. 在 standalone 组件中导入模块

独立组件需要在 imports 中声明 ReactiveFormsModule。模板里的 formGroup 和 formControlName 才能识别表单模型。

```html
<div>
  <form [formGroup]="form" (ngSubmit)="save()"><button type="submit">保存资料</button></form>
</div>
```

```ts
@Component({ standalone: true, imports: [ReactiveFormsModule], templateUrl: './profile.html' })
export class ProfileComponent { /* 表单模型与事件方法 */ }
```

## 3. 明确字段错误的展示时机

表单初始为空时不必立即显示全部错误。通常在字段被触碰或提交尝试后才展示反馈。

```html
<div>
  <label>名称 <input formControlName="name" [attr.aria-invalid]="name.invalid && name.touched"></label>
  @if (name.invalid && name.touched) { <p role="alert">请输入名称</p> }
</div>
```

`name` 可以作为 `form.controls.name` 的 getter。提交失败时调用 markAllAsTouched，使用户能够定位尚未完成的字段。

## 4. 定义类型化联系人组

FormArray 的元素可以是同一类型的 FormGroup。以独立工厂创建每个联系人，避免共享同一个控件实例。

```html
<div>
  <fieldset><legend>联系人</legend><label>邮箱 <input type="email" formControlName="email"></label></fieldset>
</div>
```

```ts
type ContactControls = { email: FormControl<string>; primary: FormControl<boolean> };
function createContact(): FormGroup<ContactControls> {
  return new FormGroup({ email: new FormControl('', { nonNullable: true, validators: [Validators.required, Validators.email] }), primary: new FormControl(false, { nonNullable: true }) });
}
```

## 5. 动态新增和删除联系人

使用 FormArray 的 push、removeAt 等方法管理控件，而不是直接修改 controls 数组。这能让框架正确维护控件关系和状态。

```html
<div>
  <button type="button" (click)="addContact()">新增联系人</button>
</div>
```

```ts
addContact() { this.form.controls.contacts.push(createContact()); }
removeContact(index: number) { this.form.controls.contacts.removeAt(index); }
```

## 6. 在模板中渲染动态控件

循环中通过 formGroupName 关联当前控件组。追踪控件实例可以减少删除中间行时的身份错配。

```html
<div>
  <section formArrayName="contacts">
    @for (contact of form.controls.contacts.controls; track contact; let i = $index) {
      <fieldset [formGroupName]="i"><label>邮箱 <input formControlName="email"></label><button type="button" (click)="removeContact(i)">删除</button></fieldset>
    }
  </section>
</div>
```

## 7. 实现跨字段校验

如果业务要求最多一个主要联系人，校验应放在整个联系人数组上，而不是某个邮箱字段。校验函数返回错误对象或 null。

```html
<div>
  @if (form.controls.contacts.hasError('multiplePrimary')) { <p role="alert">只能指定一个主要联系人</p> }
</div>
```

```ts
const onePrimary: ValidatorFn = control => {
  const values = control.value as { primary: boolean }[];
  return values.filter(value => value.primary).length > 1 ? { multiplePrimary: true } : null;
};
form.controls.contacts.addValidators(onePrimary);
form.controls.contacts.updateValueAndValidity();
```

## 8. 处理异步校验的等待状态

邮箱是否已被使用可能需要远程验证。AsyncValidatorFn 应返回 Promise 或 Observable，并处理网络异常；不能把无法验证直接等同于合法。

```html
<div>
  @if (form.pending) { <p role="status">正在校验输入</p> }
  <button type="submit" [disabled]="form.pending">保存</button>
</div>
```

远程唯一性校验可能具有竞态，服务端仍需最终唯一约束。可使用 updateOn: 'blur' 减少每次键入带来的请求。

## 9. 区分 value 与 getRawValue

禁用控件通常不参与父控件的 value 聚合。若业务确实需要包含禁用字段，可以读取 getRawValue，再显式选择需要提交的字段。

```html
<div>
  <p>系统标识由服务器管理，不能直接编辑。</p><input formControlName="name" aria-label="资料名称">
</div>
```

```ts
save() {
  if (this.form.invalid || this.form.pending) { this.form.markAllAsTouched(); return; }
  const input = this.form.getRawValue();
  void this.profileService.save({ name: input.name, contacts: input.contacts });
}
```

## 10. 验证动态结构与校验规则

测试至少包含新增删除、重复主要联系人、非法邮箱、空数组和禁用字段。模板测试应确认错误提示与具体控件关联。

```html
<div>
  <section aria-label="保存结果"><p role="status">{{ saveStatus }}</p></section>
</div>
```

服务端返回字段级错误时，应通过明确的错误映射定位相应控件，不要直接把任意服务端字符串转换成模板代码。

## 总结

Reactive Forms 的优势在于可以将复杂表单表达为结构化模型。类型化控件、FormArray 和分层校验使动态表单更可维护；最终保存仍依赖独立的权限、接口校验和业务约束。

## 热搜标题与本站资料

以下热搜标题仅作当日资讯记录；本站链接为独立资料，并非该新闻的报道。

- 热搜：田秉毅任国家羽毛球队主教练；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-creator-live-schedule/>
- 热搜：女子登山上班往返约6小时已瘦12斤；本站资料：<https://www.yxtiktok.com/blog/tiktok-new-streamer-pre-live-checklist/>
- 热搜：培养更多高技能人才和大国工匠；本站资料：<https://www.yxtiktok.com/blog/tiktok-short-video-creator-recruitment/>
- 热搜：分享你拍的超级月亮；本站资料：<https://www.yxtiktok.com/blog/tiktok-creator-network-cross-region-compliance/>
- 热搜：Anthropic上市5天即破发；本站资料：<https://www.yxtiktok.com/blog/tiktok-streamer-recruitment-channels-by-country/>
- 热搜：余承东现场提词器曝光；本站资料：<https://www.yxtiktok.com/blog/what-is-tiktok-live-backstage-agency-functions/>
- 热搜：黄磊真空围裙；本站资料：<https://www.yxtiktok.com/blog/tiktok-creator-discovery-ranking-live-profile-workflow/>
- 热搜：动辄千元起步的户外徒步鞋火了；本站资料：<https://www.yxtiktok.com/blog/avoid-spam-repeated-dm-tiktok-agency-recruitment/>
- 热搜：年产1500万根的“棍子” 卖出10亿元；本站资料：<https://www.yxtiktok.com/blog/tiktok-creator-network-multi-account-invite-risk/>
- 热搜：Mate X8 华为史上最强大折叠；本站资料：<https://www.yxtiktok.com/blog/tiktok-streamer-flexible-live-scheduling/>
- 热搜：19楼落地窗底部装无限位外凸窗；本站资料：<https://www.yxtiktok.com/blog/screen-tiktok-creator-public-live-content-policy-risk/>
- 热搜：冯远征笑称“考北京人艺其实不难”；本站资料：<https://www.yxtiktok.com/blog/reduce-duplicate-work-multiple-tiktok-agency-accounts/>
- 热搜：伪人AI广告 已经成了一种新的城市病；本站资料：<https://www.yxtiktok.com/blog/prevent-tiktok-agency-recruiter-duplicate-contact/>
- 热搜：车企老板娘女装卖了30亿；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-recruitment-support-pitch/>
- 热搜：特朗普连发30多张照片“回味”深圳；本站资料：<https://www.yxtiktok.com/blog/new-tiktok-agency-first-month-action-plan/>
- 热搜：重庆男孩被坠楼男子砸中送医；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-creator-network-violation-consequences/>
- 热搜：大众拟裁员10万人 工会威胁罢工；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-recruitment-channels/>
- 热搜：加州给548名死刑犯减刑 特朗普震怒；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-streamer-feedback-follow-up/>
- 热搜：“民主党向黄仁勋和英伟达宣战”；本站资料：<https://www.yxtiktok.com/blog/why-tiktok-agency-deduplicate-creator-uid/>
- 热搜：文吉儿从小就活跃在各大荧幕；本站资料：<https://www.yxtiktok.com/blog/personalize-tiktok-creator-outreach-by-live-content/>
- 热搜：课堂上的“嘉豪、嘉欣” 已不敢举手；本站资料：<https://www.yxtiktok.com/blog/tiktok-backstage-creator-management-relationship/>
- 热搜：东北外卖15分钟冻成冰；本站资料：<https://www.yxtiktok.com/blog/manual-vs-software-tiktok-streamer-recruitment/>
- 热搜：睡眠大赛4人躺赢平分6000元；本站资料：<https://www.yxtiktok.com/blog/record-tiktok-creator-invite-check-history/>
- 热搜：太平洋暖得“失控”了；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-awkward-silence-topic-switching/>
- 热搜：泰警方确认女子异地遇害被肢解抛尸；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-pk-interaction-planning/>
- 热搜：特朗普致电黄仁勋：答应我一件事；本站资料：<https://www.yxtiktok.com/blog/how-to-check-tiktok-creator-agency-invite-eligibility/>
- 热搜：一校一主题 消谣育青年；本站资料：<https://www.yxtiktok.com/blog/early-signs-streamer-unhappy-with-agency/>
- 热搜：注水长剧的“狼”来了；本站资料：<https://www.yxtiktok.com/blog/assess-third-party-tiktok-live-ranking-coverage/>
- 热搜：中国人登月又近一步；本站资料：<https://www.yxtiktok.com/blog/tiktok-streamer-live-schedule-reminder-scripts/>
- 热搜：大爷抵脖子不放后小伙倒地 地铁回应；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-ranking-find-creators/>
- 热搜：旺仔牛奶更不好卖了；本站资料：<https://www.yxtiktok.com/blog/tiktok-diamonds-vs-creator-rewards/>
- 热搜：小米Q3净利润48.84亿 同比减少56.8%；本站资料：<https://www.yxtiktok.com/blog/select-tiktok-entertainment-live-creator-content-types/>
- 热搜：新娘提1341次分手新郎回1342次不行；本站资料：<https://www.yxtiktok.com/blog/tiktok-country-ranking-find-live-creators/>
- 热搜：外交部回应“黄仁勋重回英伟达”；本站资料：<https://www.yxtiktok.com/blog/schedule-cross-timezone-tiktok-creator-recruiting-team/>
- 热搜：飞机与“超级月亮”浪漫同框；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-multilingual-recruitment-script-library/>
- 热搜：香奈儿包日租仅需25元？押金要一万六；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-streamer-manager-performance-kpis/>
- 热搜：天还没亮 学生就集体站在操场晨读；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-streamer-onboarding-handoff/>
- 热搜：华为首款小折叠电脑发布；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-recruiter-performance-kpis/>
- 热搜：央行将开展8000亿元MLF操作；本站资料：<https://www.yxtiktok.com/blog/tiktok-live-feed-creator-recruitment/>
- 热搜：换脸还没消停 “AI融脸”又来了；本站资料：<https://www.yxtiktok.com/blog/tiktok-coins-gifts-diamonds-difference/>
- 热搜：上海市委书记陈吉宁被查；本站资料：<https://www.yxtiktok.com/blog/which-live-metrics-tiktok-agency-should-evaluate/>
- 热搜：受贿1.13亿余元！吴胜华被判无期；本站资料：<https://www.yxtiktok.com/blog/tiktok-pk-gifts-live-data-review/>
- 热搜：几个姑娘在-25℃的黑龙江清凉上阵；本站资料：<https://www.yxtiktok.com/blog/tiktok-agency-creator-incentive-communication/>
- 热搜：国产可口可乐不再标生产日期；本站资料：<https://www.yxtiktok.com/blog/tiktok-chat-streamer-improvisation-training/>
- 热搜：中国城市 重排全球座次；本站资料：<https://www.yxtiktok.com/blog/tiktok-game-streamer-audience-interaction/>
- 热搜：华为Mate 90 Air售价4999元起；本站资料：<https://www.yxtiktok.com/blog/find-largest-dropoff-tiktok-agency-recruitment-funnel/>
- 热搜：中考要不要考历史 教材主编征集意见；本站资料：<https://www.yxtiktok.com/blog/tiktok-streamer-performance-decline-recovery-plan/>
- 热搜：松鼠妈妈六次示范教小松鼠跳树成功；本站资料：<https://www.yxtiktok.com/blog/tiktok-singing-live-setlist-interaction/>
- 热搜：黄奕用AI视频还原过去三十年人生；本站资料：<https://www.yxtiktok.com/blog/tiktok-streamer-recruitment-communities/>
- 热搜：北京天空现“超级月亮”；本站资料：<https://www.yxtiktok.com/blog/tiktok-part-time-vs-full-time-streamer-management/>

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

## 本站 URL 数据样本

以下为本站公开页面地址样本，与上文技术示例及下方热搜无关。

```text
https://www.yxtiktok.com/blog/small-tiktok-live-agency-priority-creator-types/
https://www.yxtiktok.com/blog/tiktok-agency-recruiter-performance-kpis/
https://www.yxtiktok.com/blog/why-tiktok-creator-invite-eligibility-changes/
https://www.yxtiktok.com/blog/tiktok-live-creator-network-violation-consequences/
https://www.yxtiktok.com/blog/prioritize-tiktok-creators-by-live-activity-eligibility/
https://www.yxtiktok.com/blog/why-recheck-creator-eligibility-before-contact/
https://www.yxtiktok.com/blog/tiktok-entertainment-live-opening-retention/
https://www.yxtiktok.com/blog/tiktok-live-diamonds-stability-vs-one-day-peak/
https://www.yxtiktok.com/blog/tiktok-agency-recruitment-funnel-from-lead-to-join/
https://www.yxtiktok.com/blog/tiktok-agency-streamer-onboarding-handoff/
https://www.yxtiktok.com/blog/top-vs-mid-tier-tiktok-live-creators-agency/
https://www.yxtiktok.com/blog/tiktok-agency-recruiter-offboarding-leads-handover/
https://www.yxtiktok.com/blog/tiktok-streamer-rejects-agency-advice/
https://www.yxtiktok.com/blog/tiktok-creator-segment-recruitment-difficulty-value/
https://www.yxtiktok.com/blog/tiktok-dance-live-content-interaction-plan/
https://www.yxtiktok.com/blog/tiktok-pk-gifts-live-data-review/
https://www.yxtiktok.com/blog/tiktok-agency-multilingual-recruitment-script-library/
https://www.yxtiktok.com/blog/tiktok-live-feed-creator-recruitment/
https://www.yxtiktok.com/blog/tiktok-agency-turnover-revenue-gross-net-profit/
https://www.yxtiktok.com/blog/tiktok-live-cohost-chatting-tips/
https://www.yxtiktok.com/blog/tiktok-agency-low-live-hours-diagnosis/
https://www.yxtiktok.com/blog/can-individual-apply-tiktok-live-creator-network/
https://www.yxtiktok.com/blog/three-tier-tiktok-creator-lead-priority-system/
https://www.yxtiktok.com/blog/tiktok-new-vs-experienced-streamer-outreach/
https://www.yxtiktok.com/blog/how-to-check-tiktok-creator-agency-invite-eligibility/
https://www.yxtiktok.com/blog/tiktok-new-streamer-training-curriculum/
https://www.yxtiktok.com/blog/solo-vs-team-tiktok-agency-recruitment-workflow/
https://www.yxtiktok.com/blog/tiktok-live-entertainment-gaming-shopping-creators/
https://www.yxtiktok.com/blog/prevent-tiktok-agency-recruiter-duplicate-contact/
https://www.yxtiktok.com/blog/creator-already-in-other-network-new-invite/
https://www.yxtiktok.com/blog/screen-tiktok-creator-public-live-content-policy-risk/
https://www.yxtiktok.com/blog/tiktok-streamer-live-schedule-reminder-scripts/
https://www.yxtiktok.com/blog/benefits-of-joining-tiktok-live-agency/
https://www.yxtiktok.com/blog/find-tiktok-live-creators-streaming-now/
https://www.yxtiktok.com/blog/tiktok-agency-streamer-manager-performance-kpis/
https://www.yxtiktok.com/blog/reduce-duplicate-work-multiple-tiktok-agency-accounts/
https://www.yxtiktok.com/blog/why-tiktok-agency-needs-employee-audit-logs/
https://www.yxtiktok.com/blog/tiktok-coins-gifts-diamonds-difference/
https://www.yxtiktok.com/blog/assess-tiktok-creator-live-positioning-from-profile/
https://www.yxtiktok.com/blog/tiktok-live-league-tiers-creator-discovery/
https://www.yxtiktok.com/blog/tiktok-agency-creator-tiered-management/
https://www.yxtiktok.com/blog/tiktok-live-ranking-find-creators/
https://www.yxtiktok.com/blog/tiktok-game-streamer-audience-interaction/
https://www.yxtiktok.com/blog/clean-invalid-stale-tiktok-creator-lead-records/
https://www.yxtiktok.com/tutorials/tiktok-ios-non-jailbreak-dm-tutorial/
https://www.yxtiktok.com/blog/tiktok-creator-pipeline-status-management/
https://www.yxtiktok.com/blog/tiktok-live-pk-interaction-planning/
https://www.yxtiktok.com/blog/tiktok-creator-network-cross-region-compliance/
https://www.yxtiktok.com/blog/record-tiktok-creator-invite-check-history/
https://www.yxtiktok.com/blog/why-tiktok-live-ranking-differs-from-real-performance/
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

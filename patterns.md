# 动态工作流模式

一份编排形态的目录。每一条先说明该形态适用的时机，再展示
它的正确写法。接口在显而易见之处省略——你的脚本必须
定义它提到的每一个类型。

这里的代码片段是**片段，不是可直接运行的脚本**：它们提到的类型与子代理，
周围的行文留给你补上。完整的脚本见 `examples.md`。

**从流水线形态入手。** 大多数脚本不止一个阶段，各阶段应当同时工作，
每一项一就绪就向前推进（SKILL.md §7）。#10 在一次扇出内
把一一对应的阶段串接起来；#11 用通道在阶段之间流转条目；#12
增加一条从靠后阶段回到靠前阶段的反馈回路。其余形态是
搭建这些流水线所用的零件。

API 面本身请读 SKILL.md §16。这些选择背后的理由，
请读 `SKILL.md` 的其余部分。

---

## 1. 对一个 glob 做扇出 / 扇入

**何时：** 对一组文件中的每一个问同一个相互独立的问题，
并希望它们一次性全部得到回答。

```ts
phase("Find the files to audit");
const paths = await files.glob("src/**/*.ts");
log(`fanning out over ${paths.length} files`);

phase("Security-audit each file independently");
// 下面的报告需要每一条判定，所以这里的汇合才是正确选择。
const verdicts = await Promise.all(
  paths.map((p) =>
    agent(`auditor-${p}`).ask<Verdict>(`Security-review ${p}. Report only real issues.`),
  ),
);

phase("Report the files that failed");
return verdicts.filter((v) => !v.approved).map((v) => ({ path: v.path, reason: v.reason }));
```

每个路径配一个全新子代理正是要点：这些问题互不相关，所以没有可共享的东西，
而一切都值得并行。不要把 `agent(...)` 提到 `map` 之上。

名字里带上路径，是因为子代理名在一次运行内必须唯一：在 `map` 里写 `agent("auditor")`
会为每个文件各创建一个子代理却都声称同一个名字，这在编译时就会被拒绝。
`` `auditor-${p}` `` 也是让修订后的重跑（`AmendWorkflow`）保留那些问题未变的文件的判定
的原因。匿名——`agent()`——同样合法，
并且让每个文件都从空上下文开始。

**扇出之前先收窄。** 对大仓库做一次 glob 就是一次大规模扇出。如果只有一部分文件
可能相关，先把它们找出来：

```ts
const hits = await files.grep("dangerouslySetInnerHTML", "src/**/*.tsx");
const paths = [...new Set(hits.map((h) => h.path))];
```

`files.grep` 超出上限时会拒绝而不是截断，所以过宽的模式会响亮地失败，
而不是递给你一份残缺的文件清单让你去扇出。

---

## 2. 变更文件评审扫描

**何时：** 评审进行中的工作，而不是整棵树。

```ts
phase("Find the work in progress");
let paths: string[];
try {
  paths = await git.changedFiles("origin/main");
} catch {
  paths = await files.glob("src/**/*.ts");
}

phase("Review each changed file for bugs");
const reviews = await Promise.all(
  paths.map((p) => agent(`reviewer-${p}`).ask<Review>(`Review ${p} for correctness bugs.`)),
);
```

每一次 `git` 调用在仓库之外或没有 `git` 可用时都会拒绝，所以带 `files.glob` 回退的
`try`/`catch` 是这里的惯用法，不是防御性凑数。

当评审员需要的是变更而不是文件本身时，把 diff 递给它——但按路径分开递，
不让任何单个提示词背下整个变更集：

```ts
const perFile = await Promise.all(
  paths.map(async (p) => {
    const patch = await git.diff("origin/main", p);
    return agent(`reviewer-${p}`).ask<Review>(`Review this change to ${p}:\n\n${patch}`);
  }),
);
```

`git.diff` 有上限且超限时拒绝而非截断，所以当整个工作区的 diff 会很大时，
把它收窄到单个路径。

然后在报告之前先确认——逐文件进行，每份评审一落地就确认，而不是等全部结束。
提出发现的评审员无权确认它——被要求检查自己工作的子代理打分会手松——
所以每条发现都交给一位全新复核员，它仅凭证据复现这条发现，
并被叮嘱不得修改任何东西。把复核员串接在同一个回调内部，
意味着最慢的评审员只会拖住它自己那个文件：

```ts
phase("Review each changed file and confirm its findings as they land");
const confirmed = (
  await Promise.all(
    paths.map(async (p) => {
      const review = await agent(`reviewer-${p}`).ask<Review>(`Review ${p} for correctness bugs.`);
      return Promise.all(
        review.findings.map(async (finding, index) => {
          const check = await agent(`confirmer-${p}-${index}`).ask<Confirmation>(
            `Reproduce this finding from its evidence alone. Do not edit any file.\n${JSON.stringify(finding)}`,
          );
          const reported = { ...finding, status: check.reproduced ? "verified" : "unconfirmed" };
          report(reported);
          return reported;
        }),
      );
    }),
  )
).flat();
```

未通过确认的发现会被保留并打上标记，而不是丢弃。上面的评审扇出与这一个
是同一形态为讲解写了两遍；真实脚本里只写这一个。

---

## 3. 规划者 ↔ 评审员循环

**何时：** 第一稿很少是对的，而一位批评者能说出为什么。

```ts
const planner = agent("planner", "You write concrete, minimal implementation plans.");
const reviewer = agent("reviewer", "You find the flaw in a plan. Approve only when you cannot. You never edit files.");

let feedback = "none";
for (let round = 0; round < 5; round++) {
  phase("Draft a plan for the change");
  const plan = await planner.ask<Plan>(`Plan the change. Previous critique: ${feedback}`);
  phase("Critique the plan until it holds");
  const review = await reviewer.ask<Review>(`Critique this plan:\n${JSON.stringify(plan)}`);
  if (review.approved) return plan;
  log(`round ${round + 1} rejected: ${review.feedback}`);
  feedback = review.feedback;
}
```

两个子代理都**只创建一次，在循环之外**。这正是循环便宜的原因：二者各自跨轮次
保留积累的上下文，规划者记得自己已经试过什么，
评审员记得自己已经反对过什么。在循环内部创建它们
则每一轮都把这些扔掉，并为此付全价。

评审员的角色设定说它从不编辑文件：这正是让它持续批评方案、而不是悄悄"修好"方案
的关键。批评方案只是读一个字符串时，它压根不会打开文件；
当方案涉及代码时，告诉它去读代码——没打开过文件的评审员
只能判断方案是否自洽，不能判断它是否正确。

到最后一轮，持久的评审员已经锚定在自己先前的反对意见上。把最终方案
放到一双什么都没见过的新眼睛面前，让它找失败
而不是给批准：

```ts
phase("Independent review of the final plan");
const second = await agent("independent-reviewer", "You review plans and never edit files.").ask<Review>(
  `You have not seen this plan before. Read it against the codebase. What would break it, and what is missing?
${JSON.stringify(plan)}`,
);
if (!second.approved) feedback = second.feedback;
```

---

## 4. 评委团：独立式，或校准式

**何时：** 一条发现需要第二意见。两种形态，差别很要紧。

**独立式**——N 个全新上下文，彼此互不知情。用于多数表决，
彼此相关的评委会让表决失去意义：

```ts
phase("Judge the finding from three angles");
// 多数决需要每一票，所以汇合等待全部三票。
const votes = await Promise.all(
  ["correctness", "security", "does-it-reproduce"].map((lens) =>
    agent(`judge-${lens}`)
      .ask<Verdict>(`Judge this finding through the ${lens} lens. Try to refute it:\n${claim}`),
  ),
);
const survives = votes.filter((v) => !v.refuted).length >= 2;
```

给每位评委一个不同的视角胜过三个一模一样的反驳者：多样性
能抓住冗余抓不住的失败模式。

**校准式**——一个看遍所有条目的上下文，它的各条判定彼此一致。用于排序与定级，
此时"高"必须两次是同一个意思：

```ts
phase("Rank every finding on one scale");
const triage = agent("triage", "You rank findings on one consistent scale across a whole batch.");

const ranked: Ranked[] = [];
for (const finding of findings) {
  ranked.push(await triage.ask<Ranked>(`Rank: ${JSON.stringify(finding)}`));
}
```

这里的 `for`/`await` 是刻意的——对同一子代理的 ask 本来就按 FIFO 排队，
写成 `Promise.all` 只是把串行藏起来，并不会消除它。

不过队列不是屏障。校准式子代理不需要在开工前把整批条目握在手里：
让前一阶段产出一条就喂一条（在该阶段的扇出回调内部，SKILL.md §11），
它的判定保持一致，
流水线也继续流动。

无论哪种，评委的判定是一种判断，不是复现：在评委团存活下来的发现
仍以 `unconfirmed` 进入报告，除非某位复核员或某次 `world.run` 检查
复现了它。

---

## 5. 循环直到批准，并带一个真正的出口

**何时：** 一个有界循环，轮次用尽时仍必须返回有用的东西。

```ts
let best: Attempt | undefined;
for (let round = 0; round < 6; round++) {
  phase("Attempt the fix");
  const attempt = await worker.ask<Attempt>(`Attempt the fix. Prior failure: ${lastError ?? "none"}`);
  phase("Check whether it actually passes");
  const check = await checker.ask<Check>(`Does this pass? ${JSON.stringify(attempt)}`);
  if (check.passed) return { attempt, rounds: round + 1, converged: true };
  best = attempt;
  lastError = check.reason;
}
return { attempt: best, rounds: 6, converged: false };
```

返回的形状要说明**是否收敛**，而不只是结果。一个分不清"第二轮才批准"
与"六轮后放弃"的调用者，会把后者当成前者。
绝不要让循环从末尾掉出去、什么都不返回。

---

## 6. 带类型化结果的分阶段交接

**何时：** 每个阶段都在收窄或变换下一个阶段的加工对象，而最后一个阶段
需要前一个阶段产出的全部东西。

```ts
phase("Survey which modules touch auth");
const survey = await agent("surveyor").ask<Survey>("List the modules that touch auth.");
log(`${survey.modules.length} modules in scope`);

phase("Analyse each module for auth bypasses");
// 成文要跨模块去重，所以它需要一次性拿到全部分析。
const analyses = await Promise.all(
  survey.modules.map((m) =>
    agent(`analyst-${m.path}`).ask<Analysis>(
      `Analyse ${m.path} for auth bypasses. Purpose: ${m.purpose}`,
    ),
  ),
);

phase("Write the findings up for the engineer who fixes them");
const synth = agent("synthesist", "You write findings up for an engineer who will fix them.");
return synth.ask<Writeup>(`Write these up, deduplicated:\n${JSON.stringify(analyses)}`);
```

每个阶段的结果就是下一个阶段的输入，而类型实参正是让交接安全的东西——
`survey.modules.map` 之所以能编译，是因为 `Survey` 说清了返回的是什么。

注意最后一次调用的形态：单个综合子代理一次性拿到全部，因为跨发现去重
恰恰是那个必须看到全部的工作。
不要并行化一个其全部目的就是跨条目比较的阶段。

反过来也成立。分析阶段与勘察出的模块一一对应，所以如果分析之后跟着一个
逐模块检查，那个检查就该待在与分析同一个回调内部——
而不是躲在第二个 `Promise.all` 后面（形态 10）。把屏障留给
需要所有人的那个阶段。

---

## 7. 有界探索

**何时：** 工作没有天然的尺寸——是"找出 bug"，
而不是"检查这十二个文件"。

```ts
const MAX_ROUNDS = 6;
const found: Bug[] = [];
const seen = new Set<string>();

for (let round = 1; round <= MAX_ROUNDS; round++) {
  phase("Hunt for bugs not already found");
  const batch = await agent("hunter").ask<Bugs>(
    `Find bugs not already in this list: ${JSON.stringify([...seen])}`,
  );
  const fresh = batch.bugs.filter((b) => !seen.has(`${b.path}:${b.line}`));
  if (fresh.length === 0) break; // 已干涸：停止，剩下的轮次只会重复

  for (const bug of fresh) {
    seen.add(`${bug.path}:${bug.line}`);
    report(bug);
    found.push(bug);
  }
  log(`${found.length} found after round ${round}`);
}
return found;
```

两道保险，而且两道都需要：轮次上限防止猎手不断有收获时循环永远跑下去，
干涸检查则在猎手再也找不到新东西时停住它。上限是你在脚本里选定并随修订调整的旋钮——
harness 不强制任何节点上限，所以没有上限的循环只有用户取消运行才会结束。
与全部**见过的**去重，而不是与全部保留的——
对保留清单去重会让被否决的发现
每一轮都重新出现，循环永不收敛。

---

## 8. 靠 report 抢救

**何时：** 任何时候，任何长得足以在中途失败的运行。

```ts
// 错误写法：四十项工作，第十二项失败就什么都返回不了。
const results: Result[] = [];
for (const item of items) results.push(await worker.ask<Result>(`Handle ${item.id}`));
return results;

// 正确写法：每个结果在诞生的瞬间就发布。
phase("Handle each item and publish as it lands");
const results: Result[] = [];
for (const item of items) {
  const result = await worker.ask<Result>(`Handle ${item.id}`);
  report(result);
  results.push(result);
}
return results;
```

已报告的条目在出错与停止的运行上也会随完成通知一起送达，成功运行亦然，
而且它们会被记录在案，所以续跑的运行不会重复发布它们。
返回值只有脚本走到 `return` 才存活；已报告的条目
无论如何都存活。报告发现，不要报告闲聊——每次运行有条目上限，
超出上限会让运行失败。

同一个调用还能喂饱一张图。给条目打上一个预设产物 id——
`report(result, "progress")`——它就会同时落入本次运行的结果和以该名字命名的
图表、表格、指标或看板，随循环实时更新（SKILL.md §10）。

在单个任务外包一层 `try`/`catch`，能把一次**逻辑**失败——子代理结果未通过校验、
某道门禁未通过、一次超出上限的 world 读取、一次因文件缺失而失败的产物发布——
变成部分结果而不是死掉的运行。它不是为 provider 错误准备的：
模型侧错误永远到不了脚本。瞬时错误（限流、过载、
网络错误、超时、未知 provider 错误）由运行时无限制重试；
确定性错误（登录过期、模型不在套餐内、配额上限）会以 `stopped`
停下整个运行，让用户修好原因后续跑。为它们在 ask 外套重试循环
是死代码。脚本唯一能捕获的模型相关错误是 `ContextLimit`：
ask 本身经压缩后对模型而言仍然过大，修法是缩小 ask。

```ts
for (const item of items) {
  try {
    report(await worker.ask<Result>(`Handle ${item.id}`));
  } catch (error) {
    log(`skipped ${item.id}: ${String(error)}`);
  }
}
```

---

## 9. 带门禁的核对员循环 (world.run)

**何时：** 停止条件是机器可判定的——一次构建、一个证明检查器、一套测试——
而子代理宣称的成功不值得信任。

```ts
const prover = agent("prover", "You repair the proof. Fix exactly what the checker reports.");

phase("Write the first proof attempt");
await prover.ask<Attempt>(`Prove the open theorem in ${FILE}.`);
let clean = false;
for (let round = 1; round <= 8; round++) {
  phase("Check the file with the fast checker");
  const check = await world.run("lake", ["env", "lean", FILE], { timeoutMs: 600_000 });
  clean = check.exitCode === 0 && !check.stderr.includes("sorry");
  if (clean) break;
  log(`round ${round}: checker rejected`);
  phase("Repair what the checker rejected");
  await prover.ask<Attempt>(`The checker rejected the file:\n${check.stderr}\nRepair ${FILE}.`);
}
if (!clean) return { proved: false };

phase("Build the whole project once before handing over");
const build = await world.run("lake", ["build"], { timeoutMs: 1_800_000 });
return { proved: build.exitCode === 0 };
```

子代理做开放式的工作；脚本做裁决，而裁决不可外包。
`world.run` 真实执行检查器，所以"已验证"从来不是一个宣称——
只是一个退出码。非零退出是一个**值**，这正是循环读取 `check.exitCode`
并把 `check.stderr` 向前传递、而不是捕获任何东西的原因；拒绝只留给
世界未能应答的情形（spawn 失败、超时、输出超上限），而
在调用外包一层 `try`/`catch` 就是脚本为这些情形选择回退的方式。

分两档，而差别正是全部要点。逐文件检查器是快档：
它几秒内应答，所以由它驱动各轮。它不是裁决——
`lake env lean` 对仍写着 `sorry` 的证明退出码为 0，只在 stderr 上警告，
这正是循环也读 stderr 的原因。全项目构建是强档：
它才是请求真正关心的事，即使快档已经说了"通过"，它也在最后运行一次。
两档都要靠读仓库来选——README 里"运行这个来验证"的那一行、
`Makefile`、`package.json` 脚本——绝不要凭习惯；
仓库针对用户所求所能提供的最强检查，
才是说了算的那一个（SKILL.md §5）。

命令名按规则是编译期字面量——用户在确认时批准脚本的命令集——
所以把路径、旗标和轮次编号插值进**参数**，
绝不要插进命令。对比形态 5：同样的循环，但那里的检查器是子代理；
只要真实命令能给出裁决，就优先用这个形态。

---

## 10. 逐条目流水线

**何时：** 两个或更多阶段一一对应——每文件一个猎手、每发现一个复核员，
每文件一个迁移器、每文件一个检查器——而后段没有任何东西需要看到
前段的整体。

```ts
phase("Hunt for bugs in each file and confirm them as they are found");
const confirmed = (
  await Promise.all(
    paths.map(async (p) => {
      const hunt = await agent(`hunter-${p}`).ask<Hunt>(`Hunt for correctness bugs in ${p}.`);
      return Promise.all(
        hunt.bugs.map(async (bug, index) => {
          const check = await agent(`confirmer-${p}-${index}`).ask<Confirmation>(
            `Reproduce this bug from its evidence alone. Do not edit any file.\n${JSON.stringify(bug)}`,
          );
          const reported = { ...bug, status: check.reproduced ? "verified" : "unconfirmed" };
          report(reported);
          return reported;
        }),
      );
    }),
  )
).flat();
```

一次汇合，在末尾，放在报告需要每一条目的地方。对比双屏障版本——
先对猎手做 `Promise.all`，再对复核员做 `Promise.all`——它要等最慢的猎手
做完才启动任何复核员，中间让并发槽位闲置。
修订重跑的缓存标识不变：每个子代理仍以相同顺序被命名和询问。

一个 phase 覆盖整条流水线，以它对每个条目所做的事命名。不要在回调内部
给每个阶段放一个标记：一次运行只有一个当前 phase，并发的回调
乱序重入两个标记会互相盖戳。

当一个条目失败应当只损失一个条目时，在回调内部捕获（examples #3），
或使用 `Promise.allSettled` 并逐个读取结果；普通 `Promise.all` 内部的
一个 rejection 会连带拒绝汇合和所有兄弟。

## 11. 用通道的流式流水线

**何时：** 条目应当在产出时就流向下一阶段，而该形态是
串接（#10）写不出来的：一个由 K 个共享工人服务某阶段的池子、
生产者还在找条目而消费者已在工作、靠后阶段把工作送回（#12），
或者你想要把各阶段读成独立的块。每个阶段是一个
`future`，其函数体以自己的 phase 标记开头；一条 `channel` 在两个阶段之间搬运条目；
生产阶段在 `finally` 里拥有 `close()`。

```ts
const leads = channel<Lead>("leads");
const facts = channel<Fact>("facts");
const checked: Checked[] = [];

const seed = future(async () => {
  phase("Decide what to look into");
  try {
    const plan = await agent("planner").ask<{ questions: string[] }>(`List the questions to answer about ${topic}.`);
    for (const question of plan.questions) leads.send({ question });
  } finally {
    leads.close();
  }
});

const research = future(async () => {
  phase("Research each question");
  try {
    await Promise.all(
      Array.from({ length: RESEARCHERS }, async (_, k) => {
        // 每个工人一个上下文，提升到循环之外：每个工人的名字只铸造一次，
        // 工人在前一个问题上学到的东西，下一个问题还在。
        const researcher = agent(`researcher-${k + 1}`);
        for await (const lead of leads) {
          const found = await researcher.ask<Research>(`Research: ${lead.question}`);
          found.facts.forEach((fact, index) => facts.send({ ...fact, id: `${lead.question}-${index + 1}` }));
        }
      }),
    );
  } finally {
    facts.close(); // 研究员失败也会终结这条流，核对员得以收尾
  }
});

const verify = future(async () => {
  phase("Check each fact as it arrives");
  await Promise.all(
    Array.from({ length: VERIFIERS }, async (_, k) => {
      const verifier = agent(`verifier-${k + 1}`);
      for await (const fact of facts) {
        const verdict = await verifier.ask<Verdict>(`Check: ${JSON.stringify(fact)}`);
        checked.push({ ...fact, ...verdict });
        report({ ...fact, ...verdict });
      }
    }),
  );
});

// 先启动每个阶段，再一次性汇合全部。两者之间没有 await：当脚本
// 等待某个阶段时，另一阶段的失败没有监听者，会以未处理的
// rejection 冒出来，而不是带着它的错误让运行失败。
await Promise.all([seed, research, verify]);

phase("Write the report from the checked facts");
const summary = await agent("writer").ask<string>(`Write up:\n${JSON.stringify(checked)}`);
```

三个阶段，三个标记，一次汇合。writer 放在汇合之后，而不是放在通道上：
它需要每一条核对过的事实，所以共享数组加汇合才是那里正确的交接。
下一阶段应当早点开工，用通道；不能早开工，用汇合。

这里的池子（`researcher-1…3`、`verifier-1…4`）在每个工人碰巧收到的条目之间共享上下文，
这正是条目必须在同一把尺子上裁决、或在一个条目上学到的东西对下一个有用时
你想要的效果。代价是修订运行时的缓存局部性：哪个条目落在哪个工人身上取决于时序，
所以改动一个上游条目的修订会挪动分配，让整个池子的缓存全部落空。
当条目相互独立且你预期会修订时，改为在 `for await` 内部
为每个条目生成一个全新的、以条目命名的子代理——
把 `future` 推进数组，
在阶段结束前汇合（examples #6）。

## 12. 带计数关闭的反馈回路

**何时：** 靠后的阶段把工作送回靠前的阶段——一位怀疑某条事实的核对员
要求对它做更多研究。反馈通道没有单一的生产者知道它何时结束，
所以脚本对未完成的工作项计数，并在计数归零时关闭通道。
每个条目上的深度字段为循环设界。

```ts
const leads = channel<Lead>("leads");
const facts = channel<Fact>("facts");
// 未完成的工作项：每条已发送但尚未研究的线索，每条已发送但尚未裁决的事实。
// 计数归零时反馈通道关闭，其他任何人都不得关闭它。
let open = 0;
function finished(): void {
  open -= 1;
  if (open === 0) leads.close();
}

for (const question of seeds) {
  open += 1;
  leads.send({ question, depth: 0 });
}

const research = future(async () => {
  phase("Research each lead");
  try {
    await Promise.all(
      Array.from({ length: WORKERS }, async (_, k) => {
        const researcher = agent(`researcher-${k + 1}`);
        for await (const lead of leads) {
          const found = await researcher.ask<Research>(`Research: ${lead.question}`);
          // 不变式 1：在结束父项之前先给每个子项计数，这样在仍有工作被产出时
          // `open` 永远不会触及零。
          found.facts.forEach((fact, index) => {
            open += 1;
            facts.send({ ...fact, id: `${lead.question}-${index + 1}`, depth: lead.depth });
          });
          finished();
        }
      }),
    );
  } finally {
    // 不变式 2：研究员只在自己的循环内部发送事实，所以这次关闭在
    // 最后一名研究员离开循环之后运行，此后再无任何东西能向 `facts` 发送。
    facts.close();
  }
});

const verify = future(async () => {
  phase("Check each fact, and send doubts back for more research");
  const checks: Promise<void>[] = [];
  for await (const fact of facts) {
    checks.push(
      future(async () => {
        const verdict = await agent(`verifier-${fact.id}`).ask<Verdict>(`Check: ${JSON.stringify(fact)}`);
        if (verdict.holds) passed.push(fact);
        else if (verdict.followUp !== undefined && fact.depth < MAX_DEPTH) {
          // 这条事实尚未完成，所以 `open` 至少为 1，`leads` 不可能已关闭：
          // 后续问题在产出它的事实结束之前就已计数。
          open += 1;
          leads.send({ question: verdict.followUp, depth: fact.depth + 1 });
        } else rejected.push(fact.claim);
        finished();
      }),
    );
  }
  await Promise.all(checks); // 该阶段在其检查完成时才算完成，而不是在 `facts` 关闭时
});

await Promise.all([research, verify]);
```

为什么由计数关闭 `leads` 而其他任何东西都不行：一个条目只有在它产出的
每个条目都被计数之后才算完成，所以 `open` 恰好归零一次——在最后一代的
最后一个条目被裁决且一无所产之时。研究员随后离开各自的循环，
research 阶段关闭 `facts`，verify 阶段的 `for await` 结束，它的汇合落定。
忘掉"先计数后结束"的顺序，`open` 可能在一个后续问题正要发出时归零——
`leads.send` 随即抛出 `ChannelClosed`。忘掉对 `checks` 的汇合，阶段会带着
仍在运行的裁决提前结束；死锁检测器看不见那一种，因为
什么都没有停泊。忘掉深度上界，两个子代理可以永远互相忙碌下去。

## 13. 模型路由

**何时：** 用户要求在够用的地方用更便宜或更快的模型，而一个子代理
能分辨哪些请求需要强模型。跑在轻量模型上的路由器以类型化的键作答；
代码把键映射到脚本声明过的模型。路由器从不写模型名，
所以它只能在用户在窗口中批准的模型里挑。绝不要自作主张，
也绝不要用用户没点名的另一家 provider 的模型；当你看到更合适的，
先提出并在调用前征询（SKILL.md §2）。

<!-- compile -->
```ts
// 这次运行能用的每个模型，声明一次；名字来自 ListModels。
const MODELS = {
  light: model("GLM-5.3-Flash"),
  strong: model("GLM-5.3$high"),
};

interface Route {
  /** 仅当请求需要多步推理或细致写作时才用 "strong"。 */
  tier: "light" | "strong";
}

interface Reply {
  answer: string;
}

const questions: string[] = ["Rename the config key in the README", "Explain the retry design"];

// 路由与作答一一对应，所以每个问题在路由定下的那一刻
// 就交给它的作答者；唯一一次汇合是为了 return，它需要每一个答案。
phase("Route each question to the model it needs and answer it there");
const replies = await Promise.all(
  questions.map(async (question, index) => {
    const route = await agent(`分诊-${index + 1}`, { model: MODELS.light }).ask<Route>(
      `Would answering this well need the strong model? ${question}`,
    );
    return agent(`答复者-${index + 1}`, { model: MODELS[route.tier] }).ask<Reply>(question);
  }),
);
return replies.map((reply) => reply.answer);
```

固定的选择不需要表格：`agent("评审员", { model: "GLM-5.3-Flash" })`。把表格放在
脚本顶部：一次修订只在一处改模型，而且模型变了的那位子代理
已完成的工作仍然被复用——它的下一次活跃 ask 跑在新模型上。

---

## 14. 靠尾部洞生长

**何时：** 任务复杂到无法从请求直接规划出它的工作流——
下一步取决于上一步找到了什么。写下已知的内容，并以一个尾部洞结束脚本。
脚本随后靠嵌套生长：每次填充都是最内层函数体的语句，做一步，
并以下一个尾部洞结尾；最后一次填充返回。
（`SKILL.md` §15；带全部填充文件的完整链条是 `examples.md` #7。）

```ts
// 0. 种子，随 CreateWorkflow 提交。
phase("盘点仓库顶层目录");
const survey = await agent("勘察员").ask<Survey>("列出顶层目录，各用一句话说明作用。");
return await hole<Guide>("第1步：决定导览结构", `盘点：${JSON.stringify(survey.dirs)}`);
```

```ts
// 1. 第一次 FillWorkflowHole 之后的有效脚本：这个洞获得了一个函数体，
//    它做一步并留下下一个洞。只发送了函数体的语句。
return await hole<Guide>("第1步：决定导览结构", `盘点：…`, async () => {
  phase("分章并行撰写初稿");
  const drafts = await Promise.all(
    survey.dirs.map((d) => agent(`写手·${d.path}`).ask<Chapter>(`写「${d.path}」一章：${d.purpose}`)),
  );
  return await hole<Guide>("第2步：审读与交付", `${drafts.length} 章初稿已就绪`);
});
```

```ts
// 2. 第二次填充之后：链以一个值收尾。每个洞的站点 id 来自它的名字，
//    所以无论多深 id 都保持简短；时间线上每一步各显示一个头。
return await hole<Guide>("第1步：决定导览结构", `盘点：…`, async () => {
  /* 第 1 步同上；`survey` 与 `drafts` 在下方作用域内 */
  return await hole<Guide>("第2步：审读与交付", `…`, async () => {
    phase("独立审读并修订");
    const critique = await agent("审读员").ask<string>(`只凭文本挑毛病：${JSON.stringify(drafts)}`);
    const chapters = await agent("修订编辑").ask<Chapter[]>(`按意见修订：${critique}\n${JSON.stringify(drafts)}`);
    return { title: "仓库导览", chapters, notCovered: [] };
  });
});
```

让链条成立的一组规则：每个洞名在整个有效脚本内唯一，所以按每一步做的事
给它命名（`第N步：…`），绝不用同一个名字两次；提示词
携带下一位作者需要的进展（数量、标题、什么失败了），因为它
是值唯一的通道；结果活在绑定（`survey`、`drafts`）里，每个内层
函数体都读得到它们，所以没有任何东西需要重新询问；一步可以像任何脚本一样
开 phase、创建子代理、扇出、并用 `world.run` 设门禁。当某次填充返回
而不是留下洞时，链条结束。重启在链条等待处续上，
已完成的步骤从日志（journal）重放。

---

## 15. 由探索塑造的阶段

**何时：** 勘察决定的是下一个阶段的*拓扑*，而不只是它的输入——
多少评审员、扇出还是串行、哪道检查为它们把关。勘察的值
以数据的形式交给脚本；填充写这个阶段。

```ts
phase("剖析热点");
const hot = await agent("剖析员").ask<Hotspots>("找出最慢的三个模块，说明各自的耦合。");
const reviews = await hole<Review[]>("怎么审查这些热点", `热点：${JSON.stringify(hot.modules)}`);
phase("汇总");
return await agent("汇总员").ask<Report>(`汇总审查结果：${JSON.stringify(reviews)}`);
```

一份在 `hot` 表明两个模块相互独立、一个被两者共享之后写下的填充：
独立的一对并行评审，共享的那个由一位看过另外两份评审的校准式评审员来评，
然后是一个由项目自己的测试命令把关的有界修复-复检循环。
这些拓扑没有一样是勘察之前可知的。

```ts
phase("并行审查独立模块");
const [a, b] = await Promise.all(
  hot.modules.slice(0, 2).map((m) => agent(`审查员·${m.path}`).ask<Review>(`审查 ${m.path}：${m.coupling}`)),
);
phase("在前两份审查的基础上审查共享模块");
const shared = hot.modules[2];
const reviewer = agent("共享模块审查员", "你在别人的审查基础上工作，只找他们没看到的问题。");
let review = await reviewer.ask<Review>(`已有审查：${JSON.stringify([a, b])}\n现在审查 ${shared.path}`);
for (let round = 1; round <= 3 && review.findings.length > 0; round += 1) {
  const fix = await world.run("pnpm", ["test", "--", shared.path]);
  if (fix.exitCode === 0) break;
  review = await reviewer.ask<Review>(`测试仍失败：${fix.stderr}\n复审 ${shared.path}`);
}
return [a, b, review];
```

与普通子代理 ask 的对照：如果勘察只说了要评审**哪些**模块，
脚本本可以自己对 `hot.modules` 扇出，无需任何洞。

---

## 16. 计划无法枚举的补救

**何时：** 一道门禁失败，而修复可能需要没人规划过的 phase 和子代理。
如果补救办法是已知的——重试、收窄、跳过——脚本会持有一个 `Remedy` 值和一个
`if`；洞是给那种情形准备的：补救本身就是一段工作流，
要拿着失败现场来决定。

```ts
phase("跑完整测试");
const gate = await world.run("pnpm", ["test"], { timeoutMs: 1_800_000 });
if (gate.exitCode !== 0) {
  await hole<void>("测试没过怎么办", `退出码 ${gate.exitCode}：\n${gate.stderr.slice(0, 2000)}`);
}
phase("交付");
return await agent("报告员").ask<Report>("写交付报告。");
```

一次看似偶发的失败之后的填充：用 `world.run` 对最近几次提交做二分，
由一个全新子代理对失败是否属实给第二意见，以及——仅当属实时——
一个以三轮为界的修复循环。清晰失败之后的填充则是一位修复者加一次复检。
脚本不必猜是哪一种，而用户看到的补救是在洞的标题下
属于它自己的各个 phase。

# 动态工作流实操示例

七个完整的脚本，从头到尾。每个都是一条完整的弧线——读取世界、拓扑、循环或扇出、
抢救、产物、返回形态——所以当你想看各个部件如何拼装在一起时，就通读一个，而不是
去查某个单一形态（那件事归 `patterns.md`）。

本文件中的每个脚本都作为 `CreateWorkflow` 工具的 `script` 参数提交。它们针对编写的
API 面是 `SKILL.md` §16（§16.2 的门面代码块与 §16.3 的规则），而不是工具描述。

---

## 1. 对抗式实现 ↔ 验证循环

两个分工对立的子代理，循环往复，直到核对员再也无法击破实现。这个形态可推广到一切
"作者 + 怀疑者"的组合：代码与性质测试、证明与反例、schema 与模糊测试器。

<!-- compile -->
```ts
interface ImplUpdate {
  /** 本轮修改的文件。 */
  changed: string[];
  /** 这个版本如何回应上一条反例。 */
  notes: string;
  /** 交出之前，作者自己的构建与测试是否通过。 */
  selfCheckPassed: boolean;
}

interface Verdict {
  /** 仅当性质确实被验证过、而不只是没人测过时才为 true。 */
  verified: boolean;
  /** 验证成功时，验证产物所在之处。 */
  proofPath?: string;
  /** 验证失败时，能击破实现的具体输入。 */
  counterExamples?: string[];
  /** 原因。这是实现者下一轮的输入。 */
  reason: string;
}

const IMPL_PATH = "src/compress.ts";
const TEST_PATH = "src/compress.property.test.ts";
const PROPERTY = "decompress(compress(s)) === s for every string s";

const implementer = agent("implementer", {
  system:
    `You implement in ${IMPL_PATH}. Run the build and \`npx vitest run\` before handing off, ` +
    "and say in your notes exactly which commands you ran. " +
    "Your deliverable is the file itself: describe what you changed, never paste the source back.",
});

const verifier = agent("verifier", {
  system:
    `You are an adversarial verifier. Read ${IMPL_PATH} and write property tests into ${TEST_PATH}. ` +
    "If you cannot break it, report verified. If you can, give the exact failing input. " +
    "Never claim a verification you did not actually run.",
});

const MAX_ROUNDS = 8;

// 观察者在运行中途想知道的：进行到第几轮，还有多少攻击尚未解决。
artifact.metrics("progress", {
  title: "Verification progress",
  metrics: [{ field: "round", label: "Round" }, { field: "open", label: "Open counter-examples" }],
});

phase("Implement the first version of the compression");
let impl = await implementer.ask<ImplUpdate>(
  `Implement compress and decompress in ${IMPL_PATH} so that ${PROPERTY}.`,
);
phase("Try to break it with property tests");
let verdict = await verifier.ask<Verdict>(
  `Verify or refute: ${PROPERTY}\n` +
    `Changed: ${impl.changed.join(", ")}\nAuthor's notes: ${impl.notes}\n` +
    `Author's self-check: ${impl.selfCheckPassed ? "passed" : "did not pass"}`,
);

let round = 1;
while (!verdict.verified && round < MAX_ROUNDS) {
  phase("Fix what the tests broke");
  round = round + 1;
  const open = verdict.counterExamples?.length ?? 0;
  log(`round ${round}: ${open} counter-examples`);
  report({ round, open, reason: verdict.reason, counterExamples: verdict.counterExamples ?? [] }, "progress");

  impl = await implementer.ask<ImplUpdate>(
    `Verification failed.\nReason: ${verdict.reason}\n` +
      `Counter-examples: ${JSON.stringify(verdict.counterExamples ?? [])}\n` +
      `Fix ${IMPL_PATH} directly; the failing tests are in ${TEST_PATH}. Re-run your own check.`,
  );
  phase("Re-verify the fixed implementation");
  verdict = await verifier.ask<Verdict>(
    `The implementation changed (${impl.notes}). Verify or refute ${PROPERTY} again.`,
  );
}

// 核对员的话只是一种主张；测试文件才是事实。在把结果称为已验证之前，
// 真正跑一遍——核对员可能跑过，但这一次运行才算数。
phase("Run the property tests for real");
const proof = await world.run("npx", ["vitest", "run", TEST_PATH], { timeoutMs: 600_000 });
const verified = verdict.verified && proof.exitCode === 0;
if (verdict.verified && !verified) log(`the verifier said verified but vitest exited ${proof.exitCode}`);

await artifact.markdown(
  "report",
  [
    `# ${PROPERTY}`,
    "",
    verified
      ? `Verified after ${round} round(s); the property tests live in ${verdict.proofPath ?? TEST_PATH} and pass under vitest.`
      : verdict.verified
        ? `The verifier reported verified, but \`npx vitest run ${TEST_PATH}\` exited ${proof.exitCode}; treat the result as unverified.`
        : `Still refuted after ${round} rounds: ${verdict.reason}`,
    "",
    `Implementation: ${IMPL_PATH}. Last change: ${impl.notes}`,
  ].join("\n"),
  {
    title: "Verification report",
    description: "Whether the property holds, how many rounds it took, and where the tests live.",
    primary: true,
  },
);

return {
  conclusion: verified
    ? `${PROPERTY} holds; the verifier could not break the implementation after ${round} round(s), and the property tests pass under vitest.`
    : verdict.verified
      ? `${PROPERTY} is not verified: the verifier reported success but the property tests exit ${proof.exitCode} when run for real.`
      : `${PROPERTY} is still refuted after ${round} rounds: ${verdict.reason}`,
  verified,
  rounds: round,
  implPath: IMPL_PATH,
  proofPath: verdict.proofPath,
  finalVerdict: verdict,
  checks: [`npx vitest run ${TEST_PATH} exited ${proof.exitCode}`],
  notCovered: ["inputs the property tests did not generate", "performance"],
};
```

**为什么这样写。** 两个子代理都活在循环之外，因此各自保留了学到的一切：实现者记得
自己已经试过的思路，核对员记得哪些攻击已经奏效。二者都真实地编辑并运行文件。

子代理之间交换的是**路径与描述，从不交换文件内容**；各自用自己的工具去读对方的
工作。每一轮都在发生时上报，所以一条死在第六轮的运行仍会留下五轮的反例。返回值
里有 `verified` 和 `rounds`，调用者由此能分辨"收敛了"还是"轮数耗尽了"——返回值以
`conclusion` 开头、以 `notCovered` 收尾，因为返回值正是主代理要复述的内容。

`verified` 不是核对员说了算。核对员是子代理，而子代理的"我跑过测试了"只是一种
主张；结尾的 `world.run` 会真刀真枪地跑一遍测试文件，结果就落在它的退出码上。
实现者的角色设定点名了它必须运行的确切命令，出于同样的原因——"跑一下测试"会让
子代理挑最快变绿的办法。

两个产物，各对应一个时刻。指标瓦片是给盯着循环看的人的：轮数与未解决的反例数，
由脚本本来就在调用的 `report` 喂数据。Markdown 报告是给事后看的——与返回值相同的
事实，以用户会留存的长文形式呈现。测试文件本身不发布：它们就在仓库里，为一张
用户反正要在编辑器里打开的文件做卡片，说不出任何新东西。

---

## 2. 不稳定（flaky）测试分诊

先扫描，再在批评之下规划修复，然后独立裁决每一条发现。一个脚本里有三种拓扑，
各自因不同理由而选中。

<!-- compile -->
```ts
interface Finding {
  /** 测试运行器所报告的测试标识符。 */
  testId: string;
  /** 测试文件的工作区相对路径。 */
  file: string;
  /** 是什么让它不稳定。 */
  kind: "timing" | "ordering" | "external";
  /** 0-1。扫描对"它确实不稳定"的把握。 */
  confidence: number;
}
interface Scan {
  findings: Finding[];
}
interface Plan {
  /** 每个测试一条，按修复应施加的顺序排列。 */
  steps: { testId: string; change: string }[];
  /** 计划刻意不去处理的部分。 */
  outOfScope: string[];
}
interface Review {
  approved: boolean;
  /** 计划哪里有问题。批准时为空。 */
  feedback: string;
}
interface Judgement {
  /** 发现为误报时为 false。 */
  real: boolean;
  reason: string;
}
interface Confirmation {
  /** 仅当你亲自复现了不稳定（重跑过测试、读过时序依赖）时才为 true。 */
  reproduced: boolean;
  /** 你做了什么检查，一句话。 */
  note: string;
}
interface ReportedFinding extends Finding {
  /** 复核员复现成功时为 "verified"；无法复现时为 "unconfirmed"。 */
  status: "verified" | "unconfirmed";
}
interface FlakyReport {
  /** 用两三句话回答用户所问。 */
  conclusion: string;
  findings: ReportedFinding[];
  /** 循环收敛时，已批准的计划。 */
  plan?: Plan;
  converged: boolean;
  rounds: number;
  /** 本次运行检查了什么、怎么检查的。 */
  verified: string[];
  /** 本次运行没看什么、查不了什么，以及原因。 */
  notCovered: string[];
}

// 每个候选都是一张卡片；运行对它下判断时，卡片就在列间移动。
artifact.board("candidates", {
  title: "Flaky test candidates",
  key: "testId",
  status: "status",
  columns: ["candidate", "false positive", "verified", "unconfirmed"],
  detail: [{ field: "file" }, { field: "kind" }],
});

phase("Scan the repository for flaky tests");
const scan = await agent("scanner", "You identify flaky tests from test code and CI history.")
  .ask<Scan>("Find flaky tests in this repository. Prefer precision over recall.");

log(`${scan.findings.length} candidate flaky tests`);
for (const finding of scan.findings) report({ ...finding, status: "candidate" }, "candidates");
if (scan.findings.length === 0) {
  const empty: FlakyReport = {
    conclusion: "No flaky tests found.",
    findings: [],
    converged: true,
    rounds: 0,
    verified: ["scanned test code and CI history"],
    notCovered: ["tests that only flake under load the scan could not observe"],
  };
  return empty;
}

// 独立评委：每条发现一个全新上下文，因为见过其他发现的评委
// 会开始按曲线打分，而不是按事实本身。每位评委的名字里都带着
// 测试 id——每条发现一个子代理，就意味着每条发现一个名字。
//
// 评委的裁决是一种判断，而不是复现，所以每个幸存者接下来都会
// 交给一位全新的复核员，由复核员自己重跑测试并阅读时序依赖；
// ask 禁止任何编辑。复核员紧接在自己的评委之后、链在同一个回调内：
// 一个候选在其评委裁定的那一刻就被确认，而不是等所有评委裁定完。
// 复核员无法复现的内容会被保留并打上标签，而不是丢弃。
phase("Judge each candidate and confirm the real ones as they are judged");
// 一次汇合：下面的规划器需要所有真实发现才能开始规划。
const outcomes = await Promise.all(
  scan.findings.map(async (finding): Promise<ReportedFinding | undefined> => {
    const judgement = await agent(`judge-${finding.testId}`).ask<Judgement>(
      `Is this really flaky, or a false positive?\n${JSON.stringify(finding)}`,
    );
    if (!judgement.real) {
      report({ ...finding, status: "false positive" }, "candidates");
      return undefined;
    }
    const check = await agent(`confirmer-${finding.testId}`).ask<Confirmation>(
      `Reproduce this flakiness from the evidence alone: run the test, read the timing dependence. Do not edit any file.\n${JSON.stringify(finding)}`,
    );
    const reported: ReportedFinding = {
      ...finding,
      status: check.reproduced ? "verified" : "unconfirmed",
    };
    report(reported, "candidates");
    return reported;
  }),
);
const real = outcomes.filter((o): o is ReportedFinding => o !== undefined);
log(`${real.length} of ${scan.findings.length} survived judging`);
if (real.length === 0) {
  const empty: FlakyReport = {
    conclusion: "Every candidate was judged a false positive.",
    findings: [],
    converged: true,
    rounds: 0,
    verified: ["scanned test code and CI history", "judged each candidate independently"],
    notCovered: ["tests that only flake under load the scan could not observe"],
  };
  return empty;
}

// 规划器与评审员：只创建一次、每轮复用，从而各自累积上下文。
const planner = agent("planner", "You write minimal, concrete fixes for flaky tests.");
const reviewer = agent("reviewer", "You find the flaw in a fix plan. Approve only when you cannot find one. You never edit files.");

let feedback = "none";
let approved: Plan | undefined;
let rounds = 0;
for (let round = 0; round < 5; round++) {
  rounds = round + 1;
  phase("Draft a fix plan");
  const plan = await planner.ask<Plan>(
    `Plan fixes for these flaky tests:\n${JSON.stringify(real)}\nPrevious critique: ${feedback}`,
  );
  phase("Critique the plan");
  const review = await reviewer.ask<Review>(`Critique this plan:\n${JSON.stringify(plan)}`);
  if (review.approved) {
    approved = plan;
    break;
  }
  log(`plan rejected in round ${rounds}: ${review.feedback}`);
  feedback = review.feedback;
}

const verified = ["scanned test code and CI history", "each survivor reproduced by an independent confirmer"];
const done: FlakyReport = approved
  ? {
      conclusion: `${real.length} flaky tests confirmed; a fix plan was approved after ${rounds} round(s).`,
      findings: real,
      plan: approved,
      converged: true,
      rounds,
      verified,
      notCovered: approved.outOfScope,
    }
  : {
      conclusion: `${real.length} flaky tests confirmed, but no fix plan survived review in ${rounds} rounds; last critique: ${feedback}`,
      findings: real,
      converged: false,
      rounds,
      verified,
      notCovered: ["a fix plan — none was approved"],
    };

await artifact.markdown(
  "report",
  [
    `# ${done.conclusion}`,
    "",
    ...real.map((f) => `- ${f.testId} (${f.file}, ${f.kind}): ${f.status}`),
    "",
    approved
      ? approved.steps.map((step) => `1. ${step.testId}: ${step.change}`).join("\n")
      : `No plan was approved. Last critique: ${feedback}`,
  ].join("\n"),
  { title: "Flaky test report" },
);
return done;
```

**为什么这样写。** 评委对每条发现都是新开的，而规划器/评审员这对被提升到循环外——
相反的选择，都是刻意的。裁决需要独立性，共享上下文会主动污染它；规划需要记忆，
每轮换一个新规划器只会浪费它。复核员是第三种：新开的、逐发现一份、被告知不许
编辑，因为评委的裁决只是对一个描述的意见，只有重跑测试才能把它变成事实。复核
不出来的东西会保留在报告里并标为 `unconfirmed`。每个复核员紧跟自己的评委、处在
同一个回调内，于是扇出只有一次汇合而不是两次：确认第三个候选，永远不用等第四十个
候选的评委。

两个提前返回都有讲究。一无所获的扫描与全盘否决的裁决，都是结果为空的*成功*，
而非失败——而且都避免为一张空列表付出规划循环的代价。二者仍返回完整的报告形态，
`conclusion` 会说明发生的是哪一种。发现在规划开始之前就已上报，所以哪怕规划循环
永远不收敛，已确认的发现也会交回来。

看板是扇出的观察者视图：扫描一返回，每个候选立刻出现，并在评委或复核员下判断时
移动列，因为后面同 `testId` 的条目会替换掉那张卡片。报告在规划循环之后只发布
一次，这也是循环用 `break` 走向单一 return、而不是从循环内部 return 的原因。两个
提前返回什么都不发布："没发现不稳定测试"是一行字就能回答的事，再用一页去复述
它就是噪音。

---

## 3. 调用点迁移

先发现，再并行改写，然后验证。改写的子代理要写文件，所以这是唯一一个隔离与验证
真正要紧的形态。

<!-- compile -->
```ts
interface Migration {
  /** 这个文件是否本来就需要修改。 */
  changed: boolean;
  /** 改写了什么，一行话。 */
  summary: string;
  /** 迁移器无法安全处理而放过的部分。 */
  skipped: string[];
}
interface Check {
  /** 修改之后，文件能编译且测试通过时为 true。 */
  clean: boolean;
  /** 不通过时，坏在哪。 */
  problem: string;
}

const OLD_API = "getUserSync";
const NEW_API = "getUser";

artifact.table("migration", {
  title: "Migration by file",
  key: "path",
  columns: [
    { field: "path", label: "File" },
    { field: "summary", label: "Change" },
    { field: "clean", label: "Verified" },
  ],
});

phase("Find every call site of the old API");
const hits = await files.grep(`\\b${OLD_API}\\b`, "src/**/*.ts");
const paths = [...new Set(hits.map((h) => h.path))];
log(`${OLD_API} appears in ${paths.length} files (${hits.length} call sites)`);
if (paths.length === 0) return { migrated: [], failed: [], note: "nothing to migrate" };

// 一份角色设定，被所有迁移子代理复用。角色设定是可以共享的值；子代理
// 及其名字不是——每个文件各得一份。
const MIGRATOR = {
  system:
    `You migrate call sites from ${OLD_API} to the async ${NEW_API}, adding await and making ` +
    "the enclosing function async where needed. Change nothing else. If a call site cannot be " +
    "migrated safely, leave it and say why.",
};

// 每个文件一个迁移器：各文件的修改互不相交，因此可以同时进行，
// 而每文件一个全新上下文，能让一个文件的怪癖不泄漏进另一个文件的重写。
phase("Migrate each file and verify the result");
// 一次汇合：下面的整套树级测试需要每个文件先迁移完。
const results = await Promise.all(
  paths.map(async (path) => {
    try {
      const migration = await agent(`migrator-${path}`, MIGRATOR).ask<Migration>(
        `Migrate every ${OLD_API} call in ${path} to ${NEW_API}.`,
      );
      if (!migration.changed) return { path, migration, check: undefined };

      // 用独立上下文验证：让子代理检查自己的工作，它会打分从宽。
      // 它编译并运行测试；ask 禁止它编辑。
      const check = await agent(`checker-${path}`).ask<Check>(
        `Does ${path} still compile and pass its tests after the ${NEW_API} migration? ` +
          `Run the checks; do not edit any file. The change was: ${migration.summary}`,
      );
      report({ path, summary: migration.summary, clean: check.clean }, "migration");
      return { path, migration, check };
    } catch (error) {
      log(`skipped ${path}: ${String(error)}`);
      report({ path, summary: `skipped: ${String(error)}`, clean: false }, "migration");
      return { path, migration: undefined, check: undefined, error: String(error) };
    }
  }),
);

const migrated = results.filter((r) => r.check?.clean === true).map((r) => r.path);
const failed = results.filter((r) => r.check?.clean === false || r.error !== undefined);

// 每个文件都通过了自己的检查；现在轮到整棵树，只来一次。逐文件检查器
// 看不到另一个文件里被迁移签名弄坏的调用方——完整测试套件看得到，
// 而且这正是用户自己会跑的那次检查。
phase("Run the whole test suite once over the migrated tree");
const suite = await world.run("npm", ["test"], { timeoutMs: 1_800_000 });
const suiteClean = suite.exitCode === 0;
if (!suiteClean) log(`npm test failed after the migration:\n${suite.stderr.slice(-2000)}`);

await artifact.markdown(
  "report",
  [
    suiteClean
      ? `# ${OLD_API} → ${NEW_API}: ${migrated.length} of ${paths.length} files migrated and verified`
      : `# ${OLD_API} → ${NEW_API}: ${migrated.length} of ${paths.length} files migrated, but the full suite fails`,
    "",
    ...migrated.map((p) => `- ${p}: migrated; compiled and tests passed`),
    ...failed.map((r) => `- ${r.path}: failed — ${r.check?.problem ?? r.error}`),
    "",
    `Full suite (\`npm test\`): exit ${suite.exitCode}`,
  ].join("\n"),
  { title: "Migration report" },
);

return {
  conclusion: suiteClean
    ? `${migrated.length} of ${paths.length} files migrated from ${OLD_API} to ${NEW_API} and verified; ${failed.length} failed; the full suite passes.`
    : `${migrated.length} of ${paths.length} files migrated from ${OLD_API} to ${NEW_API}, but the full suite fails after the migration, so the result is not verified: ${suite.stderr.slice(-300)}`,
  migrated,
  failed: failed.map((r) => ({ path: r.path, problem: r.check?.problem ?? r.error })),
  skipped: results.flatMap((r) => r.migration?.skipped ?? []),
  verified: [
    ...migrated.map((p) => `${p}: compiled and tests passed, checked by a separate subagent`),
    `npm test over the whole tree exited ${suite.exitCode}`,
  ],
  notCovered: ["call sites outside src/**/*.ts", "callers in other packages that import the migrated functions"],
};
```

**为什么这样写。** `files.grep` 在扇出之前先收窄：如果对 `src/**/*.ts` 做通配、再
逐个文件问它用没用旧 API，等于每个文件花掉一个会话，只为弄清一次搜索早已知道的
事。因为 `grep` 超出上限时是拒绝而不是截断，路径列表要么完整、要么没有——绝不会
悄悄缺一角，而对迁移来说，这正是"做完了"与"悄悄做了一半"之间的差别。

扇出中的两个子代理都按路径命名。这不是装饰：一次运行会拒绝两个同名的子代理，所以
`map` 里的 `agent("migrator")` 活不过第二个文件——而且按路径命名的名字，也是修正
后的重跑（`AmendWorkflow`）匹配其导入缓存所依据的键，因此修好一处细节后重跑本脚本，
不会把已经干净的文件再迁移一遍。

检查器与迁移器是**各自独立的子代理**，这正是检查有意义的原因：让子代理给自己做的
迁移打分，它会打得很宽。它负责构建和测试，而 ask 明令禁止它修补发现的问题。检查器
还与自己的迁移器同处一个回调——一条逐条目的流水线、一次汇合——于是第一个文件在被
验证时，第四十个可能还在重写，而不是所有检查器都排在 `Promise.all` 后面等最慢的
那个迁移。`try`/`catch` 是逐文件的，于是一个迁不动的文件只损失一个文件，而不是
整条运行。返回值把 `migrated`、`failed`、`skipped` 分开，因为"我们改了 40 个文件"
不是任何人能据以行动的结论。

扇出验证每个文件；其后的 `world.run` 验证整棵树。这是两种不同的检查：四十个各自
测试通过的文件，合在一起仍可能编译不过，而某个没人迁移的文件里的调用方，恰恰是
逐文件检查器永远不会打开的东西。整套测试的运行就是用户合并前会自己跑的那次检查，
所以它在这里跑、只跑一次、带上真实测试所需的超时——一旦失败，`conclusion` 会明说
结果未经验证，而不是让四十行绿色替整棵树代言。

表格是扇出的实时视图：每文件一行、以路径为键、随各检查器应答而填入，被跳过的文件
也会落进表里。结尾的报告是同一份清单的文字版，供用户留存。迁移后的文件本身不
发布——它们就是仓库，用户在仓库里读它们。"无可迁移"的提前返回什么都不发布：
一行字的回答不需要一页纸。

---

## 4. 由真实检查器把关的证明循环

一个子代理做开放式修复，外加一扇无法用言语糊弄过去的门禁：`world.run` 执行真正的
证明检查器，循环凭它的退出码推进，而不凭任何人的主张。对比示例 1——那里的怀疑者
是另一个子代理，那在击破工作需要创造力时才合适。当一条命令能给出裁决时，就该让
命令来给。

<!-- compile -->
```ts
interface FixNotes {
  /** 本轮改了什么，一行话。 */
  summary: string;
}

const TARGET = "Proofs/Main.lean";
const MAX_ROUNDS = 10;

const prover = agent("prover", {
  system:
    `You write and repair Lean proofs in ${TARGET}. Fix exactly what the checker reports; ` +
    "never delete or weaken a theorem to silence an error.",
});

phase("Write the first proof attempt");
let notes = await prover.ask<FixNotes>(`Prove the open theorem in ${TARGET}.`);

const attempts: string[] = [];
let clean = false;
let rounds = 0;
for (let round = 1; round <= MAX_ROUNDS; round++) {
  rounds = round;
  attempts.push(notes.summary);
  phase("Check the file with the fast checker");
  // 快速档：单文件，几秒钟。对一个仍写着 `sorry` 的证明它也会退出 0，
  // 只在 stderr 上告警，所以单凭退出码并不是裁决。
  const check = await world.run("lake", ["env", "lean", TARGET], { timeoutMs: 600_000 });
  clean = check.exitCode === 0 && !check.stderr.includes("sorry");
  if (clean) {
    report({ round, summary: notes.summary, checkerClean: true });
    break;
  }

  log(`round ${round}: checker rejected`);
  report({ round, summary: notes.summary, checkerClean: false });
  phase("Repair what the checker rejected");
  notes = await prover.ask<FixNotes>(
    `The Lean checker rejected the file. Its output:\n${check.stderr}\nRepair ${TARGET}.`,
  );
}

// 强档，只跑一次：整个项目带上新证明一起构建。这才是请求关心的事，
// 即便快速档已经说了通过，它也要跑。
let proved = false;
if (clean) {
  phase("Build the whole project with the new proof");
  const build = await world.run("lake", ["build"], { timeoutMs: 1_800_000 });
  proved = build.exitCode === 0;
  if (!proved) log(`lake build rejected what the fast checker accepted:\n${build.stderr}`);
}

await artifact.markdown(
  "report",
  [
    proved
      ? `# ${TARGET}: proved in round ${rounds}`
      : `# ${TARGET}: not proved after ${MAX_ROUNDS} rounds`,
    "",
    ...attempts.map((summary, index) => `${index + 1}. ${summary}`),
  ].join("\n"),
  { title: "Proof report" },
);

return proved
  ? {
      conclusion: `The open theorem in ${TARGET} is proved; the file checked clean in round ${rounds} and the whole project builds.`,
      proved: true,
      rounds,
      lastChange: notes.summary,
      verified: [`lake env lean ${TARGET} exited 0 with no sorry warning`, "lake build exited 0"],
      notCovered: [],
    }
  : clean
    ? {
        conclusion: `Not proved: ${TARGET} checks clean on its own, but lake build fails with the new proof in the project.`,
        proved: false,
        rounds,
        lastChange: notes.summary,
        verified: [`lake env lean ${TARGET} exited 0 with no sorry warning`, "lake build exited nonzero"],
        notCovered: ["why the whole-project build disagrees with the single-file check"],
      }
    : {
        conclusion: `Not proved: the Lean checker still rejects ${TARGET} after ${MAX_ROUNDS} rounds.`,
        proved: false,
        rounds,
        lastChange: notes.summary,
        verified: [`lake env lean ${TARGET} run ${MAX_ROUNDS} times, none clean`],
        notCovered: ["approaches the prover did not try", "lake build — never reached"],
      };
```

**为什么这样写。** 证明者只创建一次并保留上下文，于是第七轮还记得前六轮试过什么。
检查器根本不是子代理：`world.run` 跑真正的命令，非零退出以**值**的形式返回，循环的
常态就是读 `exitCode`、把 `stderr` 作为下一轮的指令交给证明者——全程没有 `catch`，
因为拒绝（rejection）只留给世界本身不应答的情形（spawn 失败、超时、输出超限）。

这两次检查不可互换。对单文件的 `lake env lean` 是快速档：它驱动各轮，因为几秒内
就有答案；它同时读 stderr 和退出码，因为一个仍写着 `sorry` 的证明会以 0 退出并
附带告警。覆盖整个项目的 `lake build` 是强档——真正与请求相关的检查——无论快速档
是否已通过，它在结尾都要跑一次。一个只由快速档把关的循环，会对一个仍含 `sorry`
的文件返回"已证明"，并且永远不会得知新证明弄坏了某个导入它的模块。由哪两条命令
扮演这两个角色，是仓库自身的事实，要在写脚本之前从它的 README 或构建文件里读
出来，而不是从上一个项目带来的习惯。

命令名是字面量，变化的部分——目标路径——放在 args 数组里；这是规则，也是用户在
确认时看到并批准的东西。长 `timeoutMs` 是刻意的：证明检查被允许很慢，逐调用覆盖
正是为真实构建量级的检查而存在的。每一轮在落定时就被上报，所以死在第八轮的运行
仍会留下七轮尝试。在接上另一个检查器命令之前，先用一次一行的 `EvalWorkflowSnippet`
调用看看它的 stderr 到底长什么样——你照着猜测写出的解析，就是那个会坏掉的解析。

只有一个产物，是刻意的。报告列出每一次尝试，比 `conclusion` 说得更多，所以它配得
上那一页。没有仪表盘：观察者能跟随的唯一状态就是轮数与检查器是否通过，而阶段
时间线再次点亮"修复检查器拒绝之处"时，已经把这件事展示得分毫不差。指标瓦片只会
重复它，证明文件归仓库展示，不归卡片。

## 5. 变更文件评审：随评审落地即时确认

评审每个变更文件，在同一把尺子上对发现做分诊，独立确认每条保留下来的发现，最后
交回一份报告。一个文件需要的一切，在它自己的评审落地时就发生；唯一的汇合是需要
全部发现的跨文件去重。

<!-- compile -->
```ts
interface Finding {
  /** 工作区相对路径，适用时带上行号："src/a.ts:42"。 */
  where: string;
  /** 一句话：哪里不对。 */
  what: string;
  /** 证明它的东西：读过的行，或命令与证明它的输出。 */
  evidence: string;
  /** 有多重要。"high" 只留给数据丢失、崩溃或错误结果。 */
  severity: "low" | "medium" | "high";
}
interface Review {
  findings: Finding[];
}
interface Keep {
  /** 这条发现值得摆到人类面前时为 true。 */
  keep: boolean;
}
interface Confirmation {
  /** 仅当你依据证据亲自复现了问题时才为 true。 */
  reproduced: boolean;
  /** 你做了什么检查，一句话。 */
  note: string;
}
interface ReportedFinding extends Finding {
  /** 复核员复现成功时为 "verified"；无法复现时为 "unconfirmed"。 */
  status: "verified" | "unconfirmed";
}
interface Digest {
  /** 已确认的发现，跨文件去重已合并，每条一行。 */
  lines: string[];
  /** 两三句话，读者可据以行动。 */
  summary: string;
}
interface WorkflowReport {
  /** 用两三句话回答用户所问。 */
  conclusion: string;
  findings: ReportedFinding[];
  /** 本次运行检查了什么、怎么检查的。 */
  verified: string[];
  /** 本次运行没看什么、查不了什么，以及原因。 */
  notCovered: string[];
}

phase("Review each changed file and confirm its findings as they land");
let paths: string[];
try {
  paths = await git.changedFiles("origin/main");
} catch {
  paths = await files.glob("src/**/*.ts");
}
log(`reviewing ${paths.length} changed files`);

// 所有发现共用一个分诊子代理：严重度只有在一把一致的尺子上评判才有意义。
// 它是一个队列，不是屏障——发向它的 ask 按 FIFO 执行，所以一条发现
// 在其所属文件的评审落地的那一刻就到达这里，无论那是哪个文件。
const triage = agent("triage", "You decide which review findings deserve a human's attention. Be strict.");

// 每个文件一个评审员，各自全新：文件之间互不相关，共享上下文毫无收益，
// 而同时运行收益全拿。分诊与确认都活在同一个回调内，所以任何文件
// 都不必等最慢的那份评审结束才让自己的发现继续前进；外层的 `Promise.all`
// 是报告唯一等待的汇合。
const perFile = await Promise.all(
  paths.map(async (p) => {
    const review = await agent(`reviewer-${p}`).ask<Review>(`Review ${p} for correctness bugs.`);

    const kept: Finding[] = [];
    for (const finding of review.findings) {
      const verdict = await triage.ask<Keep>(`Worth reporting? ${JSON.stringify(finding)}`);
      if (verdict.keep) kept.push(finding);
    }

    // 每条保留的发现配一个独立复核员，对提出它的评审员全然不知情：它读
    // 代码，如果有现成检查就跑一次，并被叮嘱什么都不许修。名字里带上
    // 路径和序号，因为一次运行中的每个子代理名字都必须唯一。
    return Promise.all(
      kept.map(async (finding, index) => {
        const check = await agent(`confirmer-${p}-${index}`).ask<Confirmation>(
          `Reproduce this finding from its evidence alone: read the code, run a check if one exists. Do not edit any file.\n${JSON.stringify(finding)}`,
        );
        const reported: ReportedFinding = {
          ...finding,
          status: check.reproduced ? "verified" : "unconfirmed",
        };
        report(reported); // 立即发布，连同状态，这样它能在之后的失败中幸存
        return reported;
      }),
    );
  }),
);
const confirmed = perFile.flat();
log(`${confirmed.length} findings confirmed or labelled`);

// 真正需要全部发现的唯一一个阶段：两个评审员可能从两个文件里
// 标记出同一个根因，只有读过整份清单的人才能把它们合并。
phase("Merge duplicate findings across files and write them up");
const digest = await agent("editor", "You merge review findings that share a root cause and write them up for the engineer who fixes them.")
  .ask<Digest>(`Merge duplicates and write these up:\n${JSON.stringify(confirmed)}`);

const verifiedCount = confirmed.filter((f) => f.status === "verified").length;
await artifact.markdown(
  "report",
  [
    `# Review of ${paths.length} changed files: ${confirmed.length} findings, ${verifiedCount} reproduced`,
    "",
    digest.summary,
    "",
    ...digest.lines.map((line) => `- ${line}`),
    "",
    "## Every finding",
    ...confirmed.map((f) => `- **${f.where}** (${f.severity}, ${f.status}): ${f.what}\n  ${f.evidence}`),
  ].join("\n"),
  { title: "Review report" },
);
const result: WorkflowReport = {
  conclusion: digest.summary,
  findings: confirmed,
  verified: paths.map((p) => `reviewed ${p}; every kept finding re-checked by a separate subagent`),
  notCovered: ["files outside the change set", "runtime behaviour no existing test exercises"],
};
return result;
```

一个脚本里有三种形态，各有其理由。评审员是新开的且并行，因为文件彼此无关。分诊
子代理是共享的，因为严重度必须两次都意味着同一件事——而在这里共享毫无代价，因为
它的 FIFO 队列是随评审落地而被投喂的，而不是等所有评审结束。复核员是新开的、逐
发现一份、对评审员全然不知情，并且在自己文件的分诊一完成就开工：十二个文件配一个
慢评审员时，其余十一个文件的发现已在它还在阅读时被确认并上报。没有任何发现再叠
一层第二读者：每条发现都已被确认，而编辑的职责是合并与成文，不是复查。

## 6. 事实核查流水线：每条事实一个新核对员

同时从五个角度研究一个主题，每条事实一到达就交给一个没有做过研究的核对员去查，
然后从站得住的事实写出报告。研究到核查的交接是一条通道，于是第一个核对员开工时
四位研究者还在阅读；写手坐在唯一一次汇合之后，因为它需要全部经核实的事实。

<!-- compile -->
```ts
interface Fact {
  /** 由角度加序号构成的稳定 id："tests-3"。 */
  id: string;
  /** 一句可以对照仓库检验的话。 */
  claim: string;
  /** 研究者在哪里看到的它：带行号的路径，或命令及其输出。 */
  source: string;
  /** 它来自哪个研究角度。 */
  angle: string;
}
interface Research {
  facts: { claim: string; source: string }[];
}
interface Verdict {
  /** 仅当你依据给定的来源亲自确认了该主张时才为 true。 */
  holds: boolean;
  /** 你做了什么检查，一句话。 */
  note: string;
}
interface Finding {
  /** 事实在何处：核对员确认过的来源。 */
  where: string;
  /** 事实本身，一句话。 */
  what: string;
  /** 核对员如何确认的。 */
  evidence: string;
  /** 这里的每条发现都由一个没有做过研究的核对员确认过。 */
  status: "verified";
}
interface Draft {
  /** 报告正文，Markdown，给用户。 */
  markdown: string;
  /** 两三句话，读者可据以行动。 */
  summary: string;
}
interface WorkflowReport {
  /** 用两三句话回答用户所问。 */
  conclusion: string;
  findings: Finding[];
  /** 本次运行检查了什么、怎么检查的。 */
  verified: string[];
  /** 本次运行没看什么、查不了什么，以及原因。 */
  notCovered: string[];
}

const topic = String(args.topic ?? "this repository");
const angles = ["architecture", "data flow", "error handling", "tests", "history"];

// 研究与核查之间的交接：还在阅读的研究者继续发送，而核对员已经在
// 处理已到达的内容。研究阶段负责关闭。
const facts = channel<Fact>("facts");
const passed: Finding[] = [];
const rejected: string[] = [];

const research = future(async () => {
  phase("Research the topic from five angles");
  try {
    await Promise.all(
      angles.map(async (angle) => {
        const found = await agent(
          `researcher-${angle}`,
          "You research a codebase and state only what you saw, each fact with the place you saw it.",
        ).ask<Research>(
          `Research ${topic} from the angle of ${angle}. Return up to eight facts, each with its source.`,
        );
        found.facts.forEach((fact, index) => facts.send({ ...fact, id: `${angle}-${index + 1}`, angle }));
        log(`${angle}: ${found.facts.length} facts found`);
      }),
    );
  } finally {
    facts.close(); // 失败的研究者也会为核对员结束这条数据流
  }
});

const verify = future(async () => {
  phase("Check each fact as it arrives");
  // 每条事实一个全新核对员，以它命名：修订后的运行中，缓存身份按事实保留，
  // 而名字说明了每张卡片讲的是哪条事实。
  const checks: Promise<void>[] = [];
  for await (const fact of facts) {
    checks.push(
      future(async () => {
        const verdict = await agent(
          `verifier-${fact.id}`,
          "You check one claim about a codebase against its source. Do not edit any file.",
        ).ask<Verdict>(`Does this hold? Read the source yourself.\n${JSON.stringify(fact)}`);
        if (verdict.holds) {
          const finding: Finding = { where: fact.source, what: fact.claim, evidence: verdict.note, status: "verified" };
          passed.push(finding);
          report(finding); // 立即发布，这样它能在之后的失败中幸存
        } else {
          rejected.push(`${fact.claim} (${verdict.note})`);
        }
      }),
    );
  }
  // 汇合才是告诉这个阶段"你完成了"的东西：没有它，阶段会在通道关闭时
  // 就结束，此时检查还在跑、一半的结果还没回来。
  await Promise.all(checks);
});

// 先启动两者，再汇合一次。中间没有 await：当脚本在等待其中一个阶段时，
// 另一个阶段里的失败没有监听者，会以未处理拒绝（unhandled rejection）的
// 形式浮出水面，而不是带着错误让运行失败。
await Promise.all([research, verify]);
log(`${passed.length} facts verified, ${rejected.length} did not hold`);

phase("Write the report from the verified facts");
const draft = await agent(
  "writer",
  "You write short technical reports for engineers from verified facts only.",
).ask<Draft>(`Write a report on ${topic} from these verified facts:\n${JSON.stringify(passed)}`);
await artifact.markdown("report", draft.markdown, { title: `What we know about ${topic}` });

const result: WorkflowReport = {
  conclusion: draft.summary,
  findings: passed,
  verified: [`${passed.length} facts each re-checked against their source by a separate subagent`],
  notCovered: rejected.length > 0 ? rejected.map((claim) => `did not hold on re-check: ${claim}`) : [],
};
return result;
```

两个阶段以并发阶段（future）呈现，一条通道，一次汇合。每个 future 以自己的阶段
标记开场，而标记按词法归属，所以同时运行的两个阶段在时间线上各留各的步骤。研究
阶段把 `facts.close()` 放在 `finally` 里：失败的研究者同样会结束数据流，于是核对
员能收工，而不是停在一条再也没人发送的通道上。核对员是按事实新开并命名的，而不是
池化复用，因为事实彼此独立，且修订后的运行应当只对那一条变了的事实失去缓存，而不
是对整个池。`checks` 数组是最容易被忘掉的部分：核查阶段在它的核对员都完成时才算
完成，而不是在通道关闭时；而死锁检测器抓不住一个带着仍在飞行的工作提前返回的阶段。

---

## 7. 通过尾部洞一步步长出的仓库导览

"给这个仓库写一篇导览"这个任务，在仓库被实际看过之前根本没有工作流可言。所以
脚本只做它能规划的那一件事——勘察——然后停在一个尾部洞里。两次填充之后，这次
运行有了六个章节写手、一名独立读者、一名修订者和一篇已发布的导览，而种子脚本
没有点名其中任何一个。（`SKILL.md` §15。）

种子脚本，随 `CreateWorkflow` 一起提交：

<!-- compile -->
```ts
interface Dir {
  /** 顶层条目的工作区相对路径。 */
  path: string;
  /** 一句话：它是干什么用的。 */
  purpose: string;
}
interface Survey {
  dirs: Dir[];
}
interface Chapter {
  title: string;
  /** 章节实际覆盖的路径。 */
  covers: string[];
  /** 正文，Markdown，写给首次阅读的读者。 */
  body: string;
}
interface Guide {
  title: string;
  chapters: Chapter[];
  notCovered: string[];
}

phase("盘点仓库顶层目录");
const surveyor = agent("勘察员", "你快速摸清一个仓库的结构：实际查看目录和关键文件，如实汇报。");
const survey = await surveyor.ask<Survey>("列出顶层目录（跳过 .venv、.zcode 等产物），各用一句话说明作用。");
log(`盘点完成：${survey.dirs.length} 个条目`);

return await hole<Guide>(
  "第1步：决定导览结构",
  `盘点：\n${survey.dirs.map((d) => `- ${d.path}：${d.purpose}`).join("\n")}`,
);
```

运行抵达洞口，通知把勘察结果带在提示里。第一次填充（只含语句，作为 `script`
发送）决定章节划分，按目录扇出写手，并留下下一个洞：

```ts
phase("分章并行撰写初稿");
// 下一步的批评者要一次读完所有章，所以这次汇合要等全部完成。
const drafts = await Promise.all(
  survey.dirs.map((d) =>
    agent(`写手·${d.path}`, "你只写自己打开核实过的内容。").ask<Chapter>(
      `写《仓库导览》的「${d.path}」一章（200–400 字 Markdown）：${d.purpose}。covers 如实填写。`,
    ),
  ),
);
log(`${drafts.length} 章初稿完成`);
return await hole<Guide>(
  "第2步：审读与交付",
  `${drafts.length} 章初稿已就绪：${drafts.map((d) => d.title).join("、")}`,
);
```

第二次填充从外围作用域读到 `drafts`，加上一轮独立审读和一次修订，发布导览并
返回它，链条就此闭合：

```ts
phase("独立审读初稿");
const critique = await agent("审读员", "你只依据文本挑毛病，不读仓库，不夸奖。").ask<string>(
  `逐章挑毛病：\n${drafts.map((d) => `## ${d.title}\n${d.body}`).join("\n\n")}`,
);
phase("按审读意见修订并交付");
const chapters = await agent("修订编辑").ask<Chapter[]>(
  `按意见修订，保持章序：${critique}\n\n${JSON.stringify(drafts)}`,
);
const guide: Guide = { title: "仓库导览", chapters, notCovered: [".venv、.zcode 等产物目录未纳入"] };
await artifact.markdown("guide", chapters.map((c) => `## ${c.title}\n\n${c.body}`).join("\n\n"), {
  title: guide.title,
  description: "按目录分章的仓库导览。",
  primary: true,
});
return guide;
```

两次填充之后，运行的草稿里保存着生效脚本——种子脚本、每个洞的最后一位参数处拼接
进各自的函数体——恢复（resume）重放的就是它，修订（amend）改的也是它：

<!-- compile -->
```ts
interface Dir {
  /** 顶层条目的工作区相对路径。 */
  path: string;
  /** 一句话：它是干什么用的。 */
  purpose: string;
}
interface Survey {
  dirs: Dir[];
}
interface Chapter {
  title: string;
  /** 章节实际覆盖的路径。 */
  covers: string[];
  /** 正文，Markdown，写给首次阅读的读者。 */
  body: string;
}
interface Guide {
  title: string;
  chapters: Chapter[];
  notCovered: string[];
}

phase("盘点仓库顶层目录");
const surveyor = agent("勘察员", "你快速摸清一个仓库的结构：实际查看目录和关键文件，如实汇报。");
const survey = await surveyor.ask<Survey>("列出顶层目录（跳过 .venv、.zcode 等产物），各用一句话说明作用。");
log(`盘点完成：${survey.dirs.length} 个条目`);

return await hole<Guide>(
  "第1步：决定导览结构",
  `盘点：\n${survey.dirs.map((d) => `- ${d.path}：${d.purpose}`).join("\n")}`, async () => {
  phase("分章并行撰写初稿");
  // 下一步的批评者要一次读完所有章，所以这次汇合要等全部完成。
  const drafts = await Promise.all(
    survey.dirs.map((d) =>
      agent(`写手·${d.path}`, "你只写自己打开核实过的内容。").ask<Chapter>(
        `写《仓库导览》的「${d.path}」一章（200–400 字 Markdown）：${d.purpose}。covers 如实填写。`,
      ),
    ),
  );
  log(`${drafts.length} 章初稿完成`);
  return await hole<Guide>(
    "第2步：审读与交付",
    `${drafts.length} 章初稿已就绪：${drafts.map((d) => d.title).join("、")}`, async () => {
    phase("独立审读初稿");
    const critique = await agent("审读员", "你只依据文本挑毛病，不读仓库，不夸奖。").ask<string>(
      `逐章挑毛病：\n${drafts.map((d) => `## ${d.title}\n${d.body}`).join("\n\n")}`,
    );
    phase("按审读意见修订并交付");
    const chapters = await agent("修订编辑").ask<Chapter[]>(
      `按意见修订，保持章序：${critique}\n\n${JSON.stringify(drafts)}`,
    );
    const guide: Guide = { title: "仓库导览", chapters, notCovered: [".venv、.zcode 等产物目录未纳入"] };
    await artifact.markdown("guide", chapters.map((c) => `## ${c.title}\n\n${c.body}`).join("\n\n"), {
      title: guide.title,
      description: "按目录分章的仓库导览。",
      primary: true,
    });
    return guide;
  },
  );
},
);
```

值得注意的几点：每一步的洞都有自己的名字，因为洞就是一个阶段，两个同名的洞会并成
一个阶段；提示只带下一位作者需要的东西，因为数据由绑定携带；链条在第二步闭合，
是因为作者判断已经完成，而不是因为什么被用尽了。写手在 `map` 内按目录命名，因此
修订后的运行若只改了一个目录的章节，只有那一位写手失去缓存。

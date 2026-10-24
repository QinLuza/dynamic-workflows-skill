# 工具参考（SKILL.md §16）

> 本文是 SKILL.md 的第 16 节，单独拆出来按需读取：写或修订脚本、片段、洞的填充之前必读。
> §1–§15 的判断层仍在 `SKILL.md`。

## 16. 工具参考

五件编写工具——`CreateWorkflow`、`AmendWorkflow`、`FillWorkflowHole`、`SaveWorkflow`、`EvalWorkflowSnippet`——的说明刻意写得很短，而且每件都拒绝运行直到本会话加载了本技能。本节是那些说明没有承载的契约部分：编译器检查所对照的 facade、脚本必须满足的规则、每个工具接受的字段。写或修订脚本之前先读它；编译器会照着执行。

### 16.1 `CreateWorkflow`

对脚本做类型检查；编译干净后请用户确认，并在后台启动运行；运行落定时用脚本最终 `return` 值通知你。编译错误以诊断形式返回，按脚本文件计数（§13）。

**三个来源，只传一个。**

- `script`：内联写的一次性工作流。保存到 `.zcode/workflow-drafts/` 下的文件，结果给出该路径（无论编译与否）；改文件、用 `path` 重新提交来修订（§13）。
- `saved`：本项目或全局保存的工作流，按名字——`saved: { name: "pr-review", args: { pr: "123" } }`；参数在运行前按声明校验。从零写之前先查 `ListSavedWorkflows`：用户留下的已保存工作流胜过重建。运行已保存工作流是唯一不需要加载本技能的调用。
- `path`：磁盘上的脚本文件，相对工作目录或绝对路径——通常是先前结果的路径。文件在 `/* zcode-workflow` 块里声明了参数时，一并传 `args`。

无论哪种来源，用户确认运行，确认界面展示实际脚本。

**字段。**

- `name`：运行标签，用用户的语言（“PR 评审”“代码评审”）。内联脚本总是传；它给运行打标签，也是草稿文件名。缺省用已保存工作流的名字。
- `max_concurrency`：同时工作的子代理上限。只在用户要求限制并发时设置——绝不要因为 provider 限流或报错而设，运行时会自行适配。
- `subagent_model`：子代理运行的模型，`providerId/modelId` 或裸模型 id（可选 `$reasoningLevel`）。只在用户点名模型时设置；用用户给的名字，工具无法解析时从它列出的 id 或 `ListModels` 里挑。**绝不要**把子代理放到用户没点名的另一个 provider 的模型上：裸名字只在会话的 provider 内解析，跨 provider 要写全 `providerId/modelId`，而那只在用户点名那个模型时才写。如果你认为别处更便宜/更快的模型会有帮助，调用前先问用户——模型、provider、能省什么——把用户的“是”当作请求。无论如何你自己留在会话模型上。

这个工具启动新运行。要改动已存在的运行——errored、completed、stopped 或仍在跑——调 `AmendWorkflow`（§13、§16.4），绝不要再次 `CreateWorkflow`。

### 16.2 facade

每个脚本和片段都会对照这些声明做检查。它们就是喂给编译器的同一份常量，所以你读到的就是要类型检查的。这个块使用 `declare`，因为它是宿主的 ambient API 描述——不要模仿它；你的脚本定义普通 interface，永远不用 `declare`。

<!-- facade-dts:start -->
```ts
/** 一次 ask 的待定结果。await 它，或与其他一起传给 Promise.all。 */
declare interface Node<T> extends PromiseLike<T> {}

/** 用 model() 声明的模型。 */
declare class ModelRef { private constructor(); private readonly modelRef: never }

/**
 * 声明子代理可用的模型："modelId" 或 "providerId/modelId"，可选 "$level"。
 * 只能字面量（启动时检查）；运行期在多个 ModelRef 中选择。
 */
declare function model(id: string): ModelRef;

/** 子代理的角色设定，在 agent() 创建时冻结。每个子代理拥有相同的工作工具。 */
declare interface AgentPersona {
  /** 描述角色行为的系统提示。 */
  system?: string;
  /** 模型 id 字面量或 ModelRef；缺省 = 本次运行的 subagent_model。 */
  model?: ModelRef | string;
}

/**
 * 一个子代理：一段对话，跨 ask 保留上下文。子代理忙时发出的 ask 按先进先出排队。
 */
declare interface Agent {
  /**
   * 给子代理一个任务。带类型参数时，答案是 T 类型的值，按从 T 构造的 schema 校验：
   * T 是你在脚本里用普通 "interface" 声明定义的接口（不要 "declare" 修饰符）。
   * 不带类型参数时，答案是子代理的最终回复文本。
   */
  ask<T = string>(instructions: string): Node<T>;
}

/**
 * 创建子代理。每次调用打开新的空白对话；要共享上下文就共享返回的值。
 * 非空名字是身份：在一次运行内唯一（字面量重复是编译错误，运行期拼出的会失败），
 * 也是 AmendWorkflow 匹配缓存工作的键。匿名允许，且永远不复用缓存。
 */
declare function agent(name?: string, persona?: string | AgentPersona): Agent;

/**
 * 运行参数。已保存工作流的参数在运行前按声明校验并填默认值；内联脚本或片段得到 {}。
 * 永远已定义；值是 unknown，要收窄（String(args.target)、typeof 守卫）。
 */
declare const args: Readonly<Record<string, unknown>>;

/** 向用户发一条进度消息。 */
declare function log(message: string): void;

/**
 * 运行进行中发布一条中间结果；无返回值。条目有记录：恢复的运行不会重复显示，
 * 运行失败时也会随完成通知送达。条目必须可 JSON 序列化（普通对象、数组、字符串、
 * 数字、布尔、null）；函数、类实例、Date、promise 是编译错误。上限会让整个运行失败，
 * 不是单次调用失败：每次运行 65,536 条、单条序列化后 1 MiB、每次运行 1 GiB。
 *
 * artifactId 把条目路由到看板：编译期字面量，指向一个届时已执行的预设
 * （artifact.chart / table / metrics / board）。条目在那里成为一个点、一行、一个磁贴值或一张卡，
 * 同时仍落在运行结果里。标签指不到东西，或指向 file/markdown 产物，会让运行失败。
 */
declare function report(item: unknown, artifactId?: string): void;

/** 已发布产物的一个版本：发布时用的 id，以及本次调用铸出的版本号。 */
declare interface ArtifactRef { id: string; version: number }
/** 每种产物都接受的卡片元数据。 */
declare interface ArtifactOptions {
  /** 卡片标题；缺省用 id。用用户的语言。 */
  title?: string;
  /** 一两句话，产物作为运行的主交付物时显示在标题旁。 */
  description?: string;
  /** 运行的交付物：卡片和运行面板以它为首。每次运行至多一个 id；一旦设置，后续版本保持设置。 */
  primary?: boolean;
}
declare interface ArtifactFileOptions extends ArtifactOptions {
  /** 覆盖从扩展名猜出的类型（"application/pdf"、"text/html" 等）。 */
  contentType?: string;
}
/** 从上报条目里取的一个值：指向条目的点路径（"timing.after"）。 */
declare interface ArtifactField { field: string; label?: string; unit?: string }
declare interface ChartSpec extends ArtifactOptions {
  type?: "line" | "bar" | "scatter";        // 缺省 "line"
  x: ArtifactField;
  y: ArtifactField | ArtifactField[];        // 多个 = 多条序列
  scale?: "linear" | "log";                  // y 轴，缺省 "linear"
  /** 画成水平参考线的值，取第一个带该字段的条目。 */
  baseline?: ArtifactField;
}
declare interface TableSpec extends ArtifactOptions {
  columns: ArtifactField[];
  /** 标识行的字段；同键的后续条目替换该行。缺省 = 只追加。 */
  key?: string;
}
declare interface MetricsSpec extends ArtifactOptions {
  /** 每块磁贴显示拥有该字段的最新条目值。 */
  metrics: ArtifactField[];
}
declare interface BoardSpec extends ArtifactOptions {
  /** 标识卡片的字段；同键的后续条目移动/更新卡片。 */
  key: string;
  /** 持有卡片所在列的字段。 */
  status: string;
  /** 列顺序。状态不在列表里的条目落在末尾的 "other" 列。 */
  columns: string[];
  /** 卡片标题字段（缺省：键）和卡片上显示的额外字段。 */
  cardTitle?: string;
  detail?: ArtifactField[];
}
/**
 * 发布用户留下来的东西：运行旁边的卡片，结束后保留。每个 id 是编译期字面量
 * （非空、至多 64 个 [A-Za-z0-9_.-] 字符），且只与一个成员搭配使用。
 * - 内容类（file、markdown）是副作用：异步，解析为 ArtifactRef，可捕获地拒绝
 *   （文件缺失、不是文件、在工作区外、超上限，或本宿主没有产物存储）。
 *   字节在发布时复制；同一 id 再次发布会铸出下一版并保留旧版。
 * - 预设类（chart、table、metrics、board）是声明：同步、void，由打了
 *   report(item, "<id>") 标签的条目绘制。每个在顶层声明一次；完全相同的重复声明是空操作，
 *   不同或畸形的声明会让整个运行失败。
 * 上限：每次运行 32 个 id、每个 id 16 版、单文件 20 MiB、markdown 256 KB、
 * 标题 120 字符、描述 500 字符。
 */
declare const artifact: {
  /** 发布一个工作区文件，路径解析同 files.read()。类型来自扩展名，除非 opts.contentType 指定。 */
  file(id: string, path: string, opts?: ArtifactFileOptions): Promise<ArtifactRef>;
  /** 发布脚本拼出的 markdown 文本。 */
  markdown(id: string, content: string, opts?: ArtifactOptions): Promise<ArtifactRef>;
  /** 声明一个由 report(item, id) 喂养的图表：每个打标签的条目是一个点。 */
  chart(id: string, spec: ChartSpec): void;
  /** 声明一个由 report(item, id) 喂养的表格：每个打标签的条目是一行。 */
  table(id: string, spec: TableSpec): void;
  /** 声明一行由 report(item, id) 喂养的指标磁贴：每块显示它拥有的最新值。 */
  metrics(id: string, spec: MetricsSpec): void;
  /** 声明一个由 report(item, id) 喂养的看板：每个打标签的条目是一张卡，按 status 字段放置。 */
  board(id: string, spec: BoardSpec): void;
};

/**
 * 标记一个阶段的开始：其后步骤组成一个命名组，在用户批准的图上画成一个节点。
 * 仅作用于展示：不启动、不等待、不返回任何东西。必填。名字是非空编译期字面量，
 * 调用是独立语句。标记覆盖从它到所在块末尾的一切（嵌套块和内联 helper 都算）；
 * 两个同名标记是一个阶段。每个阶段至少含一个 ask 或 world.run。
 */
declare function phase(name: string): void;

/**
 * 一个有类型的缺口：等更早步骤的发现到了再写的代码。运行到达未填充的洞时，
 * 这条分支等待，同时你来写函数体（FillWorkflowHole）；其余运行继续。
 * prompt 是你将收到的消息，所以把决策需要的值插值进去。函数体在洞的位置运行，
 * 能看到之前声明的绑定，并返回洞的值。填充形式把函数体放在最后。
 */
declare function hole<T>(name: string, prompt?: string): Promise<T>;
declare function hole<T>(name: string, body: () => Promise<T>): Promise<T>;
declare function hole<T>(name: string, prompt: string, body: () => Promise<T>): Promise<T>;

/** files.grep 找到的一行匹配。 */
declare interface GrepMatch {
  /** 匹配所在文件的工作区相对路径。 */
  path: string;
  /** 从 1 开始的行号。 */
  line: number;
  /** 匹配行的完整文本。 */
  text: string;
}

/**
 * 工作区的只读视图。每个结果都被记录，恢复的运行拿回同样的答案而不是重读。
 * 没有写入：写是子代理的任务。
 */
declare const files: {
  /** 匹配 glob 的工作区相对路径，已排序。超过 2000 个文件时调用拒绝。 */
  glob(pattern: string): Promise<string[]>;
  /** 按 UTF-8 读一个工作区文件。有大小上限。 */
  read(path: string): Promise<string>;
  /**
   * 用兼容 ripgrep 的正则搜索内容，可选按路径 glob 收窄（"*.ts"、"src/**"）；
   * 每个匹配行一条。超过 2000 个匹配或 256KB 结果时调用拒绝。
   */
  grep(pattern: string, glob?: string): Promise<GrepMatch[]>;
};

/** 工作树状态，由 git.status() 报告。 */
declare interface GitStatus {
  /** 当前分支名；HEAD 分离时缺失。 */
  branch?: string;
  /** 没有暂存、修改或未跟踪内容时为真。 */
  clean: boolean;
  /** 已暂存待提交的工作区相对路径。 */
  staged: string[];
  /** 工作树中已修改但未暂存的工作区相对路径。 */
  unstaged: string[];
  /** git 未跟踪的路径（遵守 .gitignore）。 */
  untracked: string[];
}

/** 一次提交，由 git.log() 报告。 */
declare interface GitCommit {
  /** 完整提交哈希。 */
  hash: string;
  /** 提交信息第一行。 */
  subject: string;
  /** 作者名。 */
  author: string;
  /** 作者日期，ISO 8601。 */
  date: string;
}

/**
 * 只读 git 视图，像 files.* 一样被记录，恢复的运行拿回同样的答案；没有调用能写。
 * base 指单个 ref（不含 ".." 范围）。路径是工作区相对路径，工作区之外的改动不报告；
 * 只有 git.log 是仓库级的。上限是拒绝而不是截断：diff 512KB、log 100 次提交。
 * 在 git 仓库之外或没有 git 时，每个调用都可捕获地拒绝。
 */
declare const git: {
  /**
   * 相对 base 有变动的工作区相对路径。无 base：相对 HEAD 修改的文件，加上未跟踪文件。
   * 有 base ref：当前状态与该 ref 不同的已跟踪文件，包含未提交修改，不含未跟踪文件。
   */
  changedFiles(base?: string): Promise<string[]>;
  /**
   * 相对 base（缺省 HEAD）的统一 diff。除非收窄到一个工作区相对路径，否则覆盖整个工作区。
   */
  diff(base?: string, path?: string): Promise<string>;
  /** 当前工作树状态，面向工作区。 */
  status(): Promise<GitStatus>;
  /** 最近的提交，最新的在前。缺省 20，最多 100。 */
  log(count?: number): Promise<GitCommit[]>;
};

/** 一次 world.run 命令的结果，包含非零退出。 */
declare interface WorldRunResult {
  /** 进程退出码。非零是正常返回值——对它分支。 */
  exitCode: number;
  /** 捕获的 stdout（UTF-8）。上限 256KB；超过时调用拒绝。 */
  stdout: string;
  /** 捕获的 stderr（UTF-8）。上限与拒绝语义同 stdout。 */
  stderr: string;
}

/**
 * 运行一个命令。每次调用只跑一次，结果被记录：恢复的运行重用记录的结果而不是重跑。
 * 进程完成即 RESOLVE，非零退出也一样；promise 只在 spawn 失败或超时时（可捕获地）拒绝，
 * 超时缺省 300000ms，timeoutMs 覆盖，无上界。cmd 是编译期字面量，确认时展示给用户。
 * 固定 argv，绝不过 shell：没有管道、重定向或展开。cwd 是工作区。
 * Node 内建可用 world.run("node", ["-e", code]) 触达。
 */
declare const world: {
  run(cmd: string, args?: string[], opts?: { timeoutMs?: number }): Promise<WorldRunResult>;
};

/**
 * 两个阶段之间的类型化项目流：一个阶段 send()，另一个用 for await 消费。
 * 无界、先进先出、每项恰好给一个接收者。
 */
declare interface Channel<T> extends AsyncIterable<T> {
  /** 入队一项；同步，从不等待。close() 之后抛 ChannelClosed。 */
  send(item: T): void;
  /**
   * 结束流：接收者先拿到剩余条目，然后循环退出。幂等。
   * 再也没人能发送的通道会让运行失败为 ChannelDeadlock 并点名它，一旦在途工作不可能再发送。
   */
  close(): void;
}

/**
 * 创建通道。名字用于死锁消息；不是身份。
 * 空且开着的通道上的接收者会等待发送或关闭。
 */
declare function channel<T>(name?: string): Channel<T>;

/**
 * 把一个异步块作为并发阶段运行。它立即启动，不是等你 await，并返回 promise。
 */
declare function future<T>(body: () => Promise<T>): Promise<T>;
```
<!-- facade-dts:end -->

### 16.3 编译器与分析器强制执行的规则

- 普通 TypeScript。结果类型用普通 `interface Foo { ... }` 或 `type Foo = ...` 定义，并作为 `ask<T>` 类型参数传入。
- 在 `strict` 下编译，`noUncheckedIndexedAccess` 关闭：数组或记录索引（`items[i]`）不需要守卫。`.find()`、`.match()`、`Map.get()` 和可选属性仍然产出 `T | undefined` / `null`，使用前必须守卫。
- 绝不用 `declare` 修饰符：脚本编在一个函数体里，ambient 声明在那里非法。不要 `export` 语句——工作流的输出是最终 `return`。不要 `import`。
- 允许顶层 `await` 和最终 `return <value>`；返回的值正是完成通知携带的内容。
- 不要 Node 或 web API：`process`、`fetch`、`fs` 不存在，类型检查失败。
- `world.run` 执行真实命令。第一个参数必须是编译期字面量——脚本的命令集合在确认时展示给用户——所以运行期值插值进 args 数组，绝不插进命令名。非零退出码作为值返回，不是异常（§5）。
- `hole<T>()`（§15）：显式类型参数；名字字面量至多 128 字符，在脚本的洞和阶段标记中唯一；调用被 await 且内联书写；在所有数组方法扇出回调之外（`for...of` 体可以）；函数体不读洞之后声明的绑定。以上任何一条违反都是诊断 9012。
- 阶段是必填的（§8）：用 `phase("...")` 标记覆盖整个脚本，每个阶段头部一个。名字是编译期字面量，调用是独立语句；标记认领其所在块的后半段（嵌套块和内联 helper 调用都算），所以 `if` 里的标记只覆盖那个分支。两个同名标记是一个阶段——这是重试循环保持两个节点而不是每轮膨胀的方式。每个阶段至少含一个子代理 `ask` 或一个 `world.run`；两个 ask 之间的纯脚本逻辑不是阶段，所以别为顶部的准备或底部的 `return` 开阶段。
- 子代理名字：传给 `agent("...")` 的名字是用户看到的卡片，所以用本会话的语言、为用户而写（§9）。它也是身份：一次运行内唯一，也是修订后的运行按它匹配缓存结果的键。跨修订保持名字稳定，循环里的重复给各自算出来的名字。
- 产物 id 和 `report` 标签是编译期字面量；一个 id 只属于一种产物；至多一个产物带 `primary: true`；预设产物在顶层声明，不在循环、回调或分支里（§10），填充的函数体算在洞所在的位置。文件缺失或过大时发布可捕获地拒绝。
- 模型侧错误永远到不了脚本。限流、过载、网络错误和超时由运行时无限重试，同时它会自适应扇出；确定性的错误（登录过期、模型不在计划内、配额上限）让运行以 `stopped` 停下，等用户修复并恢复。不要为 provider 错误写重试循环或 `try`/`catch`；把它们留给逻辑失败——结果校验失败、门禁没过、世界读取超上限、产物源缺失、或 `ContextLimit`（ask 即使压缩后仍超出模型上下文：拆开它）。
- `channel`、`future`、`Channel.send` 和 `Channel.close` 是 facade 可调用物：直接调用（9001 适用于存储或别名后的调用）；通道值本身可以自由存储或传递。`send` 在 `close()` 之后抛 `ChannelClosed`，可捕获。停滞让运行失败、从不超时：`ChannelDeadlock` 在接收者等待再也没人能发送的通道时失败（逐个点名），`ScriptStalled` 在其他 promise 永远无法落定时失败。
- 最终 `return` 是模型面对的结果；产物是用户面对的交付物。绝不要同一份内容同时放进两边。
- 遇到诊断时，编辑结果点名的文件并再次调用工具、传 `path`；绝不要第二次粘贴脚本。

### 16.4 `AmendWorkflow`

修订一个已存在的运行：它启动一个 supersede 旧运行的新运行，并把旧运行已完成的工作作为缓存导入，所以只有你改动的部分重新付费。§13 讲缓存规则和运行中修复的动作。它对项目的任何运行生效——completed、errored、stopped 或仍在跑；仍在跑的前任在同一次调用里被停止并 supersede，所以绝不要先 `TaskStop` 它，也绝不要用 `CreateWorkflow` 从零重写。要原样继续一个 stopped 的运行，用 `ResumeWorkflowRun`。

**字段——每个省略的字段保留前任的值。**

- `run_id`（必填）：来自 `CreateWorkflow` 或 `AmendWorkflow` 的结果、通知、`GetWorkflowRun` 或 `ListWorkflowRuns`。
- `path` 或 `script`，不能同时传。`path` 是常用形式：errored 通知和 `GetWorkflowRun` 点名运行的脚本文件——原地编辑并用同一路径交回，修订只花一次 `Edit` 而不是第二份完整脚本。字节未改的文件会被拒绝（`script_unchanged`），除非调用同时改了 `max_concurrency` 或 `subagent_model`；什么都不会停，也不会创建。
- `name`：新的展示标签；缺省用前任的。
- `max_concurrency`：省略保留前任的限值，`null` 移除，数字修改（只在用户要求时）。只带 `run_id` 和 `max_concurrency` 的调用，对仍在跑的运行时原地调参而不是启动新运行：同一次运行，什么都不停、什么都不重跑。
- `subagent_model`：省略保留前任的选择，`null` 把子代理放回会话模型，模型 id 修改（只在用户要求时；绝不要用用户没点名的另一个 provider 的模型——觉得有帮助就先问）。`ListModels` 列出 id。

只改设置、对运行中的运行，仍会停掉它；它在途的东西会在新运行里重跑，除了一个命名子代理未完成的 ask，如果你没改该 ask，它从断点继续而不是重新开始。

**缓存怎么工作。** 命名子代理跨两个脚本按名字匹配；每个子代理的 ask 按顺序、按逐字节相同的指令匹配。匹配的从记录结果零 token 落定；改动或新增的 ask 真的跑。缓存保持开放，直到某个 live 子代理第一次写工作区（或某个 live `world.run` 执行）：从那一刻起，缓存的 world 读取和“其子代理读过或跑过东西”的缓存 ask 描述的是一个已不存在的工作区，所以它们也转为真跑。只回答过的 ask 继续从缓存落定。编辑 ask 文本就是强制它重跑的方式；名字保持稳定，可调常量不要进 ask 文本（§13）。

出现诊断时什么都没停、也没启动；修结果点名的文件，再用 `path` 调用一次。

**确认。** 本会话启动的运行（经 `CreateWorkflow`、之前的 `AmendWorkflow` 或工作流中心）在修订时不开确认窗口，即使正在跑。用户停掉的运行、或其他会话启动的运行，需要用户先确认；会让子代理落到会话和运行都没用过的 provider 上的修订也一样（你在 `subagent_model` 或脚本里点名的另一个 provider 下的模型）：这个选择是用户的，窗口就是他们做选择的地方。结果会点名被 supersede 的运行（如果停了）和新运行的 id。被 supersede 的运行自己不通知；新运行落定时通知。别等它，也别用 `TaskOutput` 轮询。

### 16.5 `SaveWorkflow`

保存脚本及其元数据，以便按名字再次运行——通过 `CreateWorkflow` 的 `saved` 来源，用 `ListSavedWorkflows` 发现。项目定义放在 `.zcode/workflows/<name>.dwf.ts`，随仓库提交；全局定义放在 `~/.zcode/workflows/<name>.dwf.ts`，本机每个项目都可用。

**什么时候调。** 绝不主动调。保存会把文件写进用户仓库；那是用户的决定，不是你的。刚构建的工作流看起来可复用（换输入再跑一次是合理的）时，用一句话建议保存什么、为什么，然后停下等；用户同意或直接要求（“保存这个工作流”）后才调 `SaveWorkflow`。为单一问题定制的一次性脚本不值得建议。**字段**：`name` 文件安全标识（字母、数字、点、横线、下划线）；覆盖已有名字会替换该工作流，确认会说文件是新建还是覆盖——复用名字是更新，变体取新名字。`description`（必填）和 `whenToUse` 写给没看过本会话的读者。`scope`（必填，无缺省）：脚本引用本仓库文件、命令、约定或布局时用 `project`；不依赖项目里任何东西时用 `global`。`script` 或 `script_path` 不能同时传；`script_path` 保存草稿而不重新发送；该文件里的 `/* zcode-workflow` 块会被丢弃，因为元数据来自本次调用的字段。`script` 只是函数体——不带元数据块。`args` 精确命名每次运行之间会变的值——PR 号、目录、深度。每个声明给 `type`（`string`、`number`、`boolean` 或 `json`），可选 `description`、`required: true` 和 `default`；之后每次调用都按它们校验。保存的文件是合法 TypeScript：先是 `/* zcode-workflow` 的 YAML 元数据块，然后是脚本原文。脚本在询问用户前由同一个编译器类型检查；诊断意味着什么都没写。§16.3 的规则适用，包括阶段——未来的调用者没见过本会话，阶段名就是他们仅有的信息。

### 16.6 `EvalWorkflowSnippet`

在同一个编译器、沙箱和 world-read 执行路径上同步编译并运行一小段代码，并在工具调用里返回结果。什么都不持久：没有日志行、没有后台任务、没有运行。它是 §6 的试验台，用于脚本或洞函数体的固定逻辑；不是用于编排的。

- `code` 或 `path`，不能同时传：片段内联，或指向文件原样读取（不做元数据解析）。
- `timeoutMs`：整个片段的墙钟上限，毫秒；缺省 60000，最多 600000。
- 片段 facade 是 §16.2 里 `args`、`log`、`files`/`git`、`world.run`、`channel` 和 `future` 的部分；`agent()`、`report()`、`artifact.*`、`hole()` 和 `phase` 标记是仅有的缺席者——它们不存在，类型检查会失败。语言规则同脚本：普通 interface、顶层 `await`、最终 `return <value>`。返回值会被序列化进结果，所以返回摘要而不是转储；超过 256KB 会让调用失败。`log(...)` 行按顺序返回。含 `world.run` 的片段在运行前请用户确认。诊断意味着什么都没跑：修片段再调用。

### 16.7 `FillWorkflowHole`

写一个运行正等着的洞的函数体（§15）。它不启动任何东西：运行保留 id、日志和子代理，而有效脚本——拼入函数体的运行脚本——是恢复所重放的、修订所修改的、`SaveWorkflow` 所保存的。

- `run_id` 和 `hole_id`（必填）：来自通知或 `GetWorkflowRun`；洞 id 是站点 id（`hole#21b40fca`），不是名字。
- `script` 或 `path`，不能同时传：只写函数体的语句。内联 `script` 在编译前写到 `.zcode/workflow-drafts/` 下的填充文件，结果给出该路径；`path` 通常是 `Edit` 之后的那个文件，字节与上次被拒的尝试相同时拒绝为 `fill_unchanged`。

函数体在洞的位置编译，对照整份脚本；诊断锚定在它所在的文件——函数体内的填在填充文件，外面的在运行草稿。出现诊断时什么都没拼进去。函数体只能点名运行启动时带的模型；任何其他模型是诊断 9011——要加模型，修订运行。`hole_not_waiting` 表示运行不在飞行中或不在等那个洞（已填、还没到、未知 id）；先恢复 stopped 的运行，洞会再问。函数体可以以新的尾洞结束而不是返回（§15）。成功时响应列出新增的阶段和运行草稿——已原地重写为有效脚本。确认遵循 amend 规则（§16.4）：本会话的运行无需窗口即可填充；否则窗口展示新站点和函数体新增的 `world.run` 命令。

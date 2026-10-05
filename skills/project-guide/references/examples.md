# 分阶段输出示例

下面的示例都基于一个虚构的 TypeScript Web 框架 Tern，其中的路径、行号、commit、issue 全部是虚构的，只用来展示写法。

你要学的是写法：怎样从问题切入，段落之间怎样用因果衔接，出处怎样标注，确认的事实和推测怎样区分，每一节讲到什么深度。不要照搬示例的措辞、篇幅和小节安排；也不要因为示例里出现了某种内容（比如某个 issue 的讨论），就给每个项目都硬凑出同样的内容。真实项目里找不到的依据，就老实说找不到。

阶段二单个功能的讲解示例见 `style.md` 中的"中间件"一节，这里不再重复。

---

## 阶段一示例

### 「这是一个什么项目」一节

> 在用 Node.js 原生的 `http` 模块写 Web 服务的时候，你很快会发现大部分代码都不是业务逻辑：要自己解析 URL 和查询参数、按路径和方法把请求分发到不同的处理函数、读取并解析请求体、校验参数，还要在出错时返回统一格式的响应。每个项目都要重写一遍这些代码，而且各写各的。Tern 就是把这些重复劳动收拢起来的 Web 框架：你只需要声明"哪个路径由哪个函数处理、参数长什么样"，剩下的交给框架来处理。
>
> 在 `examples/` 下的所有示例里，用户代码都只是通过 `app.use`、`app.get` 等方法把函数注册给 Tern，真正决定何时调用这些函数的是 Tern 自己（入口在 `src/core/application.ts` 的 `handle` 方法）。这意味着学习 Tern 的关键不在于记住 API，而在于理解它的执行流程：你的函数会在什么时机、按什么顺序、拿到什么参数被调用。
>
> Tern 的中间件机制采用了和 Koa 相同的"洋葱模型"，但路由和参数校验是内置的，不交给第三方中间件。README 的"Why Tern"一节给出了理由：路由和校验放在框架内部，才能在应用启动时就发现路由冲突和 schema 错误，不用等到请求进来才报错。这个"尽早发现错误"的思路会在后面的架构设计中反复出现。

### 「模块划分」一节中单个模块的写法

> **router** 解决了这样一个问题：一个请求进来，应该交给哪个处理函数？它位于 `src/router/`，对外只暴露 `Router` 类（`src/router/index.ts`），内部用一棵基数树存储所有路由。基数树（radix tree）是一种把公共前缀合并成同一个节点的前缀树，`/users/:id` 和 `/users/me` 会共享 `users` 这个节点。
>
> 路由之所以单独成一个模块，是因为它和请求处理的其他环节几乎没有耦合。它的输入只有方法和路径，输出是匹配到的处理函数和路径参数，至于请求体、响应格式、中间件，它一概不关心。这种独立性带来两个好处：router 可以单独测试（`test/router/` 下的测试完全不启动 HTTP 服务）；插件也可以创建自己的子路由器，挂到主路由上。依赖关系上，router 只依赖 `core` 中的 `Context` 类型，用来注入路径参数；反过来，`core` 中的 `Application` 会在启动时把 router 包装成一个中间件，放在中间件链的末尾。

### 「一次操作的完整旅程」一节

> 我们跟踪一个最典型的请求 `GET /users/42`，假设应用注册了一个日志中间件和一条路由 `app.get('/users/:id', getUser)`。
>
> 请求首先到达 Node.js 的 HTTP 服务器。服务器调用的是 Tern 在 `app.listen` 时注册的回调，也就是 `src/core/application.ts` 中的 `Application.handle`：
>
> ```ts title="application.ts"
> async handle(req: IncomingMessage, res: ServerResponse) {
>   const ctx = new Context(this, req, res);
>   ctx.status = 404;
>   try {
>     await this.fn(ctx);
>   } catch (err) {
>     ctx.onerror(err);
>     return;
>   }
>   respond(ctx);
> }
> ```
>
> `handle` 做的第一件事，是用原始的 `req` 和 `res` 创建一个 `Context` 对象。`Context` 是 Tern 里最重要的抽象，之后所有的中间件和处理函数拿到的都是它，而不是原始的 `req`/`res`。这样做的好处是：框架可以把解析好的参数统一挂在 `Context` 上，提供 `ctx.json()` 这样的便捷方法，同时把对 Node.js 原生对象的依赖限制在 `Context` 内部。紧接着的 `ctx.status = 404` 是一个默认值：如果整条链路走完都没有人设置响应，请求就自然以 404 结束，不需要专门的"未找到"处理逻辑。链路中途抛出的异常则由 `catch` 交给 `ctx.onerror`，它负责输出错误响应，所以这时直接 `return`，不再走正常的 `respond`。
>
> 接着，`this.fn(ctx)` 调用中间件链。`this.fn` 是启动时在 `Application.listen` 里由 `compose` 组合好的，只组合一次，不会每个请求都重新组合。请求依次穿过用户注册的中间件。在这个例子里，日志中间件记下开始时间，然后调用 `await next()`，把控制权交给下一层。
>
> 中间件链的最后一层是路由中间件，同样由 `Application.listen` 在组合之前自动追加到用户中间件的后面。它调用 `Router.find('GET', '/users/42')`，在基数树中匹配到 `/users/:id` 这个节点，得到处理函数 `getUser` 和参数 `{ id: '42' }`，再把参数写入 `ctx.params`。注意这里的 `id` 是字符串 `'42'`，不是数字，因为路由层只负责匹配、不做类型转换。如果这个路由声明了 schema，下一步的校验层会把它转换成数字。把"匹配"和"校验"分成两个环节，是 Tern 有意为之的设计。
>
> 校验通过后，Tern 调用 `getUser(ctx)`。它的返回值不会马上写进响应，而是先存在 `ctx.body` 上。等整个中间件链返回（日志中间件也在 `await next()` 之后记完了耗时），`handle` 才执行上面摘录中的最后一行 `respond(ctx)`（定义在 `src/core/respond.ts`），按 `ctx.body` 的类型决定如何序列化：对象序列化为 JSON，流直接 pipe，字符串原样输出。之所以把写响应推迟到最后，是为了让外层中间件能在返回途中修改响应，比如统一包装响应格式，或者压缩响应体。
>
> ```mermaid
> sequenceDiagram
>     participant N as Node http
>     participant A as Application.handle
>     participant L as 日志中间件
>     participant R as 路由中间件
>     participant V as 校验层
>     participant H as getUser
>     N->>A: req, res
>     A->>A: 创建 Context
>     A->>L: ctx
>     L->>R: await next()
>     R->>R: Router.find 匹配 /users/:id
>     R->>V: ctx.params = { id: '42' }
>     V->>H: 校验并转换参数
>     H-->>R: 返回值存入 ctx.body
>     R-->>L: 返回
>     L-->>A: 记录耗时后返回
>     A->>N: respond 序列化 ctx.body
> ```
>
> 这条链路里有两个设计值得记住：所有环节都通过同一个 `Context` 传递状态；响应要等链路完全返回之后才写出。后面讲到的大多数功能，都是在这条链路的某个环节上做文章。

---

## 阶段三示例（主题：路由匹配）

### 「实现细节」中调用链某一步的写法

> 匹配的核心是 `src/router/tree.ts` 中的 `Node.search` 方法。它从根节点出发，在每一层按固定的优先级尝试子节点：先试静态子节点，再试参数子节点（`:id` 这种），最后试通配符子节点（`*`）。
>
> ```ts title="tree.ts"
> search(segments: string[], params: Params): Node | null {
>   if (segments.length === 0) return this.handlers ? this : null;
>   const [segment, ...rest] = segments;
>
>   const s = this.staticChildren.get(segment);
>   if (s) { const r = s.search(rest, params); if (r) return r; }
>
>   if (this.paramChild) {
>     params[this.paramChild.name] = segment;
>     const r = this.paramChild.search(rest, params);
>     if (r) return r;
>     delete params[this.paramChild.name];
>   }
>
>   if (this.wildcardChild) {
>     params['*'] = segments.join('/');
>     return this.wildcardChild;
>   }
>
>   return null;
> }
> ```
>
> 关键在 `if (r) return r` 这种写法：静态分支匹配失败时，`search` 不会直接宣告失败，而是回退到参数分支继续尝试。所以同时存在 `/users/me` 和 `/users/:id` 时，`/users/me` 总会命中静态路由；而对于 `/users/me/posts`，如果静态分支下找不到 `posts`，它会回退去尝试 `/users/:id/posts`。参数分支失败后那行 `delete params[...]` 也是为回溯服务的：它把试探时写入的参数撤掉，免得失败分支的残留值污染最终结果。通配符分支则是最后的兜底：它把剩下的所有路径段拼回去放进 `params['*']`，直接返回，不再往下匹配。正是这种回溯，让匹配结果与路由的注册顺序无关，只取决于路由本身有多"具体"。代价是最坏情况下要回溯多个分支。不过在实际项目中，同一层同时出现多种子节点的情况并不多；`bench/router.bench.ts` 中的基准测试也主要覆盖不需要回溯的场景，回溯的开销没有专门测量过。

### 「站在设计者的角度」一节

> 路由匹配最朴素的做法，是把每条路由编译成一个正则表达式放进数组，请求进来时依次尝试，第一个匹配上的胜出。这种做法实现简单，而且路径里可以写任意正则，表达能力很强。Tern 早期就是这么做的：在 commit `a1b2c3d` 之前，`src/router/index.ts` 里就是一个 `RegExp` 数组。
>
> 改用基数树的直接原因，记录在这次 commit 关联的 issue #212 中：有用户的应用注册了两千多条路由，而线性扫描的耗时与路由数量成正比，匹配成了明显的瓶颈。但从这次重写的内容看，收获不只是性能。同一个 commit 删掉了文档中"请注意路由的注册顺序"的警告，并新增了启动时的冲突检测（`tree.ts` 中的 `assertNoConflict`）。在正则数组的方案里，`/users/:id` 和 `/users/:name` 可以同时注册，后者永远匹配不到，而且不会有任何报错。基数树要求同一位置只能有一个参数节点，这类错误在应用启动时就会暴露。这与 README 中"在启动时而不是请求时发现错误"的理念一脉相承。
>
> 这个选择的代价是表达能力。基数树按路径段匹配，所以 Tern 不再支持在路径里写任意正则，动态写法只剩下 `:param` 和末尾的 `*` 两种。需要更复杂的约束时（比如 `id` 必须是数字），作者的答案是交给校验层：issue #230 中有人请求恢复正则路由，维护者建议改用 schema 来约束参数格式。由此可以看出作者的分工思路：路由只负责"这个请求归谁处理"，"参数是否合法"是另一个环节的事。
>
> 还有一种更常见的折中做法，是在基数树节点上附加可选的正则约束，不过代码和讨论中都没有说明为什么没有采用它。一个可能的原因是，那样会让冲突检测变得无法完成：两个任意正则能否匹配到同一个路径段，很难在启动时判断出来（推测）。

### 「常见的坑」一节中的一条

> **路径参数永远是字符串。** 很多人会写 `if (ctx.params.id === 42)`，结果条件永远不成立。原因前面说过：路由层只做匹配，不做类型转换，`Node.search` 的参数分支里 `params[this.paramChild.name] = segment` 这一行，把切出来的路径段原样赋给了 `params`。需要数字时，应该给路由声明 schema，让校验层完成转换，不要在每个处理函数里手动调用 `Number()`。手动转换时，`Number('abc')` 会得到 `NaN`，然后悄悄流进业务逻辑；schema 校验则会直接返回 400。

---

## 对话回复示例（阶段一完成后）

> 全景文档已写入 `/path/to/tern/project-guide/01-overview.md`。
>
> Tern 是一个用 TypeScript 写的 Web 框架。它的骨架可以概括成一句话：每个请求都被包装成一个 `Context`，依次穿过一条由中间件组成的"洋葱"，路由本身也只是这条链路末端的一个中间件。读懂 `src/core/application.ts` 中的 `handle` 和 `compose` 这两个函数，你就掌握了 Tern 的骨架，总共大约两百行代码。
>
> 另一个贯穿整个项目的思路是"尽量在启动时、而不是请求时发现错误"：路由冲突、写错的 schema、缺失的插件依赖，都会在 `app.listen` 之前抛出。读源码时你会看到不少看似多余的检查，它们都是为这个目标服务的。
>
> 接下来建议先看功能讲解：`/project-guide features`。如果想直接深入，最推荐这两个：
>
> - `/project-guide deep 路由匹配`：基数树的实现，以及它取代正则方案的过程，是理解 Tern 设计取舍的最好入口。
> - `/project-guide deep 插件系统`：Tern 的大部分内置功能本身就是用插件实现的，读懂它，你就能自己扩展这个框架。

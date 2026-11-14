# 报错根因
> `✘ [ERROR] The detected framework ("Hono") cannot be automatically configured.`

Wrangler 4.68+ 新增了**自动框架检测自动配置**功能，这个项目是原生 Hono Worker，**没有配套的自动适配适配器**，并且仓库里**缺少 wrangler.toml / wrangler.jsonc**，CI 非交互式环境下自动配置流程直接炸掉。
你的日志：`tsc --noEmit` 只是类型检查，它本身**不会打包输出 worker 脚本**，这也是这个项目的特点。

> 重点：只要项目根目录放一个合法的 wrangler 配置文件，Wrangler 就会**跳过自动框架检测**，直接走正常部署流程。

## 方案1（推荐，一次性修复，推到github）
在项目根目录新建 `wrangler.toml`，内容（适配这个 cf-outlook-email 项目）：
```toml
name = "cf-outlook-email"
main = "src/index.ts"
compatibility_date = "2026-10-01"
compatibility_flags = ["nodejs_compat"]
```
- `main = "src/index.ts"`：入口文件，该项目源码就是 src/index.ts
- `nodejs_compat`：这个邮件项目需要 node 兼容标志，必不可少

然后修改 Cloudflare Workers Builds 里的 **Deploy 命令**，不要直接 `npx wrangler deploy`
改成：
```bash
npx wrangler deploy src/index.ts
```

> 原理：有 wrangler.toml + 手动指定入口，自动框架检测逻辑直接不触发，不再识别 Hono 框架、不再尝试自动适配。

## 方案2 如果你不想新增文件，直接改部署命令（临时）
部署命令写成：
```bash
npx wrangler deploy src/index.ts --config wrangler.toml
```
但是前提还是要有 wrangler.toml，不能绕过。
> ⚠️ 没有参数可以直接“关闭自动检测”，官方机制：**存在 wrangler 配置文件 = 关闭自动配置**

## 方案3 本地先跑一遍 wrangler setup 生成配置再提交
本地 clone 仓库，执行
```bash
pnpm install
npx wrangler setup
```
本地交互完成后会生成 wrangler.jsonc，把这个文件 commit 推 github，CI 再跑就不会触发自动框架检测报错。

---

# 你当前 Cloudflare 构建参数修正清单
1. Build command：`pnpm run build`（保留，它就是 `tsc --noEmit`，只做类型校验）
2. Deploy command：**改成 `npx wrangler deploy src/index.ts`**
3. 仓库根目录添加上面的 `wrangler.toml` 并提交到仓库
4. 确认 Workers Builds 绑定的是这个仓库、根目录是 `/`，不是子目录

# 额外坑提醒（cf-outlook-email 这个项目特有）
- 这个项目依赖 KV 绑定，你后面还要在 Cloudflare Workers 面板添加 KV namespace，并且在 wrangler.toml 加上 bindings，否则部署成功后运行时报错：
```toml
[[kv_namespaces]]
binding = "EMAIL_KV"
id = "这里填你的KV ID"
```
- 所有密钥（微软 OAuth、client secret 等）**在 Cloudflare Workers 面板添加 Secret**，不要写在代码里。

# 快速验证
你可以本地先测试这条命令，确认不报这个 Hono 自动配置错误：
```bash
npx wrangler deploy src/index.ts
```

如果你需要，我直接给你一份完整可直接提交、带KV绑定模板的 wrangler.toml。

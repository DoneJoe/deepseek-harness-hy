# 5. 配置

[English](05-config.md) | 中文

`cordis.yml` 中的每个 Cordis 配置项都可以携带 `config` 块，插件则声明一个 schema，在运行 `apply` 前验证该块。错误配置会导致加载失败，并给出准确的错误：插件绝不会在配置不完整时启动。

## 可配置插件

创建 `config-demo.ts`，并将其放在 `tmp/cordis-tutorial` 中：

```ts
import type { Context } from '@deepseek-ai/cordis'
import Schema from '@deepseek-ai/schemastery'

export const name = 'config-demo'

export interface Config {
  greeting: string
  targets: string[]
}

export const Config: Schema<Config> = Schema.object({
  greeting: Schema.string().default('Hello'),
  targets: Schema.array(String).default(['world']),
})

export function apply(ctx: Context, config: Config) {
  for (const target of config.targets) {
    console.log(`${config.greeting}, ${target}!`)
  }
}
```

导出的 `Config` 既是 TypeScript 接口，也是同名的运行时 schema：消费方获得类型，Cordis 获得验证器。本仓库使用 [Schemastery](https://github.com/shigma/schemastery) 定义 schema；Cordis 本身接受任意 [Standard Schema](https://standardschema.dev/) 验证器，因此将普通对象导出为 `Config` 无法工作。

对其进行配置：

```yaml
- name: './config-demo.ts'
  config:
    targets: ['alpha', 'beta']
```

运行：

```
Hello, alpha!
Hello, beta!
```

未提供 `greeting`，因此 schema 默认值会将其补齐：`apply` 始终会收到完整且经过验证的配置。

## 明确报错

现在向它传入无效内容：

```yaml
- name: './config-demo.ts'
  config:
    targets: 'not-an-array'
```

再次运行：问候语消失了，进程以状态码 0 退出，一切看似正常——但插件并没有启动。schema 验证在 `apply` 之前执行，产生的错误是：

```
ValidationError: invalid config:
  - $.targets expected array but got not-an-array (at targets)
```

错误经 Cordis logger 服务记录，插件的 fiber 进入 FAILED：插件绝不会在配置不完整时启动。只是本教程的启动器没有挂载控制台导出器，报告只留在 logger 的内存缓冲区里，终端中看不到它（导出器机制与启动期可见性见[第 1 章](01-first-plugin.zh.md)「尝试制造错误」一节与[第 6 章](06-composition-and-hmr.zh.md)）。loader 按配置项隔离失败：其余插件照常运行，进程在无事可做后以状态码 0 退出。如果某个插件的配置通过了 schema 验证，但其中指定的资源或提供方不可用，该插件也应当在能解析该引用时立即拒绝。

## 计算得到的配置值

本仓库使用的 loader 支持 `!!js` 标签，用于必须在加载时计算的配置值：

```yaml
- name: './config-demo.ts'
  config:
    greeting: !!js process.env.DEMO_GREETING ?? 'Hello'
```

`!!js` 仅在 `config` 与条目 `disabled` 字段内有效。`disabled: !!js ...` 在每次挂载决策时基于 loader 上下文求值（本仓库的扩展），可以按平台或环境门控一行；其余元数据（`name`、`id`、`inject` 等）保持静态，其中的表达式是普通真值数据。详见 [loader 配置](../cordis-primer.zh.md#loader-configuration)。

下一章：[组合与 HMR（热模块替换）](06-composition-and-hmr.zh.md)：将 `cordis.yml` 视为应用。

[![](https://img.shields.io/badge/powered_by-dsh-4D6BFE?style=flat-square&logo=deepseek&logoColor=white)](https://github.com/deepseek-ai/deepseek-harness)

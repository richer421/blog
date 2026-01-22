---
title: CUE 配置语言详解：比 YAML 更强大的配置方案
date: 2026-01-22 23:15:00
tags:
  - CUE
  - 配置管理
  - 云原生
  - DevOps
categories:
  - 云原生
  - 工具
---

## 配置管理的痛点

写过 Kubernetes YAML 的人都知道，配置管理有多痛苦：

```yaml
# deployment-dev.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 1
  template:
    spec:
      containers:
      - name: app
        image: my-app:dev
        resources:
          limits:
            memory: "128Mi"

---
# deployment-prod.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 10
  template:
    spec:
      containers:
      - name: app
        image: my-app:v1.0
        resources:
          limits:
            memory: "512Mi"
```

问题很明显：
- **重复代码**：大量配置在不同环境重复
- **没有类型检查**：拼写错误要到运行时才发现
- **难以验证**：无法保证配置的正确性
- **缺乏约束**：无法定义"replicas 必须大于 0"这类规则

<!-- more -->

## CUE 是什么

[CUE](https://cuelang.org/)（Configure, Unify, Execute）是 Google 前员工 Marcel van Lohuizen 开发的配置语言。它不是简单的数据格式（如 JSON/YAML），而是一种带有类型系统和约束验证的配置语言。

CUE 的核心理念：**配置即代码，但不是编程**。

### 与其他工具的对比

| 工具 | 定位 | 图灵完备 | 类型检查 | 约束验证 |
|------|------|---------|---------|---------|
| YAML | 数据格式 | ❌ | ❌ | ❌ |
| JSON | 数据格式 | ❌ | ❌ | ❌ |
| Go Template | 文本模板 | ✅ | ❌ | ❌ |
| Jsonnet | 配置语言 | ✅ | ⚠️ | ⚠️ |
| **CUE** | 配置语言 | ❌ | ✅ | ✅ |

CUE 故意不图灵完备，避免在配置中写复杂逻辑。

## CUE 基础语法

### 基本数据类型

```cue
// 字符串
name: "my-app"

// 数字
port: 8080
replicas: 3

// 布尔值
enabled: true

// 列表
tags: ["web", "api", "v1"]

// 对象
metadata: {
    name: "my-app"
    namespace: "default"
}
```

### 类型约束

CUE 的核心特性是类型约束：

```cue
// 定义类型
name: string
port: int
enabled: bool

// 具体值必须符合类型
name: "my-app"  // ✅ 正确
port: 8080      // ✅ 正确
port: "8080"    // ❌ 错误：类型不匹配
```

### 默认值

使用 `*` 定义默认值：

```cue
// port 默认是 80，但可以被覆盖
port: *80 | int

// 如果不指定，port 就是 80
// 如果指定了，必须是 int 类型
```

### 约束条件

CUE 可以定义值的约束：

```cue
// replicas 必须在 1 到 100 之间
replicas: int & >=1 & <=100

// 端口必须在有效范围内
port: int & >0 & <65536

// 字符串必须匹配正则
email: string & =~"^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$"

// 枚举值
env: "dev" | "staging" | "prod"
```

### 结构定义

定义可复用的结构：

```cue
// 定义一个 Container 结构
#Container: {
    name: string
    image: string
    port: int & >0 & <65536
    replicas: *1 | int & >=1 & <=100
}

// 使用这个结构
myApp: #Container & {
    name: "web"
    image: "nginx:1.20"
    port: 80
    replicas: 3
}
```

## CUE 的核心能力：统一（Unification）

CUE 最强大的特性是**统一**（Unification）：多个配置可以合并，只要它们不冲突。

### 基础统一

```cue
// 文件 1：base.cue
deployment: {
    apiVersion: "apps/v1"
    kind: "Deployment"
    metadata: name: string
}

// 文件 2：app.cue
deployment: {
    metadata: name: "my-app"
    spec: replicas: 3
}

// 统一后的结果
deployment: {
    apiVersion: "apps/v1"
    kind: "Deployment"
    metadata: name: "my-app"
    spec: replicas: 3
}
```

### 冲突检测

```cue
// 文件 1
port: 80

// 文件 2
port: 8080

// ❌ 错误：冲突！80 != 8080
```

### 约束合并

```cue
// 文件 1：定义类型
replicas: int

// 文件 2：添加约束
replicas: >=1 & <=100

// 文件 3：提供具体值
replicas: 3

// 统一后：replicas 是 3，且满足所有约束
```

## 实战：用 CUE 管理 Kubernetes 配置

### 定义基础模板

```cue
// base.cue
package k8s

// 定义 Deployment 结构
#Deployment: {
    apiVersion: "apps/v1"
    kind: "Deployment"
    metadata: {
        name: string
        namespace: *"default" | string
        labels: [string]: string
    }
    spec: {
        replicas: *1 | int & >=1 & <=100
        selector: matchLabels: [string]: string
        template: {
            metadata: labels: [string]: string
            spec: {
                containers: [...#Container]
            }
        }
    }
}

// 定义 Container 结构
#Container: {
    name: string
    image: string
    ports: [...{
        containerPort: int & >0 & <65536
        protocol: *"TCP" | "UDP"
    }]
    resources?: {
        limits?: {
            cpu?: string
            memory?: string
        }
        requests?: {
            cpu?: string
            memory?: string
        }
    }
}

// 定义 Service 结构
#Service: {
    apiVersion: "v1"
    kind: "Service"
    metadata: {
        name: string
        namespace: *"default" | string
    }
    spec: {
        selector: [string]: string
        ports: [...{
            port: int & >0 & <65536
            targetPort: int & >0 & <65536
            protocol: *"TCP" | "UDP"
        }]
        type: *"ClusterIP" | "NodePort" | "LoadBalancer"
    }
}
```

### 定义应用配置

```cue
// app.cue
package k8s

// 应用参数
#AppConfig: {
    name: string
    image: string
    port: int
    replicas: int
    env: "dev" | "staging" | "prod"
}

// 根据参数生成 Deployment
deployment: #Deployment & {
    metadata: {
        name: #AppConfig.name
        labels: {
            app: #AppConfig.name
            env: #AppConfig.env
        }
    }
    spec: {
        replicas: #AppConfig.replicas
        selector: matchLabels: app: #AppConfig.name
        template: {
            metadata: labels: {
                app: #AppConfig.name
                env: #AppConfig.env
            }
            spec: containers: [{
                name: #AppConfig.name
                image: #AppConfig.image
                ports: [{
                    containerPort: #AppConfig.port
                }]
            }]
        }
    }
}

// 生成 Service
service: #Service & {
    metadata: name: #AppConfig.name
    spec: {
        selector: app: #AppConfig.name
        ports: [{
            port: 80
            targetPort: #AppConfig.port
        }]
    }
}
```

### 不同环境的配置

```cue
// dev.cue
package k8s

#AppConfig: {
    name: "my-app"
    image: "my-app:dev"
    port: 8080
    replicas: 1
    env: "dev"
}

// prod.cue
package k8s

#AppConfig: {
    name: "my-app"
    image: "my-app:v1.0.0"
    port: 8080
    replicas: 10
    env: "prod"
}

// 添加生产环境的资源限制
deployment: spec: template: spec: containers: [{
    resources: {
        limits: {
            cpu: "1000m"
            memory: "512Mi"
        }
        requests: {
            cpu: "100m"
            memory: "128Mi"
        }
    }
}]
```

### 导出配置

```bash
# 导出开发环境配置
cue export base.cue app.cue dev.cue -o dev.yaml

# 导出生产环境配置
cue export base.cue app.cue prod.cue -o prod.yaml
```

## CUE 的高级特性

### 条件逻辑

CUE 使用统一而非 if-else：

```cue
// 定义参数
#Config: {
    env: "dev" | "prod"

    // 根据环境设置不同的值
    if env == "dev" {
        replicas: 1
        resources: {
            limits: memory: "128Mi"
        }
    }

    if env == "prod" {
        replicas: 10
        resources: {
            limits: memory: "512Mi"
        }
    }
}
```

但更推荐用统一的方式：

```cue
#Config: {
    env: "dev" | "prod"

    // 定义默认值
    replicas: 1
    resources: limits: memory: "128Mi"

    // 生产环境覆盖
    if env == "prod" {
        replicas: 10
        resources: limits: memory: "512Mi"
    }
}
```

### 列表推导

```cue
// 定义服务列表
services: ["web", "api", "worker"]

// 为每个服务生成 Deployment
deployments: {
    for svc in services {
        "\(svc)": {
            apiVersion: "apps/v1"
            kind: "Deployment"
            metadata: name: svc
        }
    }
}

// 结果：
// deployments: {
//     web: {...}
//     api: {...}
//     worker: {...}
// }
```

### 引用和模板

```cue
// 定义通用配置
#CommonLabels: {
    app: string
    version: string
    team: "platform"
}

// 在多处使用
deployment: {
    metadata: labels: #CommonLabels & {
        app: "my-app"
        version: "v1.0"
    }
}

service: {
    metadata: labels: #CommonLabels & {
        app: "my-app"
        version: "v1.0"
    }
}
```

### 验证和测试

```cue
// 定义配置
config: {
    replicas: 3
    port: 8080
}

// 定义验证规则
config: {
    replicas: int & >=1 & <=100
    port: int & >0 & <65536
}

// 运行验证
// cue vet config.cue
```

## CUE 在实际项目中的应用

### 1. Kubernetes 配置管理

替代 Helm 或 Kustomize，用 CUE 管理多环境配置：

```
k8s/
├── base.cue          # 基础定义
├── app.cue           # 应用模板
├── dev.cue           # 开发环境
├── staging.cue       # 预发环境
└── prod.cue          # 生产环境
```

### 2. CI/CD 配置

生成 GitHub Actions、GitLab CI 等配置：

```cue
// 定义 CI 流水线
#Pipeline: {
    name: string
    on: ["push", "pull_request"]
    jobs: [string]: #Job
}

#Job: {
    "runs-on": "ubuntu-latest"
    steps: [...#Step]
}

#Step: {
    name: string
    run?: string
    uses?: string
}
```

### 3. 配置验证

在 CI 中验证配置的正确性：

```bash
# 验证所有 CUE 文件
cue vet ./...

# 导出并检查
cue export ./... | kubectl apply --dry-run=client -f -
```

## CUE vs Helm vs Kustomize

| 特性 | CUE | Helm | Kustomize |
|------|-----|------|-----------|
| **类型检查** | ✅ 强类型 | ❌ 无 | ❌ 无 |
| **约束验证** | ✅ 内置 | ❌ 需要额外工具 | ❌ 无 |
| **配置合并** | ✅ 统一机制 | ⚠️ values 覆盖 | ✅ overlay |
| **学习曲线** | ⚠️ 较陡 | ✅ 简单 | ✅ 简单 |
| **生态成熟度** | ⚠️ 较新 | ✅ 成熟 | ✅ 成熟 |
| **模板复杂度** | ✅ 声明式 | ❌ 命令式 | ✅ 声明式 |

## CUE 的局限性

说实话，CUE 也有明显的问题：

1. **学习曲线陡峭**：需要理解统一、约束等概念
2. **生态不成熟**：工具链、IDE 支持还不够完善
3. **错误信息难懂**：统一失败时的错误提示不够友好
4. **性能问题**：大型配置文件处理较慢
5. **社区较小**：相比 Helm/Kustomize，资源和案例较少

更根本的问题是：**配置真的需要这么复杂吗？**

对于简单场景，YAML + Kustomize 可能更实际。CUE 更适合：
- 大型项目，配置复杂度高
- 需要严格的类型检查和验证
- 多环境配置差异大
- 有专职的平台团队维护

## 实用技巧

### 1. 渐进式采用

不用一次性重写所有配置，可以先用 CUE 包装现有 YAML：

```cue
// 导入现有 YAML
import "encoding/yaml"

// 读取 YAML 文件
deployment: yaml.Unmarshal("""
apiVersion: apps/v1
kind: Deployment
...
""")

// 添加验证
deployment: {
    spec: replicas: int & >=1 & <=100
}
```

### 2. 使用 cue fmt

自动格式化 CUE 文件：

```bash
cue fmt ./...
```

### 3. 调试技巧

```bash
# 查看统一后的结果
cue eval config.cue

# 导出为 JSON
cue export config.cue

# 验证配置
cue vet config.cue

# 查看详细错误
cue vet -v config.cue
```

### 4. 模块化组织

```
project/
├── cue.mod/
│   └── module.cue      # 模块定义
├── pkg/
│   ├── k8s/            # Kubernetes 定义
│   └── common/         # 通用定义
└── apps/
    ├── web/
    └── api/
```

## 总结

CUE 试图解决配置管理的核心问题：如何在保持声明式的同时，提供类型检查、约束验证和配置复用能力。

它的核心优势：
- ✅ 强类型系统，编译时发现错误
- ✅ 统一机制，优雅地合并配置
- ✅ 约束验证，保证配置正确性
- ✅ 不图灵完备，避免过度复杂

但也有明显的挑战：
- ⚠️ 学习曲线陡峭
- ⚠️ 生态不够成熟
- ⚠️ 可能过度设计

对于大型项目和复杂配置场景，CUE 值得尝试。对于小型项目，YAML + Kustomize 可能更实际。

关键是理解你的配置复杂度是否真的需要 CUE 这样的工具。

## 参考资料

- [CUE 官方网站](https://cuelang.org/)
- [CUE 语言规范](https://cuelang.org/docs/references/spec/)
- [CUE 教程](https://cuelang.org/docs/tutorials/)
- [CUE GitHub](https://github.com/cue-lang/cue)

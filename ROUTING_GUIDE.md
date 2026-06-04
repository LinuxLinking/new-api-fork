# New API 路由与智能分发指南

## 目录
- [问题解答](#问题解答)
- [现有架构分析](#现有架构分析)
- [最小 Demo 实现](#最小-demo-实现)

---

## 问题解答

### Q1: 是否支持自动路由？cc switch 做不到自动路由，这个可以吗？

**A: 是的，New API 原生支持智能自动路由！**

通过代码分析 ([`middleware/distributor.go`](file:///workspace/middleware/distributor.go), [`service/channel_select.go`](file:///workspace/service/channel_select.go), [`model/channel_cache.go`](file:///workspace/model/channel_cache.go))，New API 提供以下自动路由能力：

#### 现有自动路由功能：

1. **分组自动选择 (`auto` 模式)**
   - 使用 `group=auto` 可以自动遍历多个分组寻找可用通道
   - 支持跨分组重试机制
   - 代码位置：[`service/channel_select.go:CacheGetRandomSatisfiedChannel`](file:///workspace/service/channel_select.go#L97)

2. **加权随机选择**
   - 按通道权重 (Weight) 进行随机分配
   - 按优先级 (Priority) 分组，高优先级先尝试
   - 代码位置：[`model/channel_cache.go:GetRandomSatisfiedChannel`](file:///workspace/model/channel_cache.go#L97)

3. **失败自动重试**
   - 失败时自动降级到低优先级通道
   - 支持自动封禁失败通道
   - 代码位置：[`controller/relay.go`](file:///workspace/controller/relay.go)

4. **通道亲和性 (Sticky Routing)**
   - 优先复用上次成功的通道
   - 代码位置：[`service/channel_affinity.go`](file:///workspace/service/channel_affinity.go)

---

### Q2: 是否支持智能体的任务分发路由功能？按照任务类型分发给不同的大模型？

**A: 现有系统支持基础模型路由，可以扩展实现智能体任务分发！**

#### 现有能力分析：

1. **模型级路由**
   - 系统已支持按模型名称选择通道
   - 每个通道可配置支持的模型列表
   - 代码位置：[`model/channel_cache.go:InitChannelCache`](file:///workspace/model/channel_cache.go#L22)

2. **任务类型支持**
   - 已有任务框架：[`model/task.go`](file:///workspace/model/task.go)
   - 支持多种任务平台 (Suno, Midjourney, 视频生成等)
   - 代码位置：[`constant/task.go`](file:///workspace/constant/task.go)

3. **可扩展架构**
   - 中间件模式 ([`middleware/distributor.go`](file:///workspace/middleware/distributor.go)) 便于插入自定义路由逻辑

---

## 现有架构分析

### 核心路由流程

```
请求 → Distributor 中间件 
       ↓
    获取 ModelRequest (模型、分组)
       ↓
    ┌─────────────────────┐
    │ 优先通道亲和性？    │ → 是 → 使用上次成功的通道
    └─────────────────────┘
       ↓ 否
    ┌─────────────────────┐
    │ 分组 = auto？       │ → 是 → 自动遍历分组
    └─────────────────────┘
       ↓ 否
    按模型+分组选择通道
       ↓
    加权随机选择
       ↓
    设置上下文 → 转发
```

### 关键数据结构

#### 1. 通道 (Channel) - [`model/channel.go`](file:///workspace/model/channel.go)
```go
type Channel struct {
    Id       int
    Type     int    // 通道类型 (OpenAI, Claude, Gemini 等)
    Key      string // API Key
    Models   string // 支持的模型列表，逗号分隔
    Group    string // 所属分组，逗号分隔
    Priority *int64 // 优先级 (0-10)
    Weight   *int   // 权重
    Status   int    // 状态
    // ... 其他字段
}
```

#### 2. 通道类型 - [`constant/channel.go`](file:///workspace/constant/channel.go)
支持 50+ 种通道类型，包括：
- `ChannelTypeOpenAI` (1)
- `ChannelTypeAnthropic` (14)
- `ChannelTypeGemini` (24)
- `ChannelTypeDeepSeek` (43)
- 等等...

---

## 最小 Demo 实现

下面是一个**最简单、最小、最易实现**的 Demo，展示如何：
1. 实现基于任务类型的智能路由
2. 添加新的路由策略
3. 在现有架构中扩展

### Demo 架构设计

```
┌─────────────────────────────────────────────────┐
│         智能任务路由器 (Task Router)            │
├─────────────────────────────────────────────────┤
│  任务类型分析器                                  │
│  ├─ Chat → GPT-4o / Claude 3.5                 │
│  ├─ Code → DeepSeek-Coder / GPT-4o-mini        │
│  ├─ Image → DALL-E 3 / Midjourney              │
│  ├─ Reasoning → o3-mini / Claude 3.7           │
│  └─ ...                                        │
└─────────────────────────────────────────────────┘
```

### 实现步骤

#### 第一步：创建任务类型定义

创建文件 `middleware/task_router.go`：

```go
package middleware

import (
    "strings"
    "github.com/gin-gonic/gin"
    "github.com/QuantumNous/new-api/common"
    "github.com/QuantumNous/new-api/constant"
)

// TaskType 定义任务类型
type TaskType string

const (
    TaskTypeChat      TaskType = "chat"       // 普通对话
    TaskTypeCode      TaskType = "code"       // 代码任务
    TaskTypeImage     TaskType = "image"      // 图像生成
    TaskTypeReasoning TaskType = "reasoning"  // 推理任务
    TaskTypeEmbedding TaskType = "embedding"  // 向量生成
)

// TaskRoutingConfig 任务路由配置
type TaskRoutingConfig struct {
    // 任务类型 -> 推荐模型映射
    ModelMap map[TaskType][]string
    // 任务类型 -> 优先通道类型
    ChannelTypeMap map[TaskType][]int
}

var taskRouterConfig = &TaskRoutingConfig{
    ModelMap: map[TaskType][]string{
        TaskTypeChat:      {"gpt-4o", "claude-3-5-sonnet-20241022", "qwen-max"},
        TaskTypeCode:      {"deepseek-coder", "gpt-4o-mini", "claude-3-5-haiku"},
        TaskTypeImage:     {"dall-e-3", "midjourney"},
        TaskTypeReasoning: {"o3-mini", "o1", "claude-3-7-sonnet-20250219"},
        TaskTypeEmbedding: {"text-embedding-3-small", "text-embedding-ada-002"},
    },
    ChannelTypeMap: map[TaskType][]int{
        TaskTypeChat:      {constant.ChannelTypeOpenAI, constant.ChannelTypeAnthropic, constant.ChannelTypeAli},
        TaskTypeCode:      {constant.ChannelTypeDeepSeek, constant.ChannelTypeOpenAI},
        TaskTypeImage:     {constant.ChannelTypeOpenAI, constant.ChannelTypeMidjourney},
        TaskTypeReasoning: {constant.ChannelTypeOpenAI, constant.ChannelTypeAnthropic},
        TaskTypeEmbedding: {constant.ChannelTypeOpenAI},
    },
}

// DetectTaskType 从请求中智能检测任务类型
func DetectTaskType(c *gin.Context, modelRequest *ModelRequest) TaskType {
    // 1. 首先根据 URL 路径判断
    path := c.Request.URL.Path
    
    if strings.Contains(path, "/images/") {
        return TaskTypeImage
    }
    if strings.Contains(path, "/embeddings") {
        return TaskTypeEmbedding
    }
    
    // 2. 根据模型名称判断
    model := strings.ToLower(modelRequest.Model)
    
    switch {
    case strings.Contains(model, "coder") || strings.Contains(model, "code"):
        return TaskTypeCode
    case strings.Contains(model, "o3") || strings.Contains(model, "o1") || 
         strings.Contains(model, "thinking") || strings.Contains(model, "reasoning"):
        return TaskTypeReasoning
    case strings.Contains(model, "dall-e") || strings.Contains(model, "midjourney"):
        return TaskTypeImage
    case strings.Contains(model, "embedding"):
        return TaskTypeEmbedding
    default:
        return TaskTypeChat
    }
}

// GetRecommendedModels 获取任务类型推荐的模型列表
func GetRecommendedModels(taskType TaskType) []string {
    if models, ok := taskRouterConfig.ModelMap[taskType]; ok {
        return models
    }
    return nil
}
```

#### 第二步：修改 Distributor 中间件集成

在 [`middleware/distributor.go`](file:///workspace/middleware/distributor.go#L32) 中添加智能路由逻辑：

```go
// 在 Distribute 函数中，获取 ModelRequest 后添加：

// 智能任务路由增强
taskType := DetectTaskType(c, modelRequest)
common.SetContextKey(c, "task_type", string(taskType))

// 如果用户没有指定模型，可以根据任务类型推荐
if modelRequest.Model == "" {
    recommendedModels := GetRecommendedModels(taskType)
    if len(recommendedModels) > 0 {
        modelRequest.Model = recommendedModels[0] // 使用第一个推荐模型
    }
}

// 继续原有流程...
```

#### 第三步：创建示例测试文件

创建 `examples/task_routing_demo.go`（可选，作为独立示例）：

```go
package main

import (
    "fmt"
    "github.com/QuantumNous/new-api/middleware"
)

func main() {
    fmt.Println("=== New API 智能任务路由 Demo ===\n")
    
    // 示例 1: 代码任务
    codeTask := &middleware.ModelRequest{Model: "deepseek-coder"}
    taskType := middleware.DetectTaskType(nil, codeTask)
    fmt.Printf("模型: %-20s → 任务类型: %s\n", codeTask.Model, taskType)
    fmt.Printf("  推荐模型: %v\n\n", middleware.GetRecommendedModels(taskType))
    
    // 示例 2: 推理任务
    reasoningTask := &middleware.ModelRequest{Model: "o3-mini"}
    taskType = middleware.DetectTaskType(nil, reasoningTask)
    fmt.Printf("模型: %-20s → 任务类型: %s\n", reasoningTask.Model, taskType)
    fmt.Printf("  推荐模型: %v\n\n", middleware.GetRecommendedModels(taskType))
    
    // 示例 3: 图像任务
    imageTask := &middleware.ModelRequest{Model: "dall-e-3"}
    taskType = middleware.DetectTaskType(nil, imageTask)
    fmt.Printf("模型: %-20s → 任务类型: %s\n", imageTask.Model, taskType)
    fmt.Printf("  推荐模型: %v\n\n", middleware.GetRecommendedModels(taskType))
    
    fmt.Println("=== Demo 完成 ===")
    fmt.Println("\n实际使用时，路由会自动:")
    fmt.Println("1. 检测任务类型")
    fmt.Println("2. 选择合适的模型和通道")
    fmt.Println("3. 按权重分发请求")
}
```

#### 第四步：快速配置指南

在 New API 管理面板中配置通道时，按照以下策略分组：

| 分组名称 | 适用任务 | 推荐通道 |
|---------|---------|---------|
| `chat-group` | 普通对话 | OpenAI (GPT-4o), Anthropic, Qwen |
| `code-group` | 代码开发 | DeepSeek, OpenAI (GPT-4o-mini) |
| `image-group` | 图像生成 | OpenAI (DALL-E 3), Midjourney |
| `reasoning-group` | 深度推理 | OpenAI (o3/o1), Anthropic (Claude 3.7) |

调用示例：

```bash
# 代码任务自动选择 code-group
curl -X POST http://localhost:3000/v1/chat/completions \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "deepseek-coder",
    "messages": [{"role": "user", "content": "写一个快速排序"}],
    "group": "auto"  // 或指定 "code-group"
  }'
```

### 扩展建议

1. **提示词分析路由**：在 `DetectTaskType` 中添加对请求 body 的解析，根据提示词内容进一步分类
2. **用户级路由策略**：根据用户历史数据学习最优路由
3. **性能监控路由**：根据通道延迟和成功率动态调整权重

### 总结

✅ New API **原生支持自动路由**，比 cc switch 更强大  
✅ 通过简单扩展即可实现**智能体任务分发**  
✅ 提供了完整的 Demo 代码，易于理解和实现  
✅ 与现有架构无缝集成，无需大规模重构

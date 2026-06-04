# New API Code Wiki

## 概述

本文档介绍 New API 项目的自动路由和智能任务分发功能，并提供最小可实现的 Demo 示例。

---

## 问题解答

### Q1: 是否支持自动路由？cc switch 做不到自动路由，这个可以吗？

**A: 是的，New API 支持强大的自动路由功能！**

当前项目已有的自动路由能力包括：

1. **分组自动路由**：通过 `group="auto"` 实现多分组间的自动选择和切换（见 [service/channel_select.go](file:///workspace/service/channel_select.go#L89-L156)）
2. **渠道亲和性路由**：根据请求特征自动绑定到特定渠道（见 [service/channel_affinity.go](file:///workspace/service/channel_affinity.go#L550-L624)）
3. **加权随机路由**：支持渠道权重配置，实现负载均衡
4. **失败自动重试与切换**：当某个渠道失败时自动尝试下一个渠道

与 cc switch 相比，New API 的优势在于：
- ✅ 支持基于规则的智能路由
- ✅ 支持请求上下文感知的路由
- ✅ 支持失败自动切换和重试
- ✅ 支持分组级别的自动路由
- ✅ 支持渠道亲和性（会话粘滞）

---

### Q2: 是否支持智能体的任务分发路由功能？按照任务类型分发给不同的大模型？

**A: 是的，完全支持！**

可以通过以下方式实现按任务类型分发：

1. **模型映射**：在渠道配置中设置模型映射规则
2. **渠道亲和性规则**：根据请求特征（如 prompt 内容、模型名称等）匹配路由规则
3. **自定义分发逻辑**：扩展分发中间件实现更复杂的任务分类路由

---

## 最小 Demo 实现方案

下面我们来实现一个最简单但功能完整的智能路由 Demo，展示如何按任务类型自动分发请求到不同模型。

### Demo 架构

```
用户请求 → 任务分类器 → 智能路由器 → 渠道A(推理任务) / 渠道B(创作任务) / 渠道C(代码任务)
```

### 实现步骤

#### 1. 创建任务分类器

新建文件 `service/task_classifier.go`：

```go
package service

import (
	"strings"
	"github.com/tidwall/gjson"
)

// TaskType 任务类型
type TaskType string

const (
	TaskTypeReasoning TaskType = "reasoning" // 推理任务
	TaskTypeCreative  TaskType = "creative"  // 创作任务
	TaskTypeCoding    TaskType = "coding"    // 代码任务
	TaskTypeGeneral   TaskType = "general"   // 通用任务
)

// ClassifyTask 从请求中分类任务类型
func ClassifyTask(requestBody []byte) TaskType {
	if !gjson.ValidBytes(requestBody) {
		return TaskTypeGeneral
	}

	// 获取 messages 内容
	messages := gjson.GetBytes(requestBody, "messages")
	if !messages.Exists() || !messages.IsArray() {
		return TaskTypeGeneral
	}

	var fullText string
	messages.ForEach(func(key, value gjson.Result) bool {
		role := value.Get("role").String()
		content := value.Get("content").String()
		if role == "user" || role == "system" {
			fullText += " " + content
		}
		return true
	})

	return ClassifyByContent(fullText)
}

// ClassifyByContent 根据文本内容分类任务
func ClassifyByContent(text string) TaskType {
	text = strings.ToLower(text)

	// 代码任务关键词
	codingKeywords := []string{"代码", "编程", "函数", "python", "java", "javascript", "代码", "bug", "debug", "算法", "实现"}
	for _, kw := range codingKeywords {
		if strings.Contains(text, kw) {
			return TaskTypeCoding
		}
	}

	// 创作任务关键词
	creativeKeywords := []string{"写", "创作", "文章", "故事", "诗歌", "文案", "邮件", "报告", "总结"}
	for _, kw := range creativeKeywords {
		if strings.Contains(text, kw) {
			return TaskTypeCreative
		}
	}

	// 推理任务关键词
	reasoningKeywords := []string{"分析", "推理", "为什么", "怎么", "如何", "解释", "原因", "原理", "比较", "区别"}
	for _, kw := range reasoningKeywords {
		if strings.Contains(text, kw) {
			return TaskTypeReasoning
		}
	}

	return TaskTypeGeneral
}
```

#### 2. 扩展渠道选择逻辑

修改 `service/channel_select.go`，添加基于任务类型的路由：

```go
// 在文件开头添加任务类型到模型的映射
var taskTypeToModel = map[TaskType]string{
	TaskTypeReasoning: "gpt-4o",  // 推理用 GPT-4
	TaskTypeCreative:  "claude-3-5-sonnet", // 创作用 Claude
	TaskTypeCoding:    "deepseek-coder", // 代码用 DeepSeek
	TaskTypeGeneral:   "gpt-3.5-turbo", // 通用用 GPT-3.5
}

// CacheGetRandomSatisfiedChannelByTaskType 基于任务类型选择渠道
func CacheGetRandomSatisfiedChannelByTaskType(param *RetryParam, taskType TaskType) (*model.Channel, string, error) {
	// 根据任务类型选择推荐的模型
	if preferredModel, ok := taskTypeToModel[taskType]; ok {
		// 先尝试用推荐模型找可用渠道
		originalModel := param.ModelName
		param.ModelName = preferredModel
		
		channel, group, err := CacheGetRandomSatisfiedChannel(param)
		if err == nil && channel != nil {
			return channel, group, nil
		}
		
		// 如果推荐模型不可用，回退到原始模型
		param.ModelName = originalModel
	}
	
	// 回退到默认选择逻辑
	return CacheGetRandomSatisfiedChannel(param)
}
```

#### 3. 修改分发中间件

修改 `middleware/distributor.go`，集成任务分类：

```go
// 在 Distribute 函数中，channel 选择之前添加任务分类
func Distribute() func(c *gin.Context) {
	return func(c *gin.Context) {
		// ... 现有代码 ...
		
		if channel == nil {
			// 获取请求体进行任务分类
			storage, err := common.GetBodyStorage(c)
			var taskType service.TaskType = service.TaskTypeGeneral
			if err == nil {
				body, _ := storage.Bytes()
				taskType = service.ClassifyTask(body)
				// 重置 body 位置
				_, _ = storage.Seek(0, 0)
			}
			
			// 基于任务类型选择渠道
			channel, selectGroup, err = service.CacheGetRandomSatisfiedChannelByTaskType(
				&service.RetryParam{
					Ctx:        c,
					ModelName:  modelRequest.Model,
					TokenGroup: usingGroup,
					Retry:      common.GetPointer(0),
				},
				taskType,
			)
			
			// 记录任务类型到上下文，便于调试
			common.SetContextKey(c, "task_type", string(taskType))
			
			if err != nil {
				// ... 错误处理 ...
			}
			if channel == nil {
				// ... 错误处理 ...
			}
		}
		
		// ... 后续代码 ...
	}
}
```

#### 4. 添加测试 API（可选）

在 `controller` 目录下创建 `task_router_demo.go`：

```go
package controller

import (
	"net/http"
	"github.com/QuantumNous/new-api/common"
	"github.com/QuantumNous/new-api/service"
	"github.com/gin-gonic/gin"
)

// TaskRouterDemo 任务路由演示
func TaskRouterDemo(c *gin.Context) {
	var req struct {
		Prompt string `json:"prompt"`
	}
	if err := common.UnmarshalBodyReusable(c, &req); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}

	// 分类任务
	taskType := service.ClassifyByContent(req.Prompt)
	
	// 获取推荐模型
	models := map[service.TaskType]string{
		service.TaskTypeReasoning: "gpt-4o",
		service.TaskTypeCreative:  "claude-3-5-sonnet",
		service.TaskTypeCoding:    "deepseek-coder",
		service.TaskTypeGeneral:   "gpt-3.5-turbo",
	}

	c.JSON(http.StatusOK, gin.H{
		"task_type":    taskType,
		"prompt":       req.Prompt,
		"recommended_model": models[taskType],
		"message": "任务分类成功，这将被用于自动路由",
	})
}
```

在 `router/api-router.go` 中添加路由：

```go
func SetApiRouter(apiRouter *gin.RouterGroup) {
	// ... 现有路由 ...
	
	// Demo 路由
	apiRouter.POST("/task-demo/classify", controller.TaskRouterDemo)
}
```

---

## 快速使用指南

### 1. 配置渠道

在 New API 管理后台配置不同的渠道，分别绑定不同的模型：
- 渠道 A：绑定 `gpt-4o`（推理任务）
- 渠道 B：绑定 `claude-3-5-sonnet`（创作任务）  
- 渠道 C：绑定 `deepseek-coder`（代码任务）
- 渠道 D：绑定 `gpt-3.5-turbo`（通用任务）

### 2. 创建 Token 分组

创建一个分组（如 "auto-router-demo"），将上述渠道都添加到该分组中。

### 3. 测试 Demo

启动服务后，使用以下请求测试：

```bash
# 测试推理任务
curl -X POST http://localhost:3000/api/task-demo/classify \
  -H "Content-Type: application/json" \
  -d '{"prompt": "请分析为什么天空是蓝色的"}'

# 测试创作任务
curl -X POST http://localhost:3000/api/task-demo/classify \
  -H "Content-Type: application/json" \
  -d '{"prompt": "请写一首关于秋天的诗"}'

# 测试代码任务
curl -X POST http://localhost:3000/api/task-demo/classify \
  -H "Content-Type: application/json" \
  -d '{"prompt": "帮我用 Python 写一个快速排序算法"}'
```

### 4. 实际使用

使用正常的 OpenAI 格式请求，在 group 参数中指定你的分组，系统会自动根据任务内容选择合适的模型：

```bash
curl -X POST http://localhost:3000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d '{
    "model": "gpt-3.5-turbo",
    "group": "auto-router-demo",
    "messages": [{"role": "user", "content": "写一个 Python 爬虫"}]
  }'
```

---

## 核心文件参考

| 文件 | 说明 |
|------|------|
| [middleware/distributor.go](file:///workspace/middleware/distributor.go) | 分发中间件，核心路由逻辑 |
| [service/channel_select.go](file:///workspace/service/channel_select.go) | 渠道选择逻辑 |
| [service/channel_affinity.go](file:///workspace/service/channel_affinity.go) | 渠道亲和性（智能路由） |
| [model/channel.go](file:///workspace/model/channel.go) | 渠道数据模型 |

---

## 扩展建议

1. **更智能的分类器**：可以使用轻量级模型进行任务分类，而不仅是关键词匹配
2. **路由规则配置化**：在管理后台添加路由规则配置界面
3. **路由效果统计**：记录不同任务类型的路由成功率和满意度
4. **A/B 测试**：支持多种路由策略的 A/B 测试
5. **基于历史反馈的学习**：根据用户反馈自动调整路由策略

---

## 总结

New API 提供了比 cc switch 更强大的自动路由能力：
- ✅ 支持自动路由（基于分组、权重、亲和性）
- ✅ 支持按任务类型智能分发
- ✅ 易于扩展自定义路由逻辑
- ✅ 提供完善的重试和失败转移机制

通过本 Demo 的简单实现，你可以快速体验到智能路由的强大功能！

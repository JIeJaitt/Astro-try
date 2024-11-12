---
title: Fiber
description: Fiber
pubDate: 12 11 2024
categories:
  - tech
tags:
  - go
  - fiber
---

帮我把这个函数合二为一：
```go
// FromContext returns the request ID from context.
// If there is no request ID, an empty string is returned.
func FromContext(c fiber.Ctx) string {
	if rid, ok := c.Locals(requestIDKey).(string); ok {
		return rid
	}
	return ""
}

// FromUserContext returns the request ID from the UserContext.
// If there is no request ID, an empty string is returned.
// Compared to Local, UserContext is more suitable for transmitting requests between microservices
func FromUserContext(ctx context.Context) string {
	if rid, ok := ctx.Value(requestIDKey).(string); ok {
		return rid
	}
	return ""
}
```

您可以将这两个函数合并为一个，通过接受一个通用的接口参数，根据参数的类型进行处理。下面是合并后的函数：

```go
// FromAnyContext 返回上下文中的请求 ID。
// 如果没有请求 ID，则返回空字符串。
func FromAnyContext(c interface{}) string {
	switch ctx := c.(type) {
	case fiber.Ctx:
		if rid, ok := ctx.Locals(requestIDKey).(string); ok {
			return rid
		}
	case context.Context:
		if rid, ok := ctx.Value(requestIDKey).(string); ok {
			return rid
		}
	default:
		// 可以根据需要处理其他类型的上下文
	}
	return ""
}
```

这样，您就可以使用同一个函数来获取不同类型上下文中的请求 ID。例如：

```go
rid := FromAnyContext(c) // c 可以是 fiber.Ctx 或 context.Context
```

## 还有更好的写法嘛

是的，您可以采用更优雅的方式，将这两个函数合并为一个，并且避免使用空接口和类型断言。由于 `fiber.Ctx` 提供了 `UserContext()` 方法，可以获取关联的 `context.Context`，您可以统一从 `context.Context` 中获取请求 ID。

下面是优化后的代码：

```go
// FromContext 返回上下文中的请求 ID。
// 如果没有请求 ID，则返回空字符串。
func FromContext(ctx context.Context) string {
	if rid, ok := ctx.Value(requestIDKey).(string); ok {
		return rid
	}
	return ""
}
```

在使用时：

- 对于标准的 `context.Context`，直接传入即可：

  ```go
  rid := FromContext(ctx)
  ```

- 对于 `fiber.Ctx`，先获取其 `UserContext()`，然后传入：

  ```go
  rid := FromContext(c.UserContext())
  ```

这样，您就可以统一使用一个函数，从 `context.Context` 中获取请求 ID，无需进行类型断言或处理不同的上下文类型。这种方法提高了代码的简洁性和可维护性。

## 这样的话通过Local存进去的key可以被拿出来嘛

您说得很对，通过 `c.Locals` 存储的键值对无法直接通过 `c.UserContext().Value()` 获取。因为在 Fiber 中，`c.Locals` 和 `c.UserContext()` 是两个独立的存储机制。`c.Locals` 是用于存储请求范围内的数据，而 `c.UserContext()` 返回的是与请求关联的标准 `context.Context`，默认情况下它们并不共享数据。

因此，之前我建议的方案无法满足您的需求，非常抱歉。为了将这两个函数合二为一，同时确保能够从 `c.Locals` 和 `context.Context` 中获取请求 ID，您可以考虑以下方法：

### 方法一：接受两个参数，尝试从 `c.Locals` 和 `ctx.Value` 中获取

```go
// FromContext 返回上下文中的请求 ID。
// 如果没有请求 ID，则返回空字符串。
func FromContext(c fiber.Ctx, ctx context.Context) string {
    if c != nil {
        if rid, ok := c.Locals(requestIDKey).(string); ok {
            return rid
        }
    }
    if ctx != nil {
        if rid, ok := ctx.Value(requestIDKey).(string); ok {
            return rid
        }
    }
    return ""
}
```

**使用示例：**

- 如果有 `fiber.Ctx`：

  ```go
  rid := FromContext(c, c.UserContext())
  ```

- 如果只有 `context.Context`：

  ```go
  rid := FromContext(nil, ctx)
  ```

### 方法二：统一使用 `context.Context`，并在中间件中同步 `c.Locals` 到 `c.UserContext()`

如果您希望统一只使用 `context.Context`，可以在设置请求 ID 的地方（例如中间件）将请求 ID 同时存储到 `c.UserContext()` 中。

**在中间件中：**

```go
rid := generateRequestID()
c.Locals(requestIDKey, rid)
ctx := context.WithValue(c.UserContext(), requestIDKey, rid)
c.SetUserContext(ctx)
```

**合并后的函数：**

```go
// FromContext 返回 context.Context 中的请求 ID。
// 如果没有请求 ID，则返回空字符串。
func FromContext(ctx context.Context) string {
    if rid, ok := ctx.Value(requestIDKey).(string); ok {
        return rid
    }
    return ""
}
```

**使用示例：**

```go
rid := FromContext(c.UserContext())
```

**注意：**这种方法要求您在中间件中确保请求 ID 被存储到 `c.UserContext()` 中。

### 方法三：创建一个接口，支持从不同类型的上下文中获取请求 ID

如果您需要更通用的解决方案，可以定义一个接口，并为不同的上下文类型实现该接口。

```go
type RequestIDGetter interface {
    GetRequestID() string
}

func FromContext(rg RequestIDGetter) string {
    return rg.GetRequestID()
}
```

**为 `fiber.Ctx` 实现接口：**

```go
func (c fiber.Ctx) GetRequestID() string {
    if rid, ok := c.Locals(requestIDKey).(string); ok {
        return rid
    }
    return ""
}
```

**为 `context.Context` 实现接口：**

```go
type ContextWrapper context.Context

func (ctx ContextWrapper) GetRequestID() string {
    if rid, ok := ctx.Value(requestIDKey).(string); ok {
        return rid
    }
    return ""
}
```

**使用示例：**

- 对于 `fiber.Ctx`：

  ```go
  rid := FromContext(c)
  ```

- 对于 `context.Context`：

  ```go
  rid := FromContext(ContextWrapper(ctx))
  ```

### 结论

由于 `c.Locals` 和 `c.UserContext()` 是独立的存储机制，直接从 `c.UserContext()` 获取 `c.Locals` 中的值是不行的。您需要根据实际情况选择合适的方法：

- **方法一**适用于需要同时处理 `fiber.Ctx` 和 `context.Context` 的场景。
- **方法二**适用于希望统一使用 `context.Context`，并且可以修改中间件逻辑的场景。
- **方法三**提供了一种面向接口的解决方案，使代码更加灵活和可扩展。

希望这些方案能帮助您合并函数，实现您想要的功能。

## 我觉得还是采用这种写法吧
// FromContext returns the request ID from context.
// If there is no request ID, an empty string is returned.
func FromContext(c interface{}) string {
	switch ctx := c.(type) {
	case fiber.Ctx:
		if rid, ok := ctx.Locals(requestIDKey).(string); ok {
			return rid
		}
	case context.Context:
		if rid, ok := ctx.Value(requestIDKey).(string); ok {
			return rid
		}
	default:
		return ""
	}
}


好的，这种写法可以实现您的需求。通过使用空接口 `interface{}` 作为参数，并在函数内部使用类型断言，您可以在同一个函数中处理 `fiber.Ctx` 和 `context.Context` 两种类型的上下文。这使得代码更加简洁和通用。

您的合并函数如下：

```go
// FromContext returns the request ID from context.
// If there is no request ID, an empty string is returned.
func FromContext(c interface{}) string {
	switch ctx := c.(type) {
	case fiber.Ctx:
		if rid, ok := ctx.Locals(requestIDKey).(string); ok {
			return rid
		}
	case context.Context:
		if rid, ok := ctx.Value(requestIDKey).(string); ok {
			return rid
		}
	default:
		return ""
	}
	return ""
}
```

**使用示例：**

```go
rid := FromContext(c) // c 可以是 fiber.Ctx 或 context.Context
```

**注意事项：**

- **类型安全性**：使用 `interface{}` 和类型断言会降低类型安全性。在传递参数时，需要确保参数确实是 `fiber.Ctx` 或 `context.Context`，否则可能会导致运行时错误或返回空字符串。
- **代码可读性**：虽然这种方法减少了函数的数量，但在团队协作中，明确的函数签名可能有助于代码的可读性和可维护性。
- **错误处理**：您可以在 `default` 分支中添加日志或错误处理，以便在传入不支持的类型时进行提示。

**改进建议：**

如果您希望代码更加健壮，可以在函数中添加对未知类型的警告：

```go
func FromContext(c interface{}) string {
	switch ctx := c.(type) {
	case fiber.Ctx:
		if rid, ok := ctx.Locals(requestIDKey).(string); ok {
			return rid
		}
	case context.Context:
		if rid, ok := ctx.Value(requestIDKey).(string); ok {
			return rid
		}
	default:
		// 添加警告或日志
		log.Printf("Unsupported context type: %T", c)
	}
	return ""
}
```

这样，当传入不支持的类型时，您可以收到提示，有助于调试和维护。

**总结：**

您的方案有效地将两个函数合并为一个，简化了代码结构。在确保类型安全和代码可读性的前提下，这种方法可以提高代码的灵活性和可维护性。

## 测试代码你看看怎么优化比较好
```go
// go test -run Test_RequestID_Locals
func Test_RequestID_FromContext(t *testing.T) {
	t.Parallel()
	reqID := "ThisIsARequestId"

	app := fiber.New()
	app.Use(New(Config{
		Generator: func() string {
			return reqID
		},
	}))

	var ctxVal string

	app.Use(func(c fiber.Ctx) error {
		ctxVal = FromContext(c)
		return c.Next()
	})

	_, err := app.Test(httptest.NewRequest(fiber.MethodGet, "/", nil))
	require.NoError(t, err)
	require.Equal(t, reqID, ctxVal)
}

// go test -run Test_RequestID_FromUserContext
func Test_RequestID_FromUserContext(t *testing.T) {
	t.Parallel()
	reqID := "ThisIsARequestId"

	app := fiber.New()
	app.Use(New(Config{
		Generator: func() string {
			return reqID
		},
	}))

	var ctxVal string

	app.Use(func(c fiber.Ctx) error {
		ctxVal = FromContext(c.Context())
		return c.Next()
	})

	_, err := app.Test(httptest.NewRequest(fiber.MethodGet, "/", nil))
	require.NoError(t, err)
	require.Equal(t, reqID, ctxVal)
}
```

好的，我来看一下您的测试代码，并提出优化建议。

### 您的测试代码分析

您提供了两个测试函数：

1. **Test_RequestID_FromContext**

   - 使用 `FromContext(c)` 从 `fiber.Ctx` 中获取请求 ID。
   - 验证获取的请求 ID 是否与预期的 `reqID` 相等。

2. **Test_RequestID_FromUserContext**

   - 使用 `FromContext(c.Context())` 从 `context.Context` 中获取请求 ID。
   - 验证获取的请求 ID 是否与预期的 `reqID` 相等。

这两个测试函数的结构和逻辑非常相似，唯一区别在于传递给 `FromContext` 函数的参数不同。

### 优化建议

为了减少代码重复，提高测试的可维护性和可读性，您可以考虑以下优化：

1. **使用表驱动测试**

   将两个测试用例合并到一个测试函数中，使用表驱动测试的方法。这样可以方便地添加更多的测试用例，同时减少代码重复。

2. **提取公共代码**

   将重复的初始化和验证逻辑提取到辅助函数或匿名函数中。

### 优化后的代码

以下是优化后的测试代码，采用了表驱动测试的方法：

```go
func Test_RequestID_FromContext(t *testing.T) {
	t.Parallel()

	type args struct {
		input interface{}
	}

	tests := []struct {
		name       string
		inputFunc  func(c fiber.Ctx) interface{}
	}{
		{
			name: "From fiber.Ctx",
			inputFunc: func(c fiber.Ctx) interface{} {
				return c
			},
		},
		{
			name: "From context.Context",
			inputFunc: func(c fiber.Ctx) interface{} {
				return c.Context()
			},
		},
	}

	for _, tt := range tests {
		tt := tt // 注意需要重新绑定 tt 变量，以避免闭包问题
		t.Run(tt.name, func(t *testing.T) {
			t.Parallel()

			reqID := "ThisIsARequestId"

			app := fiber.New()
			app.Use(New(Config{
				Generator: func() string {
					return reqID
				},
			}))

			var ctxVal string

			app.Use(func(c fiber.Ctx) error {
				ctxVal = FromContext(tt.inputFunc(c))
				return c.Next()
			})

			_, err := app.Test(httptest.NewRequest(fiber.MethodGet, "/", nil))
			require.NoError(t, err)
			require.Equal(t, reqID, ctxVal)
		})
	}
}
```

### 优化要点说明

1. **使用表驱动测试**

   - 定义一个测试用例的切片 `tests`，其中包含测试名称和一个函数 `inputFunc`，用于生成传递给 `FromContext` 的参数。
   - 使用 `t.Run` 来运行子测试，子测试的名称为测试用例的名称。

2. **处理闭包变量**

   - 在循环内部使用 `tt := tt` 来重新绑定变量，以避免闭包引用导致的测试用例混淆。

3. **并行测试**

   - 使用 `t.Parallel()` 来并行运行测试，提高测试速度。

4. **代码简洁性**

   - 通过提取公共逻辑，减少了代码重复，提高了测试的可读性和可维护性。

### 进一步的改进

如果您预计会有更多类似的测试用例，或者需要对更多的上下文类型进行测试，可以进一步优化测试结构。例如，您可以将输入参数和期望的输出封装得更加通用。

### 完整的优化代码

```go
package main

import (
	"net/http/httptest"
	"testing"

	"github.com/gofiber/fiber/v2"
	"github.com/stretchr/testify/require"
)

// 假设 requestIDKey 和 FromContext 已经定义

func Test_RequestID_FromContext(t *testing.T) {
	t.Parallel()

	reqID := "ThisIsARequestId"

	type args struct {
		inputFunc func(c fiber.Ctx) interface{}
	}

	tests := []struct {
		name string
		args args
	}{
		{
			name: "From fiber.Ctx",
			args: args{
				inputFunc: func(c fiber.Ctx) interface{} {
					return c
				},
			},
		},
		{
			name: "From context.Context",
			args: args{
				inputFunc: func(c fiber.Ctx) interface{} {
					return c.Context()
				},
			},
		},
	}

	for _, tt := range tests {
		tt := tt // 重新绑定变量
		t.Run(tt.name, func(t *testing.T) {
			t.Parallel()

			app := fiber.New()
			app.Use(New(Config{
				Generator: func() string {
					return reqID
				},
			}))

			var ctxVal string

			app.Use(func(c fiber.Ctx) error {
				ctxVal = FromContext(tt.args.inputFunc(c))
				return c.Next()
			})

			_, err := app.Test(httptest.NewRequest(fiber.MethodGet, "/", nil))
			require.NoError(t, err)
			require.Equal(t, reqID, ctxVal)
		})
	}
}
```

### 注意事项

- **线程安全**：在并行测试中，确保没有共享的可变状态。如果有，需要使用锁或其他同步机制。
- **代码可读性**：虽然表驱动测试可能会增加一些复杂性，但在测试用例较多时，可以显著减少代码重复，提高可维护性。

### 总结

通过上述优化，您可以：

- 减少测试代码的重复，提高可维护性。
- 使用表驱动测试，方便添加新的测试用例。
- 保持测试的清晰性和可读性。

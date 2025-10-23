# API参考

<cite>
**本文档引用的文件**
- [handler.go](file://api/handler.go)
- [request.go](file://model/request.go)
- [response.go](file://model/response.go)
- [search_service.go](file://service/search_service.go)
- [config.go](file://config/config.go)
</cite>

## 目录
1. [搜索端点概述](#搜索端点概述)
2. [HTTP方法与URL路径](#http方法与url路径)
3. [请求参数](#请求参数)
4. [请求头要求](#请求头要求)
5. [认证机制](#认证机制)
6. [请求JSON Schema](#请求json-schema)
7. [响应JSON Schema](#响应json-schema)
8. [MCP服务接口](#mcp服务接口)
9. [客户端调用示例](#客户端调用示例)
10. [HTTP状态码](#http状态码)
11. [错误响应格式](#错误响应格式)
12. [参数校验规则](#参数校验规则)

## 搜索端点概述

搜索端点是本系统的核心功能，提供统一的RESTful接口用于执行跨平台资源搜索。该端点支持从Telegram频道和多个第三方插件来源获取数据，能够合并处理并返回结构化的搜索结果。端点设计灵活，支持多种参数配置，允许客户端精确控制搜索行为、结果类型和数据来源。

**Section sources**
- [handler.go](file://api/handler.go#L20-L206)
- [search_service.go](file://service/search_service.go#L350-L509)

## HTTP方法与URL路径

搜索端点支持两种HTTP方法，以适应不同的使用场景：

- **GET方法**: 适用于简单的搜索请求，所有参数通过URL查询字符串传递。这种方式便于在浏览器中直接调用和调试。
- **POST方法**: 适用于复杂的搜索请求，参数通过请求体以JSON格式传递。这种方式支持更复杂的数据结构，特别是`ext`扩展参数。

无论使用哪种方法，URL路径保持一致。

**URL路径**: `/api/search`

**Section sources**
- [handler.go](file://api/handler.go#L20-L206)
- [router.go](file://api/router.go#L30-L31)

## 请求参数

搜索端点支持以下请求参数，这些参数控制搜索的行为和返回结果的格式。

| 参数名 | 类型 | 必填 | 描述 | 默认值 |
| :--- | :--- | :--- | :--- | :--- |
| `kw` | 字符串 | 是 | 搜索关键词，不能为空。 | 无 |
| `channels` | 字符串数组 | 否 | 指定要搜索的Telegram频道列表，以逗号分隔。 | 配置文件中的`DefaultChannels` |
| `conc` | 整数 | 否 | 搜索的并发数量，控制同时执行的搜索任务数。 | 配置文件中的`DefaultConcurrency` |
| `refresh` | 布尔值 | 否 | 是否强制刷新，不使用缓存。设为`true`时将忽略缓存，直接发起新请求。 | `false` |
| `res` | 字符串 | 否 | 结果类型，控制响应中包含的数据。可选值：`all`, `results`, `merge`。 | `merge` |
| `src` | 字符串 | 否 | 数据来源类型，决定搜索的范围。可选值：`all`, `tg`, `plugin`。 | `all` |
| `plugins` | 字符串数组 | 否 | 指定要使用的插件列表，以逗号分隔。仅当`src`为`plugin`或`all`时有效。 | 搜索所有启用的插件 |
| `cloud_types` | 字符串数组 | 否 | 指定要返回的网盘类型列表，以逗号分隔。 | 返回所有类型 |
| `ext` | 对象 | 否 | 扩展参数，一个JSON对象，用于向特定插件传递自定义参数。 | `{}` |

**参数互斥逻辑**:
- 当 `src=tg` 时，`plugins` 参数将被忽略。
- 当 `src=plugin` 时，`channels` 参数将被忽略。
- 当 `src=all` 且 `plugins` 未指定时，`plugins` 参数将被设为 `nil`。

**Section sources**
- [request.go](file://model/request.go#L3-L13)
- [handler.go](file://api/handler.go#L20-L206)

## 请求头要求

此API端点对请求头没有特殊要求。客户端可以使用标准的`Content-Type`头来指定请求体的格式。

- **对于POST请求**: 建议设置 `Content-Type: application/json`。
- **对于GET请求**: 无需设置特定的请求头。

服务器会正确处理没有`Content-Type`头的请求。

**Section sources**
- [handler.go](file://api/handler.go#L20-L206)

## 认证机制

当前版本的搜索端点**不包含任何认证机制**。该API是公开的，任何能够访问服务器的客户端都可以调用。

**重要提示**: 在生产环境中部署时，建议在反向代理（如Nginx）或API网关层面添加认证层，以保护API免受未授权访问。

**Section sources**
- [handler.go](file://api/handler.go#L20-L206)

## 请求JSON Schema

此Schema定义了通过POST方法发送的请求体的结构。

```json
{
  "type": "object",
  "required": ["kw"],
  "properties": {
    "kw": {
      "type": "string",
      "description": "搜索关键词"
    },
    "channels": {
      "type": "array",
      "items": {
        "type": "string"
      },
      "description": "搜索的频道列表"
    },
    "conc": {
      "type": "integer",
      "description": "并发搜索数量"
    },
    "refresh": {
      "type": "boolean",
      "description": "强制刷新，不使用缓存"
    },
    "res": {
      "type": "string",
      "enum": ["all", "results", "merge"],
      "description": "结果类型"
    },
    "src": {
      "type": "string",
      "enum": ["all", "tg", "plugin"],
      "description": "数据来源类型"
    },
    "plugins": {
      "type": "array",
      "items": {
        "type": "string"
      },
      "description": "指定搜索的插件列表"
    },
    "ext": {
      "type": "object",
      "additionalProperties": true,
      "description": "扩展参数，用于传递给插件的自定义参数"
    },
    "cloud_types": {
      "type": "array",
      "items": {
        "type": "string"
      },
      "description": "指定返回的网盘类型列表"
    }
  }
}
```

**Section sources**
- [request.go](file://model/request.go#L3-L13)

## 响应JSON Schema

此Schema定义了API响应的结构。所有响应都遵循统一的格式。

### 通用响应格式

```json
{
  "type": "object",
  "properties": {
    "code": {
      "type": "integer",
      "description": "状态码，0表示成功"
    },
    "message": {
      "type": "string",
      "description": "状态信息"
    },
    "data": {
      "oneOf": [
        { "$ref": "#/definitions/SearchResponse" },
        {}
      ],
      "description": "响应数据，成功时为SearchResponse，失败时可能为空"
    }
  },
  "required": ["code", "message"]
}
```

### 搜索响应格式 (SearchResponse)

```json
{
  "type": "object",
  "properties": {
    "total": {
      "type": "integer",
      "description": "结果总数"
    },
    "results": {
      "type": "array",
      "items": { "$ref": "#/definitions/SearchResult" },
      "description": "原始搜索结果列表"
    },
    "merged_by_type": {
      "type": "object",
      "additionalProperties": {
        "type": "array",
        "items": { "$ref": "#/definitions/MergedLink" }
      },
      "description": "按网盘类型合并的链接"
    }
  },
  "required": ["total"]
}
```

### 搜索结果格式 (SearchResult)

```json
{
  "type": "object",
  "properties": {
    "message_id": {
      "type": "string",
      "description": "消息ID"
    },
    "unique_id": {
      "type": "string",
      "description": "全局唯一ID"
    },
    "channel": {
      "type": "string",
      "description": "频道名称"
    },
    "datetime": {
      "type": "string",
      "format": "date-time",
      "description": "消息发布时间"
    },
    "title": {
      "type": "string",
      "description": "消息标题"
    },
    "content": {
      "type": "string",
      "description": "消息内容"
    },
    "links": {
      "type": "array",
      "items": { "$ref": "#/definitions/Link" },
      "description": "网盘链接列表"
    },
    "tags": {
      "type": "array",
      "items": { "type": "string" },
      "description": "标签列表"
    },
    "images": {
      "type": "array",
      "items": { "type": "string" },
      "description": "图片链接列表"
    }
  },
  "required": ["message_id", "channel", "title", "links"]
}
```

### 网盘链接格式 (Link)

```json
{
  "type": "object",
  "properties": {
    "type": {
      "type": "string",
      "description": "网盘类型"
    },
    "url": {
      "type": "string",
      "format": "uri",
      "description": "网盘链接"
    },
    "password": {
      "type": "string",
      "description": "链接密码"
    }
  },
  "required": ["type", "url"]
}
```

### 合并链接格式 (MergedLink)

```json
{
  "type": "object",
  "properties": {
    "url": {
      "type": "string",
      "format": "uri",
      "description": "网盘链接"
    },
    "password": {
      "type": "string",
      "description": "链接密码"
    },
    "note": {
      "type": "string",
      "description": "备注信息"
    },
    "datetime": {
      "type": "string",
      "format": "date-time",
      "description": "消息发布时间"
    },
    "source": {
      "type": "string",
      "description": "数据来源：tg:频道名 或 plugin:插件名"
    },
    "images": {
      "type": "array",
      "items": { "type": "string" },
      "description": "图片链接列表"
    }
  },
  "required": ["url"]
}
```

**Section sources**
- [response.go](file://model/response.go#L45-L49)
- [response.go](file://model/response.go#L38-L42)
- [response.go](file://model/response.go#L12-L22)
- [response.go](file://model/response.go#L5-L9)
- [response.go](file://model/response.go#L25-L32)

## MCP服务接口

MCP（Multi-Channel Plugin）服务接口是系统内部的核心服务，由`SearchService`结构体实现。它负责协调和执行跨多个数据源的搜索任务。

### 功能
- **并行搜索**: 同时向Telegram频道和多个插件发起搜索请求，提高响应速度。
- **结果合并**: 将来自不同来源的结果进行智能合并，去重并保留最完整的信息。
- **缓存管理**: 利用两级缓存（内存+磁盘）机制，显著提升重复搜索的性能。
- **智能排序**: 根据时间、关键词优先级和插件等级对结果进行综合排序。

### 调用方式
`SearchService`是一个Go语言的结构体，其`Search`方法是主要的入口点。该方法被`SearchHandler`调用，接收从HTTP请求解析出的参数。

```go
func (s *SearchService) Search(
    keyword string, 
    channels []string, 
    concurrency int, 
    forceRefresh bool, 
    resultType string, 
    sourceType string, 
    plugins []string, 
    cloudTypes []string, 
    ext map[string]interface{}
) (model.SearchResponse, error)
```

**Section sources**
- [search_service.go](file://service/search_service.go#L200-L202)
- [search_service.go](file://service/search_service.go#L350-L509)

## 客户端调用示例

以下是使用不同编程语言和工具调用搜索端点的示例。

### 使用curl (GET方法)

```bash
curl -G "http://localhost:8888/api/search" \
  --data-urlencode "kw=电影" \
  --data-urlencode "channels=tgsearchers3,tgsearchers4" \
  --data-urlencode "conc=10" \
  --data-urlencode "refresh=false" \
  --data-urlencode "res=merge" \
  --data-urlencode "src=all" \
  --data-urlencode "plugins=pan666,qupansou" \
  --data-urlencode "cloud_types=baidu,quark" \
  --data-urlencode "ext={\"site\":\"example.com\"}"
```

### 使用JavaScript fetch (POST方法)

```javascript
const response = await fetch('http://localhost:8888/api/search', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
  },
  body: JSON.stringify({
    kw: '电视剧',
    channels: ['tgsearchers3'],
    conc: 5,
    refresh: false,
    res: 'all',
    src: 'plugin',
    plugins: ['miaoso'],
    cloud_types: ['aliyun'],
    ext: { "timeout": 30 }
  })
});

const data = await response.json();
console.log(data);
```

### 使用Python requests (POST方法)

```python
import requests

url = "http://localhost:8888/api/search"
payload = {
    "kw": "纪录片",
    "channels": ["tgsearchers3"],
    "conc": 8,
    "refresh": True,
    "res": "results",
    "src": "all",
    "plugins": ["panwiki"],
    "cloud_types": ["123pan"],
    "ext": {"proxy": "socks5://127.0.0.1:1080"}
}

response = requests.post(url, json=payload)
print(response.json())
```

**Section sources**
- [handler.go](file://api/handler.go#L20-L206)

## HTTP状态码

API会根据请求的处理情况返回相应的HTTP状态码。

| 状态码 | 含义 | 说明 |
| :--- | :--- | :--- |
| `200 OK` | 成功 | 请求已成功处理，响应体中包含搜索结果。 |
| `400 Bad Request` | 错误的请求 | 请求参数无效或格式错误。例如，`kw`参数缺失，或`ext`参数不是有效的JSON。 |
| `500 Internal Server Error` | 内部服务器错误 | 服务器在处理请求时发生错误。例如，搜索服务内部出现异常。 |

**Section sources**
- [handler.go](file://api/handler.go#L20-L206)

## 错误响应格式

当API返回非200状态码时，会返回一个标准化的错误响应。

```json
{
  "code": 400,
  "message": "无效的ext参数格式: invalid character 'x' looking for beginning of value"
}
```

- **`code`**: 与HTTP状态码对应的整数，提供更具体的错误代码。
- **`message`**: 人类可读的错误描述，通常包含调试信息。

**Section sources**
- [response.go](file://model/response.go#L61-L66)
- [handler.go](file://api/handler.go#L20-L206)

## 参数校验规则和边界情况处理

`SearchHandler`函数在调用`SearchService`之前，会对所有参数进行严格的校验和预处理。

### 校验规则
- **`kw`参数**: 必填。如果为空，会返回400错误。
- **`channels`参数**: 如果未提供，将使用配置文件中的`DefaultChannels`。
- **`conc`参数**: 如果小于等于0，将使用配置文件中的`DefaultConcurrency`。
- **`refresh`参数**: 只有当值为`"true"`时才被视为`true`，其他任何值（包括`"false"`）都视为`false`。
- **`res`参数**: 如果为空，将默认为`"merge"`，并在内部转换为`"merged_by_type"`。
- **`src`参数**: 如果为空，将默认为`"all"`。
- **`plugins`和`cloud_types`参数**: 支持逗号分隔的字符串，会自动分割并去除首尾空格。
- **`ext`参数**: 必须是有效的JSON字符串。如果为`"{}"`，则解析为空对象。如果为空字符串，则解析为空对象。

### 边界情况处理
- **空参数**: 对于可选的数组参数（如`channels`, `plugins`），空字符串或不存在的参数被视为未指定，将使用默认值或`nil`。
- **参数互斥**: 严格遵守`src`参数与其他参数的互斥逻辑，确保搜索范围的正确性。
- **并发控制**: 即使请求的并发数很高，系统也会根据配置和服务器负载进行合理控制。
- **缓存一致性**: 即使`forceRefresh=false`，系统也会确保从缓存中获取到最新的数据，避免返回过时的结果。

**Section sources**
- [handler.go](file://api/handler.go#L20-L206)
- [search_service.go](file://service/search_service.go#L350-L509)
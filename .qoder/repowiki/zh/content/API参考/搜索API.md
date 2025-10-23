# 搜索API

<cite>
**本文档引用的文件**
- [request.go](file://model/request.go)
- [response.go](file://model/response.go)
- [handler.go](file://api/handler.go)
- [search_service.go](file://service/search_service.go)
- [config.go](file://config/config.go)
</cite>

## 目录
1. [简介](#简介)
2. [端点规范](#端点规范)
3. [请求参数](#请求参数)
4. [请求与响应模型](#请求与响应模型)
5. [参数行为说明](#参数行为说明)
6. [客户端调用示例](#客户端调用示例)
7. [错误处理与状态码](#错误处理与状态码)

## 简介
搜索API提供了一个RESTful接口，用于在多个数据源中执行关键词搜索。该API支持从Telegram频道和各种插件获取结果，并提供灵活的参数来控制搜索行为、结果格式和数据源。API设计为高性能和可扩展，支持并发搜索和结果缓存。

**Section sources**
- [handler.go](file://api/handler.go#L1-L207)

## 端点规范
搜索API通过`/search`端点提供服务，支持GET和POST两种HTTP方法。

### HTTP方法
- **GET**: 通过URL查询参数传递搜索请求
- **POST**: 通过请求体（JSON格式）传递搜索请求

### URL路径
```
/search
```

### 请求头
- **Content-Type**: 对于POST请求，必须设置为`application/json`
- 无特殊认证机制，API为公开访问

### 认证机制
本API无需认证，为公开访问接口。

**Section sources**
- [handler.go](file://api/handler.go#L35-L207)

## 请求参数
以下参数可用于控制搜索行为。

### 基础参数
| 参数 | 类型 | 必需 | 默认值 | 描述 |
|------|------|------|--------|------|
| `kw` | string | 是 | 无 | 搜索关键词 |
| `channels` | string数组 | 否 | 配置中的默认频道 | 指定搜索的Telegram频道列表，多个频道用逗号分隔 |
| `conc` | int | 否 | 配置中的默认并发数 | 并发搜索数量 |
| `refresh` | boolean | 否 | false | 强制刷新，不使用缓存 |
| `res` | string | 否 | "merge" | 结果类型：all(返回所有结果)、results(仅返回results)、merge(仅返回merged_by_type) |
| `src` | string | 否 | "all" | 数据来源类型：all(全部来源)、tg(仅Telegram)、plugin(仅插件) |
| `plugins` | string数组 | 否 | 所有插件 | 指定搜索的插件列表，多个插件用逗号分隔 |
| `cloud_types` | string数组 | 否 | 所有类型 | 指定返回的网盘类型列表，多个类型用逗号分隔 |
| `ext` | object | 否 | {} | 扩展参数，用于传递给插件的自定义参数 |

### 参数互斥逻辑
- 当`src=tg`时，忽略`plugins`参数
- 当`src=plugin`时，忽略`channels`参数
- 当`src=all`时，如果`plugins`为空或未指定，则搜索所有插件

**Section sources**
- [request.go](file://model/request.go#L3-L13)
- [handler.go](file://api/handler.go#L35-L207)

## 请求与响应模型
### 请求模型 (SearchRequest)
```json
{
  "kw": "string",
  "channels": ["string"],
  "conc": 0,
  "refresh": false,
  "res": "string",
  "src": "string",
  "plugins": ["string"],
  "ext": {},
  "cloud_types": ["string"]
}
```

### 响应模型 (Response)
```json
{
  "code": 0,
  "message": "string",
  "data": {}
}
```

### 搜索响应 (SearchResponse)
```json
{
  "total": 0,
  "results": [
    {
      "message_id": "string",
      "unique_id": "string",
      "channel": "string",
      "datetime": "2023-01-01T00:00:00Z",
      "title": "string",
      "content": "string",
      "links": [
        {
          "type": "string",
          "url": "string",
          "password": "string"
        }
      ],
      "tags": ["string"],
      "images": ["string"]
    }
  ],
  "merged_by_type": {
    "string": [
      {
        "url": "string",
        "password": "string",
        "note": "string",
        "datetime": "2023-01-01T00:00:00Z",
        "source": "string",
        "images": ["string"]
      }
    ]
  }
}
```

**Section sources**
- [request.go](file://model/request.go#L3-L13)
- [response.go](file://model/response.go#L3-L66)

## 参数行为说明
### `refresh`参数对缓存行为的影响
`refresh`参数控制是否使用缓存结果：
- 当`refresh=true`时，强制刷新，不使用任何缓存，重新执行搜索
- 当`refresh=false`或未指定时，优先从缓存获取结果，如果缓存命中则直接返回缓存数据

缓存机制在Telegram搜索和插件搜索中均有应用，缓存键基于关键词和相关参数生成。

### `res`参数控制的返回格式差异
`res`参数控制响应中包含的数据：
- `all`: 返回完整的`SearchResponse`，包含`total`、`results`和`merged_by_type`
- `results`: 只返回`results`数组和`total`计数
- `merge`或`merged_by_type`: 只返回按网盘类型分组的`merged_by_type`和`total`计数

### `src`指定数据源的逻辑
`src`参数控制搜索的数据来源：
- `all`: 搜索Telegram频道和所有启用的插件
- `tg`: 仅搜索Telegram频道
- `plugin`: 仅搜索插件

**Section sources**
- [search_service.go](file://service/search_service.go#L350-L509)
- [search_service.go](file://service/search_service.go#L1146-L1215)
- [search_service.go](file://service/search_service.go#L1218-L1365)

## 客户端调用示例
### curl示例
```bash
# GET请求
curl "http://localhost:8888/search?kw=电影&channels=tgsearchers3&res=merge"

# POST请求
curl -X POST http://localhost:8888/search \
  -H "Content-Type: application/json" \
  -d '{
    "kw": "电视剧",
    "channels": ["tgsearchers3"],
    "res": "all",
    "src": "all"
  }'
```

### JavaScript fetch示例
```javascript
// POST请求
fetch('/search', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
  },
  body: JSON.stringify({
    kw: '纪录片',
    channels: ['tgsearchers3'],
    res: 'results',
    src: 'all'
  })
})
.then(response => response.json())
.then(data => console.log(data));
```

### Python requests示例
```python
import requests

# POST请求
response = requests.post('http://localhost:8888/search', json={
    'kw': '动漫',
    'channels': ['tgsearchers3'],
    'res': 'merge',
    'src': 'all'
})

print(response.json())
```

**Section sources**
- [handler.go](file://api/handler.go#L35-L207)

## 错误处理与状态码
### 参数校验规则
- `kw`参数为必需，如果未提供将返回400错误
- 所有参数在GET请求中从查询参数获取，在POST请求中从JSON请求体获取
- `ext`参数必须为有效的JSON格式

### 默认值填充机制
- `channels`: 如果未指定，使用配置中的`DefaultChannels`
- `conc`: 如果未指定或小于等于0，使用配置中的`DefaultConcurrency`
- `res`: 如果未指定，默认为"merge"
- `src`: 如果未指定，默认为"all"

### 边界情况处理
- **空关键词**: 如果`kw`为空，将返回400错误
- **分页越界**: 本API无分页参数，不涉及分页越界问题
- **无效参数**: 对于无效的枚举值（如`res`），将使用默认行为

### HTTP状态码
| 状态码 | 含义 | 说明 |
|--------|------|------|
| 200 | OK | 请求成功，返回搜索结果 |
| 400 | Bad Request | 请求参数错误，如缺少必需参数或JSON格式无效 |
| 429 | Too Many Requests | 未使用，当前无速率限制 |
| 500 | Internal Server Error | 服务器内部错误，搜索失败 |

**Section sources**
- [handler.go](file://api/handler.go#L35-L207)
- [config.go](file://config/config.go#L1-L516)
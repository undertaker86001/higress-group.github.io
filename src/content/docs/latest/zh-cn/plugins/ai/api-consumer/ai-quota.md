---
title: AI 配额管理
keywords: [ AI网关, AI配额 ]
description: AI 配额管理插件配置参考
---

## 功能说明

`ai-quota` 插件实现给特定 consumer 根据分配固定的 quota 进行 quota 策略限流，同时支持 quota 管理能力，包括查询 quota 、刷新 quota、增减 quota。

`ai-quota` 插件需要配合 认证插件比如 `key-auth`、`jwt-auth` 等插件获取认证身份的 consumer 名称，同时需要配合 `ai-statistics` 插件获取 AI Token 统计信息。

## 运行属性

插件执行阶段：`默认阶段`
插件执行优先级：`750`

## 配置说明

| 名称                 | 数据类型            | 填写要求                                 | 默认值 | 描述                                         |
|--------------------|-----------------|--------------------------------------| ---- |--------------------------------------------|
| `redis_key_prefix` | string          |  选填                                     |   chat_quota:   | qutoa redis key 前缀                         |
| `admin_consumer`   | string          | 必填                                   |      | 管理 quota 管理身份的 consumer 名称                 |
| `admin_path`       | string          | 选填                                   |   /quota   | 管理 quota 请求 path 前缀                        |
| `redis`            | object          | 是                                    |      | redis相关配置                                  |

`redis`中每一项的配置字段说明

| 配置项       | 类型   | 必填 | 默认值                                                     | 说明                        |
| ------------ | ------ | ---- | ---------------------------------------------------------- | --------------------------- |
| service_name | string | 必填 | -                                                          | redis 服务名称，带服务类型的完整 FQDN 名称，例如 my-redis.dns、redis.my-ns.svc.cluster.local     |
| service_port | int    | 否   | 服务类型为固定地址（static service）默认值为80，其他为6379 | 输入redis服务的服务端口     |
| username     | string | 否   | -                                                          | redis用户名                 |
| password     | string | 否   | -                                                          | redis密码                   |
| timeout      | int    | 否   | 1000                                                       | redis连接超时时间，单位毫秒 |



## 配置示例

### 识别请求参数 apikey，进行区别限流
```yaml
redis_key_prefix: "chat_quota:"
admin_consumer: consumer3
admin_path: /quota
redis:
  service_name: redis-service.default.svc.cluster.local
  service_port: 6379
  timeout: 2000
```


###  刷新 quota

如果当前请求 url 的后缀符合 admin_path，例如插件在 example.com/v1/chat/completions 这个路由上生效，那么更新 quota 可以通过

```bash
curl https://example.com/v1/chat/completions/quota/refresh -H "Authorization: Bearer credential3" -d "consumer=consumer1&quota=10000"
```

Redis 中 key 为 chat_quota:consumer1 的值就会被刷新为 10000


### 查询 quota

查询特定用户的 quota 可以通过 

```bash
curl https://example.com/v1/chat/completions/quota?consumer=consumer1 -H "Authorization: Bearer credential3"
```

将返回： {"quota": 10000, "consumer": "consumer1"}

### 增减 quota 

增减特定用户的 quota 可以通过 

```bash
curl https://example.com/v1/chat/completions/quota/delta -d "consumer=consumer1&value=100" -H "Authorization: Bearer credential3"
```

这样 Redis 中 Key 为 chat_quota:consumer1 的值就会增加100，可以支持负数，则减去对应值。

### 配置根据模型参数路由时如何访问

当路由配置根据请求 body 中的 model 参数路由生效时，按上述方式访问 quota API 会出现 404 的情况，因为 quota API 并不遵循 openai 的 body 协议，无法提取出 model 参数。

此时若希望命中匹配特定 model 参数的路由，可以通过附加 `x-higress-llm-model` 请求头来实现，例如：

```bash
### 以刷新 quota 为例，其他也是一样
curl https://example.com/v1/chat/completions/quota/refresh -H 'x-higress-llm-model: deepseek-r1' -H "Authorization: Bearer credential3" -d "consumer=consumer1&quota=10000" 
```

这样就可以命中模型参数匹配 `deepseek-r1` 的路由，当然还需要注意路由本身的 Method 和路径匹配规则，可以匹配到 quota API

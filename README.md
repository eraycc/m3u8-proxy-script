# M3U8 Proxy Filter Script
# 项目说明文档

# 代理地址说明
脚本内示例的代理地址`https://proxy.mengze.vip/proxy/`已失效，请自行部署cf代理或其他代理进行替换，cf代理部署参考[cloudflare-safeproxy](https://github.com/eraycc/cloudflare-safeproxy)，部署后，将代理地址替换为API代理地址，URL编码配置为false(也可自行修改cf代理脚本代码，使其支持URL编码)。
## 项目概述

M3U8 Proxy Filter Script 是一个多语言实现的 HLS (HTTP Live Streaming) 代理过滤脚本，支持Nodejs、Cloudflare Worker(cf Pages)、Deno 和 PHP 环境。脚本提供 M3U8 播放链接的去广告、代理加速功能，并支持多种 HLS 协议特性。

## 功能特性

### 核心功能
- **代理重写**：使用代理获取 M3U8 文件并重写其中的 TS/fMP4 分片 URL
- **EXT-X-MAP 支持**：完整支持初始化段代理
- **加密流处理**：支持 EXT-X-KEY 加密流处理
- **discontinuity 标记过滤**：可配置是否过滤 discontinuity 标记
- **缓存支持**：
  - PHP：本地文件缓存
  - Deno：内存缓存
  - Cloudflare Worker：
    worker.js: KV 存储版（需设置变量名称为 `M3U8_PROXY_KV`）
    worker-chache.js: 基于worker的边缘网络自带cache缓存，可以直接调用，直接缓存，无限制且无需配置(感谢L站edwa佬友提供的思路)

### 高级功能
- **主播放列表解析**：自动解析主播放列表（带递归深度限制）
- **非 M3U8 内容处理**：
  - 音视频/图片文件：使用 TS 代理跳转加速
  - 其他内容：直接跳转原始 URL
- **双代理设置**：全部脚本支持双代理配置
- **广告处理**：支持 M3U8 全局加速及去除广告标记

## 部署与使用
```
deno 部署：
fork该项目，打开deno面板，导入复刻的项目，Entrypoint填写deno.ts

cf worker 部署：
新建kv后，绑定时设置变量名称为M3U8_PROXY_KV，复制worker.js到新建worker内部署。** worker-cache版不用配置kv **

PHP 部署：
把PHP脚本复制到PHP(CURL)环境服务器，设置伪静态规则
```

### 通用调用方式
```
https://deployurl/?url=https://test-streams.mux.dev/x36xhzz/x36xhzz.m3u8
或
https://deployurl/m3u8filter/https://test-streams.mux.dev/x36xhzz/x36xhzz.m3u8
```

### 环境特定配置

#### PHP 环境
```
m3u8filter.php?url=m3u8link
或
m3u8filter.php/m3u8filter/m3u8link
或设置伪静态后
m3u8filter/m3u8link
```

**PHP伪静态规则配置**：

**Nginx**:
```nginx
rewrite ^/m3u8filter/(https?):/(.*)$ /m3u8filter.php?url=$1://$2 last;
```

**Apache**:
```apache
<IfModule mod_rewrite.c>
    RewriteEngine On
    RewriteRule ^m3u8filter/(https?):/(.*)$ m3u8filter.php?url=$1://$2 [L,QSA]
</IfModule>
```

## 技术优化

### 性能优化
- 基于哈希的缓存键，减少缓存冲突
- 请求超时处理，避免服务卡死
- 流处理改进，减少内存占用

### 功能增强
- 智能广告检测和过滤，标记可能的广告片段
- 增强对各种 HLS 标签的支持（如 EXT-X-BYTERANGE）
- CORS 标头设置完善
- 安全头添加，提高安全性
- 支持保留原始响应头的关键信息

## 架构设计

### 模块化设计
- 每个函数负责单一功能
- 详细的错误处理和日志记录
- 配置项分组和详细说明

### 错误处理
- 统一的错误响应创建
- 详细的错误信息反馈
- 全局异常捕获，防止服务中断

### 日志与调试
- 可配置日志开关(debug)
- 请求和响应的详细日志
- 时间戳和结构化日志输出

## 配置参数

### 通用配置项
| 参数名 | 类型 | 默认值 | 描述 |
|--------|------|--------|------|
| `url` | string | 必填 | 要处理的 M3U8 文件 URL |
| `PROXY_URL` | string | 可选 | 主代理服务器地址 |
| `PROXY_TS` | string | 可选 | TS视频流代理服务器地址 |
| `FILTER_DISCONTINUITY` | boolean | `true` | 是否过滤 discontinuity 标记 |
| `CACHE_TTL` | number | `3600` | 缓存时间（秒） |
| `MAX_RECURSION` | number | `3` | 最大重定向深度 |

### 脚本环境特定配置
- **Cloudflare Worker**：需配置 `M3U8_PROXY_KV` KV 存储绑定
- **PHP**：需配置可写的缓存目录
- **Deno**：内存级缓存，可直接部署使用

## 开发指南

### 构建与部署

#### Cloudflare Worker
worker.js
1. 创建新的 Worker 项目
2. 绑定 KV 命名空间（名称为 `M3U8_PROXY_KV`）
3. 部署脚本

worker-cache.js
1. 创建新的 Worker 项目
2. 复制代码
3. 部署脚本

#### 如何将worker部署到cf pages？
第一种：fork该项目，修改你要部署的cf worker脚本名为`_worker.js`，在cfpage中导入fork的仓库，如果是kv缓存还需要配置kv变量，如果是cache版则直接部署。

第二种：下载cf worker脚本，重命名为_worker.js，并打包成_worker.js.zip
在 Cloudflare Pages 控制台中选择 上传资产后，为你的项目取名后点击 创建项目，然后上传你压缩好的 _worker.js.zip 文件后点击 部署站点。
部署完成后点击 继续处理站点 后，选择 设置 > 环境变量 > 制作为生产环境定义变量 > 添加KV变量（如果不是cf kv版可不用设置），点击保存。
返回 部署 选项卡，在右下角点击 创建新部署 后，重新上传 _worker.js.zip 文件后点击 保存并部署 即可。

#### Deno
fork该项目,在deno控制面板导入fork的项目，安装和部署命令参考下方代码块，
选择边缘节点自带cache api版(deno-cache.js)和内存缓存版(deno.js)都可以，Entrypoint配置项填写deno-cache.js或deno.ts，部署后返回Github action打开deploy进行授权
```bash
安装
deno install -gArf jsr:@deno/deployctl
部署
deployctl deploy
```

#### PHP
1. 上传 `m3u8filter.php` 到 web 服务器
2. 配置伪静态规则（可选）
3. 确保缓存目录可写

## 安全注意事项

1. 建议部署后启用 HTTPS
2. 定期更新
3. 监控异常请求
4. 对速率进行限制

## 故障排除

### 常见问题
1. **播放失败**：
   - 检查原始 URL 是否可访问
   - 验证代理服务器配置
   - 检查缓存是否过期
   - 检查脚本处理逻辑

2. **广告过滤不生效**：
   - 确认广告标记检测规则
   - 检查 M3U8 文件结构是否有变化

3. **性能问题**：
   - 调整缓存 TTL
   - 检查代理服务器性能
   - 优化网络连接

## 未来计划

1. 可能没有未来计划，不出问题不会更新

## 贡献指南

欢迎通过 Issue 和 Pull Request 贡献代码。提交前请确保：
1. 代码符合项目风格
2. 通过基本测试
3. 更新相关文档

## 许可证

本项目采用 Apache-2.0 许可证开源。

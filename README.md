# iFinD (同花顺 HTTP API) 模块使用说明

## 模块介绍

在进行金融量化分析时，获取实时打点与盘后复权数据是第一步。`ifind` 模块对接了同花顺数据终端的 iFinD HTTP API 接口，将其丰富的数据网关无缝集成到 DolphinDB 中。用户不再需要编写 C++ 插件、Python 进程或者维护跨平台环境，只要通过这个原生脚本模块即可调用任何复杂的 iFinD 高阶数据，并将回传的 JSON 数据直接展开为结构化的 DolphinDB 数据表 (`Table`)。

## 第三方插件依赖

本模块利用了 DolphinDB 自带的高性能网络插件作为引擎：
- `httpClient` 插件：用于发起 HTTP POST 请求与同花顺 API 鉴权服务器通信并极速获取数据报文。

**前置操作：**
请确保在使用前在集群或本地加载了网络通信组件：
```dolphindb
installPlugin("httpClient");
loadPlugin("httpClient");
```

## 前期准备（获取免密码 Token）

iFinD HTTP 模块强制要求用户不能在脚本里面暴露账号密码。请按照以下步骤生成永久通行证：
1. 账号准备：拥有一套带量化 API (EmQuant / HTTP) 权限的同花顺终端账号（或申请的子账号）。
2. 获取 `refresh_token`：
   - 使用 Windows 上安装好的《iFinD 超级命令》客户端。
   - 使用账号密码登录后，点击顶部工具栏 **工具 -> refresh_token查询/更新**。
   - 提取出属于您的超级长效刷新密钥，这个 `refresh_token` 的寿命与账号期限等长。
   
## 接口介绍

模块将所有最核心的功能封装为统一的调用接口。

### getApiData

**语法**
```dolphindb
ifind::getApiData(refreshToken, apiName, params=NULL, columns=[])
```

#### 详情
通用的网关执行槽。不仅会自动帮您判断 Access Token 的期限去置换新的 Token，还会全量打包和发送数据，处理 iFinD 底层的 `"tables"` 多证券混合包。

#### 参数
- **refreshToken**: 字符串。提取出来的同花顺账号刷新口令。
- **apiName**: 字符串。您要获取的接口路由端点，同花顺支持以下主要后缀：
  - `"basic_data_service"`: 基础数据
  - `"date_sequence"`: 日期序列
  - `"cmd_history_quotation"`: 历史行情
  - `"high_frequency"`: 高频序列、日内分时
  - `"real_time_quotation"`: 实时行情快照
- **params**: 字典(`DICTIONARY`)。按照 iFinD 说明手册中指定的查询规范（例如 `codes`, `indicators`, `startdate`）。
- **columns**: 字符串数组。在返回的 Table 中精确提取需要的列。

#### 示例：获取历史周线下发数据

```dolphindb
loadPlugin("httpClient")
use ifind

// 1. 设置长效通行证
refreshToken = "eyJzaWduX3RpbWUiOiIy...填充您的长效凭据..." 

// 2. 组装查询参数
api = "cmd_history_quotation"
p = dict(STRING, ANY)
p["codes"] = "300033.SZ,600030.SH"
p["indicators"] = "open,close,volume"
p["startdate"] = "20230101"
p["enddate"] = "20231231"

funcPara = dict(STRING, STRING)
funcPara["Interval"] = "W" // 周线
funcPara["CPS"] = "1"      // 不复权
p["functionpara"] = funcPara

// 3. 通信与解析 (返回的是标准 DolphinDB Table)
resultTb = ifind::getApiData(refreshToken, api, p)

select * from resultTb where thscode="300033.SZ"
```

## 后期计划与已知限制 (Roadmap & Limitations)

### 1. 中文编码适配 (Unicode Decoding)
目前模块在解析包含中文的 JSON 时（如 `THS_BD` 基础数据接口），由于 DolphinDB 原生 `parseExpr` 的限制，中文字符可能会显示为 `uXXXX` 格式（如 `u8D35u5DDEu8305u53F0`）。

*   **推荐方案**: 手动安装 DolphinDB 官方的 `json` 插件（`installPlugin("json")`），并使用 `fromJson` 替代脚本逻辑。
*   **高阶方案**: `EncoderDecoder` 插件可基于规则高效解析 JSON。
    *   **限制**: `EncoderDecoder` 插件目前仅支持 **Linux x86-64** 和 **Linux ARM**。由于当前 iFinD 客户端环境常处于 Windows，该插件暂不可用。
*   **计划**: 未来将在 Linux 计算节点上引入自动探测逻辑，优先利用高性能插件处理中文转码。

### 2. 多标的并行化
目前 `getApiData` 处理大批量证券代码时是串行发送。计划引入 `pli` 或异步任务队列，支持在 DolphinDB Server 端并发请求多个证券的历史序列，进一步提升万级标的的拉取效率。

### 3. 数据类型自动转换
目前接口透传同花顺原始数据类型（多为 `DOUBLE` 或 `ANY`）。未来计划根据 `datatype` 字段映射，自动将日期字符串、布尔值等转换为 DolphinDB 原生日期类型 (`DATE`, `DATETIME`)。

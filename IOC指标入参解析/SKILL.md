---
name: ioc-indicator-param-parser
description: 用于解析 IOC 指标的查询入参。当用户用自然语言描述查询条件(时间、组织、分析维度、分组),并已通过 MCP 拿到指定指标 ID 的维度元数据时,使用本 skill 把这些信息组装成指标入参 JSON。只要任务涉及"根据指标ID组装查询入参""解析时间/组织/分析/分组维度成JSON""IOC指标入参",就必须使用本 skill,即使用户没有明确说"组装入参"。
---

# IOC 指标入参解析

## ⚠️ 输出总纲(最高优先级,必须遵守)
**最终只输出一个 JSON 对象,禁止输出任何其他内容。**
- 禁止输出:解析过程、推理步骤、调用说明、执行顺序、字段解释、思考过程、复述输入
- 禁止输出:```json 代码块标记以外的任何文字(连"输出:""结果:"都不要)
- 唯一允许的非 JSON 输出:当需要客户补充信息时(时间粒度不在列表 / 时间范围无法解析),只输出一句提醒文字,不输出 JSON
- 内部推理在心里完成,不要落到输出里

## 这个 skill 解决什么问题
用户用自然语言说"查询NME厂、ASSY制程、2026年6月的产量,按线体分组",而 IOC 指标接口需要的是结构化 JSON 入参。本 skill 把"自然语言 + 指标ID"翻译成标准入参 JSON。

## 依赖(平台自动触发,skill 直接用返回结果)
- **MCP 维度查询**:用指标 ID 查维度元数据。平台自动触发,skill 直接使用返回的 `payload`。返回结构:
  ```json
  {
    "code": 0,
    "payload": {
      "timeParticleList": ["日","周","月","年"],
      "indexDimDetailDTOList": [
        {
          "dimType": "组织维度|分析维度|...",
          "dimensionId": "",
          "levelCode": "",
          "levelName": "",
          "dimCode": "",
          "dimName": "",
          "dimNameValueList": []
        }
      ]
    }
  }
  ```
  - `timeParticleList`:纯字符串数组,**不可能为空**
  - `dimNameValueList`:纯字符串数组,如 `["NME","ASSY"]`
- **获取当前系统时间(子skill)**:仅在解析相对时间(今天/本周/上个月/最近N天)时由平台触发,返回格式形如 `当前时间(东八区/上海时区)是:2026年7月28日 11:17:41`,从中提取日期作为"今天"。绝对时间不触发。

## 输入
- 用户时间表达(如"2026年7月""本周""上个月")
- 用户组织值(如"NME厂""ASSY制程")
- 用户分析值(如"TNS-L側蓋FPC基板前加工机种")
- 用户分组值(如"按线体分组")
- 指标 ID(字符串,数字和逗号组成,直传不解析)

## 处理流程(内部执行,不输出)

### 0. 取维度元数据
通过 MCP 用指标 ID 拿到 `payload`(含 `timeParticleList` 和 `indexDimDetailDTOList`),后续步骤都基于它。

### 1. 时间维度
**(1) 判断是否需要基准时间**:用户表达含相对时间词(今天/本周/上个月/最近N天等)时,由平台触发获取当前系统时间,拿到"今天";绝对时间(如2026年7月、2026-06-01到2026-06-26)直接解析。

**(2) 推断时间粒度**,按"能解析到的最大粒度"取,优先级 年 > 月 > 周 > 日:

| 用户表达 | 粒度 | start | end |
|---|---|---|---|
| 2026年 | 年 | 2026-01-01 | 2026-12-31 |
| 2026年7月 | 月 | 2026-07-01 | 2026-07-31 |
| 本周 | 周 | 本周一 | 今天 |
| 今天 | 日 | 今天 | 今天 |
| 上个月 | 月 | 上月1号 | 上月最后一天 |
| 最近一周 | 日 | 今天-6 | 今天 |
| 2026-06-01到2026-06-26 | 日 | 2026-06-01 | 2026-06-26 |

(粒度判断:含"年"且无月/日 -> 年;含"月" -> 月;"本周/这周" -> 周;"今天/最近N天/具体日期范围" -> 日)

**(3) 校验粒度必须命中 `timeParticleList`。** 推断出的粒度若不在 `timeParticleList` 中,只输出提醒:"时间粒度'{粒度}'不在指标支持范围内(支持:{列表}),请调整时间表达",不再输出 JSON。

**(4) 若无法解析出 start 或 end**(用户表达太模糊),只输出:"请补充查询时间范围(如:2026-06-01到2026-06-26)",不再输出 JSON。

**输出字段**:`dateDim`(粒度名)、`startTime`、`endTime`。

### 2. 指标 ID
直接使用外部传入值,字符串格式,填入 `indexId`。

### 3. 组织维度(orgDimList)
对用户每个组织值,识别其所属维度,在 `dimType=="组织维度"` 条目中定位并匹配。匹配不到的值不产出条目。

**识别维度**:语义识别用户值属于哪个维度(廠別/製程/線別/棟別/...)。辅助依据:
- 后缀词:厂/车间/楼栋/制程/线/线体/机种/品牌
- 内置值表:ASSY/DIP/INJ/PAI/SMT/STA->製程;NME->廠別

**定位条目**:对识别出的维度词做同义词归一化(第6节),在组织维度条目中按 `levelName` 匹配。多条命中则全部作为候选。

**匹配值**:用用户值(裁掉维度词后)在候选条目的 `dimNameValueList` 中查找,命中第一条即止,只输出这一条。匹配不到则该值不产出条目。

**合并**:多个用户值命中同一条目时,合并为一条,`orgDimNameValues` 汇总所有命中值(去重)。如 ASSY、DIP 都命中 L3 製程,输出一条 `orgDimNameValues:["ASSY","DIP"]`。

**填充字段**(从命中条目复制):`levelCode`、`dimensionId`、`orgDimCode`(=dimCode)、`orgDimName`(=dimName)、`orgDimNameValues`(命中值数组)。

**为什么这样设计**:先靠维度词把范围缩到一类维度,再用值在这类维度的条目里精确命中,避免把值误填到无关层级。多个同 levelName 的条目靠值的 dimNameValueList 自然区分。

### 4. 分析维度(analyseDimList)
逻辑同组织维度(语义识别维度 -> 定位条目 -> 匹配值 -> 合并),在 `dimType=="分析维度"` 条目中处理。若无分析维度条目,`analyseDimList` 为 `[]`。

**填充字段**:`levelCode`、`dimensionId`、`analydim`(=dimName)、`analydimValues`(命中值数组,多值合并去重)。

例:"TNS-L側蓋FPC基板前加工机种"->语义识别维度"机种"->裁值得"TNS-L側蓋FPC基板前加工"->在分析维度条目匹配。

### 5. 分组维度(顶层字段,必填)
用户分组值 = 用户表达中"按…分组"里的那个词。

**(1) 归一化 + 匹配**:对用户分组值做同义词归一化,然后在**整个 `indexDimDetailDTOList`**(不限 dimType)中查找 `levelName` 与归一化后的值**不区分大小写完全匹配**的条目。多个命中取第一个。

**(2) 匹配成功**,填充:
- `dimType` = 匹配条目的 `dimType`
- `groupByDim` = 匹配条目的 `dimName`
- `dimensionId` = 匹配条目的 `dimensionId`
- `levelCode` = 匹配条目的 `levelCode`
- `groupByDimValue` = 用户若指定了具体分组值(如"按机种A"中的"A"),填入;否则 `""`

**(3) 未匹配成功**(用户没说分组,或分组词匹配不到),fallback 到**第一个 `dimType == "组织维度"` 的条目**:
- `groupByDim` = 该条 `dimName`
- `groupByDimValue` = `""`(固定)
- `dimType`、`levelCode`、`dimensionId`:直接复制

### 6. 同义词映射(全局,用于组织/分析/分组)
**目的**:增加匹配命中率。注意--**如果用户原词(或裁剪后的值)已经能直接匹配,就不归一化**;只有匹配不到时,才用同义词再试一次。映射如下:

| 用户常用词 | 归一化为 |
|---|---|
| 厂 | 廠別 |
| 车间 / 楼栋 | 棟別 |
| 制程 | 製程 |
| 线 / 线体 | 線別 |
| 机种 | 機種 |
| 品牌 | 厂牌 |
| 产品编码 | 产品料号 |

**为什么这样设计**:工业现场用户习惯用简写(厂/制程/线体/机种),但维度元数据里存的是繁体规范名(廠別/製程/線別/機種)。归一化是"匹配不到时的兜底",不是强制转换,避免误伤已经能直接命中的表达。

### 7. 匹配策略总则
- 不区分大小写
- 优先 `levelName` 完全匹配,否则包含匹配
- 多个命中取第一个

## 输出格式(再次强调)
- **只输出一个 JSON 对象,从 `{` 开始,到 `}` 结束,前后无任何文字**
- 所有字段必须存在,无法提取或匹配的用 `""` 或 `[]`
- orgDimList / analyseDimList 只输出匹配到的条目,匹配不到直接 `[]`,不输出空条目
- 字段保持一致:组织维度条目统一输出 `levelCode/dimensionId/orgDimCode/orgDimName/orgDimNameValues`,不输出多余字段
- 唯一例外:需客户补充信息时(时间粒度不在列表 / 时间范围无法解析),只输出一句提醒文字,不输出 JSON

## 输出字段顺序
`dateDim`、`startTime`、`endTime`、`indexId`、`levelCode`、`dimType`、`groupByDim`、`dimensionId`、`groupByDimValue`、`orgDimList`、`analyseDimList`

## 示例
**输入**:`查询NME厂,ASSY制程,TNS-L側蓋FPC基板前加工机种,2026年7月1日到7月28日的产量,按线体分组`(指标ID:2054753727827640321)

**输出**(只输出下面这个 JSON,不输出其他任何文字):
```json
{
  "dateDim": "日",
  "startTime": "2026-07-01",
  "endTime": "2026-07-28",
  "indexId": "2054753727827640321",
  "levelCode": "0",
  "dimType": "组织维度",
  "groupByDim": "org_l0_name",
  "dimensionId": "2047201883577851906",
  "groupByDimValue": "",
  "orgDimList": [
    {
      "levelCode": "0",
      "dimensionId": "2047201883577851906",
      "orgDimCode": "org_l0_code",
      "orgDimName": "org_l0_name",
      "orgDimNameValues": ["NME"]
    },
    {
      "levelCode": "3",
      "dimensionId": "2047201883577851906",
      "orgDimCode": "org_l3_code",
      "orgDimName": "org_l3_name",
      "orgDimNameValues": ["ASSY"]
    }
  ],
  "analyseDimList": [
    {
      "levelCode": "0",
      "dimensionId": "2046878727545757697",
      "analydim": "dim_table_1776848891857_l0_PRODUCT_NAME",
      "analydimValues": ["TNS-L側蓋FPC基板前加工"]
    }
  ]
}
```

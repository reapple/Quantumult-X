# Quantumult X 资源解析器参数化 UI 协议（草案 v1）

本协议定义资源解析器（resource parser）脚本与 Quantumult X 客户端之间，关于 **URL `#` hash 参数图形化编辑** 的接口约定。

通过该协议，解析器作者可声明脚本支持哪些 hash 参数、对应的 UI 控件类型，由 Quantumult X 客户端动态渲染参数编辑页。用户在 UI 中所做的修改会被序列化回 URL hash 后字符串。

> 协议方向：**作者实现，native 调用。**

> 客户端版本要求：**Quantumult X 1.5.6 (build 918)** 及以上。

> 嫌细节太长？直接跳到 [§8 完整样板 JS](#8-完整样板-js) 复制粘贴到脚本最前面，按需增删字段即可。


## 1. 入口约定

作者在脚本顶层为 `$parser` 全局对象挂载三个函数：

```js
typeof $parser === 'object' &&
typeof $parser.hashSchema === 'function' &&
typeof $parser.hashToUI === 'function' &&
typeof $parser.uiToHash === 'function'
```

任一函数缺失，参数化编辑入口将不可用。


## 2. `$parser` 注入对象

`$parser` 是预先创建好的全局空对象，作者在脚本顶层为其挂载三个函数：

```js
$parser.hashSchema = function () { ... };
$parser.hashToUI   = function (hash) { ... };
$parser.uiToHash   = function (values) { ... };
```

三个函数均为 **零或单参数**，**不传 context**——作者需要读取环境信息时直接使用现有全局对象（`$environment` / `$resource` 等）。


## 3. 三个函数的职责

### 3.1 `$parser.hashSchema()`

返回作者支持的全部参数 schema，**不携带具体 value**。

返回结构：

```js
{
  version: 1,
  sections: [ <Group> ... ]
}
```

### 3.2 `$parser.hashToUI(hash)`

入参：当前 `#` 后字符串（不含 `#`，可能为空字符串）。

返回：仅包含 hash 中**已经存在**的参数，每项**带 `value`**。

返回结构：

```js
{
  version: 1,
  sections: [ <Group> ... ],
  unknown: "raw=stuff&iDontKnow=..." // 可选，作者未识别的参数原样字符串
}
```

> hash 为空字符串时返回 `sections: []`。

### 3.3 `$parser.uiToHash(values)`

入参：UI 当前所有参数值的扁平字典：

```js
{
  emoji: "1",
  in:    ["香港", "台湾"],
  cert:  true,
  rename:"旧@新",
  __unknown: "raw=stuff&iDontKnow=..." // 透传未识别参数（如有）
}
```

返回：新的 hash 字符串（不含 `#`）。

> URL 编码、参数顺序、未识别参数拼接位置——**完全由作者决定**。


## 4. JSON 数据结构

### 4.1 顶层

```ts
{
  version: 1,           // 协议版本，当前固定 1
  sections: Group[],    // 按 section 渲染（每个 group 一个 section）
  unknown?: string      // hashToUI 专属，可选
}
```

### 4.2 Group 容器

每个 group 对应 tableView 中的一个 section。

```ts
{
  type: "group",
  title?: string,        // section header（可选）
  description?: string,  // section footer（可选）
  items: Control[]
}
```

### 4.3 Control 控件

通用字段：

```ts
{
  type: "switch" | "select" | "text" | "tags",
  key: string,            // hash 参数名
  label: string,          // cell 主标题
  description?: string,   // 用于 footer 描述
  value?: any             // 仅在 hashToUI 中携带；hashSchema 不带
}
```

> `hashSchema` 返回时**不带 `value`**；`hashToUI` 返回时**必须带 `value`**。


## 5. 四种控件类型

### 5.1 `switch`

二态开关。

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `onValue` | string | 是 | 开启时写入 hash 的值 |
| `offValue` | string | 是 | 关闭时写入 hash 的值 |
| `value` | bool | hashToUI 时必填 | `true` = on，`false` = off |

序列化语义：**只要参数存在于 values 中，就写入 hash**（写 `onValue` 或 `offValue`）。off 也写。作者不返回该参数 → 不写。

示例：

```js
{
  type: "switch", key: "cert", label: "TLS 证书验证",
  onValue: "1", offValue: "-1", value: true
}
```

### 5.2 `select`

多档枚举（任意档数）。

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `options` | `{label, value}[]` | 是 | 选项列表，至少 2 项 |
| `value` | string | hashToUI 时必填 | 当前选中项的 value（必须出现在 options 中） |

示例：

```js
{
  type: "select", key: "emoji", label: "Emoji 旗帜",
  description: "添加/删除节点名地区旗帜",
  options: [
    { label: "添加",     value: "1"  },
    { label: "国行设备", value: "2"  },
    { label: "删除",     value: "-1" }
  ],
  value: "1"
}
```

### 5.3 `text`

任意单行字符串。键值映射 / 复合结构 / base64 / 正则 等"由作者在 `description` 中说明语法"的场景，统一用 `text`。

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `placeholder` | string | 否 | 编辑页的占位符 |
| `keyboard` | `"default" \| "url" \| "numeric"` | 否 | 输入键盘类型，默认 `"default"` |
| `value` | string | hashToUI 时必填 | 当前值 |

示例：

```js
{
  type: "text", key: "regex", label: "正则保留",
  description: "对节点完整信息进行正则匹配，仅保留命中的节点",
  placeholder: "如：iplc",
  value: "iplc"
}
```

### 5.4 `tags`

`+` 分隔的多值数组。

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `placeholder` | string | 否 | 添加新值时输入页的占位符 |
| `value` | string[] | hashToUI 时必填 | 当前值数组 |

> 序列化由作者在 `uiToHash` 中处理。常见做法：`values.in.join("+")`。

示例：

```js
{
  type: "tags", key: "in", label: "保留节点",
  description: "按节点名关键字保留，每行一个关键字",
  placeholder: "如：香港",
  value: ["香港", "台湾"]
}
```


## 6. 未识别参数透传

用户可能在主 URL 输入框手写了作者**不认识**的参数。helper 必须保证这些参数在编辑过程中**不丢失**。

约定：

- `hashToUI(hash)` 返回值的 `unknown` 字段携带未识别参数的原始字符串（如 `"customParam=foo&debug=true"`）
- `uiToHash(values)` 入参中的 `values.__unknown` 包含同一字符串
- `uiToHash` 由作者负责把 `values.__unknown` 拼接到返回的 hash 字符串中（位置自定，建议放末尾）


## 7. 三个函数必须自包含

三个函数将被独立运行——脚本中其它顶层定义的变量、函数、模块在调用时**都不可见**。规则：

- 三个函数体内**不能**引用外部全局变量、外部全局函数、闭包变量
- 所有 helper 必须**定义在函数体内部**
- 三个 `$parser` 函数之间相互调用是 OK 的

**反例**（会在调用时抛 ReferenceError）：

```js
function _myHelper() { return [...]; }   // 顶层定义，提取时不会被带过来

$parser.hashSchema = function () {
  return { sections: [{ items: [
    { type: "select", options: _myHelper() }   // ❌ ReferenceError
  ] }] };
};
```

**正例**：

```js
$parser.hashSchema = function () {
  function _myHelper() { return [...]; }   // ✅ 嵌套定义，自包含

  return { sections: [{ items: [
    { type: "select", options: _myHelper() }
  ] }] };
};
```


## 8. 完整样板 JS

> 以下样板基于 [KOP-XIAO/QuantumultX 资源解析器](https://raw.githubusercontent.com/KOP-XIAO/QuantumultX/master/Scripts/resource-parser.js)（©𝐒𝐡𝐚𝐰𝐧）的参数集编写，覆盖其主要 hash 参数。其他解析器作者请将以下代码 **copy-paste 到脚本最前面**，按需增删字段、调整 label 与 description。

```js
// ==== Parser Helper UI ====
var $parser = $parser || {};

// schema：作者声明的全部参数（不带 value）
$parser.hashSchema = function () {
  // helper 必须定义在函数体内部 —— 协议要求三个函数自包含
  function _emojiOptions() {
    return [
      { label: "添加",     value: "1"  },
      { label: "国行设备", value: "2"  },
      { label: "删除",     value: "-1" }
    ];
  }
  function _ptnOptions() {
    return [
      { label: "原样",         value: ""  },
      { label: "🅰 字母方块",   value: "1" },
      { label: "🄰 字母方块（实）", value: "2" },
      { label: "𝐀 加粗",        value: "3" },
      { label: "𝗮 加粗小写",    value: "4" },
      { label: "𝔸 双线",        value: "5" },
      { label: "𝕒 双线小写",    value: "6" },
      { label: "ᵃ 上标",        value: "7" },
      { label: "ᴬ 大写上标",    value: "8" }
    ];
  }
  function _nptOptions() {
    return [
      { label: "原样",         value: ""  },
      { label: "①",            value: "1" },
      { label: "❶",            value: "2" },
      { label: "⓵",            value: "3" },
      { label: "𝟙",            value: "4" },
      { label: "¹",            value: "5" },
      { label: "₁",            value: "6" },
      { label: "𝟏",            value: "7" },
      { label: "𝟷",            value: "8" }
    ];
  }

  return {
    version: 1,
    sections: [
      {
        type: "group",
        title: "节点筛选",
        items: [
          { type: "tags",   key: "in",     label: "保留（in）",
            description: "按节点名关键字保留。每行一个关键字表示\"或\"；同行用 . 分隔表示\"与\"。例：另起一行填 香港、台湾 表示含其一即可；同一行填 香港.IPLC 表示同时含香港和 IPLC。",
            placeholder: "如：香港 / 香港.IPLC" },
          { type: "tags",   key: "out",    label: "删除（out）",
            description: "按节点名关键字删除。每行一个关键字表示\"或\"；同行用 . 分隔表示\"与\"。",
            placeholder: "如：BGP / BGP.试用" },
          { type: "text",   key: "regex",  label: "正则保留（regex）",
            description: "对节点完整信息正则匹配", placeholder: "iplc" },
          { type: "text",   key: "regout", label: "正则删除（regout）",
            description: "对节点完整信息正则删除", placeholder: "" }
        ]
      },
      {
        type: "group",
        title: "节点参数",
        items: [
          { type: "select", key: "emoji", label: "Emoji 旗帜",
            description: "添加/删除节点名地区旗帜", options: _emojiOptions() },
          { type: "switch", key: "udp",   label: "UDP Relay",
            onValue: "1", offValue: "-1" },
          { type: "switch", key: "tfo",   label: "Fast Open",
            onValue: "1", offValue: "-1" },
          { type: "switch", key: "cert",  label: "TLS 证书验证",
            description: "默认关闭", onValue: "1", offValue: "-1" },
          { type: "switch", key: "uot",   label: "UDP over TCP（仅 SS/SSR）",
            onValue: "1", offValue: "" },
          { type: "switch", key: "aead",  label: "VMess AEAD",
            description: "关闭 VMess AEAD", onValue: "", offValue: "-1" },
          { type: "text",   key: "alpn",  label: "ALPN",
            description: "over-tls 节点的 ALPN", placeholder: "h2" },
          { type: "text",   key: "host",  label: "Host",
            description: "修改已有 host；增加 host 请加 ☠️ 结尾",
            placeholder: "" },
          { type: "text",   key: "checkurl", label: "Check URL",
            description: "server_check_url 参数",
            placeholder: "http://...", keyboard: "url" }
        ]
      },
      {
        type: "group",
        title: "节点重命名",
        items: [
          { type: "text", key: "rename",  label: "Rename",
            description: "格式：旧名@新名 / 前缀@ / @后缀；多组用 + 连接；删除字段用 ☠️ 结尾",
            placeholder: "香港@HK+@[1X]" },
          { type: "text", key: "rrname",  label: "Reverse Rename",
            description: "在 emoji 之后再次重命名", placeholder: "" },
          { type: "text", key: "replace", label: "正则替换",
            description: "regex1@str1+regex2@str2", placeholder: "" },
          { type: "select", key: "ptn", label: "字母样式（ptn）",
            description: "将节点名英文替换成花式样式",
            options: _ptnOptions() },
          { type: "select", key: "npt", label: "数字样式（npt）",
            description: "将节点名数字替换成花式样式",
            options: _nptOptions() }
        ]
      },
      {
        type: "group",
        title: "Rewrite / Filter",
        description: "仅对 rewrite_remote / filter_remote 生效",
        items: [
          { type: "tags", key: "inhn",  label: "保留主机名（inhn）",
            description: "按主机名关键字保留 rewrite/filter。每行一个关键字（多行之间为\"或\"关系）。",
            placeholder: "如：weibo" },
          { type: "tags", key: "outhn", label: "删除主机名（outhn）",
            description: "按主机名关键字删除 rewrite/filter。每行一个关键字（多行之间为\"或\"关系）。",
            placeholder: "如：tb_price" },
          { type: "text", key: "policy", label: "默认策略组",
            description: "rule-set 生成策略组的名字（默认 Shawn）",
            placeholder: "Shawn" },
          { type: "text", key: "pset",   label: "Policy Set",
            description: "regex1@policy1+regex2@policy2" },
          { type: "select", key: "dst", label: "转换目标（dst）",
            options: [
              { label: "默认（不转换）", value: ""        },
              { label: "Rewrite",        value: "rewrite" },
              { label: "Filter",         value: "filter"  }
            ] },
          { type: "switch", key: "cdn",  label: "GitHub CDN 加速",
            onValue: "1", offValue: "" },
          { type: "select", key: "fcr",  label: "网络接口（fcr）",
            options: [
              { label: "默认",            value: ""  },
              { label: "强制蜂窝数据",    value: "1" },
              { label: "混合接口",        value: "2" },
              { label: "负载均衡",        value: "3" }
            ] },
          { type: "text", key: "via",    label: "via-interface",
            description: "0 = via-interface=%TUN%" }
        ]
      },
      {
        type: "group",
        title: "其他",
        items: [
          { type: "select", key: "ntf",  label: "解析通知",
            options: [
              { label: "默认",          value: "" },
              { label: "关闭",          value: "0"},
              { label: "打开",          value: "1"}
            ] },
          { type: "select", key: "type", label: "强制类型",
            options: [
              { label: "自动",         value: ""           },
              { label: "Nodes",        value: "nodes"      },
              { label: "Rule",         value: "rule"       },
              { label: "Module",       value: "module"     },
              { label: "List",         value: "list"       },
              { label: "Domain Set",   value: "domain-set" }
            ] },
          { type: "switch", key: "info", label: "流量信息",
            onValue: "1", offValue: "" },
          { type: "text",   key: "flow", label: "流量参数",
            description: "格式：到期时间:总流量GB:已用GB（如 2026-12-31:1000:54）",
            placeholder: "2026-12-31:1000:54" },
          { type: "text",   key: "relay", label: "代理链 relay",
            description: "目标策略名，将节点订阅转换为 ip/host 规则" }
        ]
      }
    ]
  };
};

// hashToUI：解析 hash，返回当前已存在参数（带 value）
$parser.hashToUI = function (hash) {
  if (!hash) return { version: 1, sections: [], unknown: "" };

  var schema = $parser.hashSchema();
  var allItems = {};
  schema.sections.forEach(function (g) {
    g.items.forEach(function (it) { allItems[it.key] = it; });
  });

  // 解析 hash
  var pairs = hash.split("&");
  var values = {};
  var unknown = [];
  pairs.forEach(function (p) {
    if (!p) return;
    var eq = p.indexOf("=");
    var k = eq === -1 ? p : p.substring(0, eq);
    var v = eq === -1 ? "" : p.substring(eq + 1);
    if (allItems[k]) {
      values[k] = v;
    } else {
      unknown.push(p);
    }
  });

  // 按 schema 结构组装回 sections，仅保留出现的
  var sections = [];
  schema.sections.forEach(function (g) {
    var items = [];
    g.items.forEach(function (it) {
      if (!(it.key in values)) return;
      var clone = JSON.parse(JSON.stringify(it));
      var raw = values[it.key];
      if (it.type === "switch") {
        clone.value = (raw === it.onValue);
      } else if (it.type === "tags") {
        clone.value = raw.split("+").filter(Boolean);
      } else {
        clone.value = raw;
      }
      items.push(clone);
    });
    if (items.length) {
      sections.push({
        type: "group",
        title: g.title,
        description: g.description,
        items: items
      });
    }
  });

  return {
    version: 1,
    sections: sections,
    unknown: unknown.join("&")
  };
};

// uiToHash：把 values 序列化成 hash 字符串
$parser.uiToHash = function (values) {
  var schema = $parser.hashSchema();
  var allItems = {};
  schema.sections.forEach(function (g) {
    g.items.forEach(function (it) { allItems[it.key] = it; });
  });

  var parts = [];
  Object.keys(values).forEach(function (k) {
    if (k === "__unknown") return;
    var meta = allItems[k];
    if (!meta) return;
    var v = values[k];
    var serialized;
    if (meta.type === "switch") {
      serialized = v ? meta.onValue : meta.offValue;
    } else if (meta.type === "tags") {
      serialized = (v || []).join("+");
    } else {
      serialized = String(v);
    }
    parts.push(k + "=" + serialized);
  });

  if (values.__unknown) parts.push(values.__unknown);
  return parts.join("&");
};
```


## 9. 协议演进

- 当前版本 `version: 1`
- 后续若新增控件类型 / 字段，**必须** bump `version`
- native 收到不认识的 `version` 时按 v1 解释（向前兼容尝试）；遇到完全无法解释的字段忽略
- 旧脚本（无 `$parser`）继续用纯文本编辑，不受影响


---

**草案版本**：v1，2026-04-28

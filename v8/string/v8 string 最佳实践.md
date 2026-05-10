下面从 **V8 String 内部表示 → 性能/内存风险 → JS 写法最佳实践 → why** 这条链路来总结。

先给结论：

> **V8 String 优化的核心不是“少用字符串”，而是：少制造大字符串、少制造临时字符串、避免长期持有大字符串的切片、避免无意义的编码升级、避免在热路径反复做字符串变换。**

---

# 一、先建立 V8 String 的性能模型

在 ECMAScript 语义里，JS 字符串本质是 **16-bit code unit 序列**，通常按 UTF-16 code unit 理解；`length` 统计的也是 code unit 数量，而不是用户肉眼看到的“字符数”。([TC39][1])

但 V8 内部不会只用一种方式存字符串。它会根据字符串来源和操作方式，使用多种表示：

```txt
SeqOneByteString     连续 one-byte 字符串
SeqTwoByteString     连续 two-byte 字符串
ConsString           拼接字符串，类似 rope
SlicedString         切片字符串，引用父字符串 + offset + length
ExternalString       外部字符串，内容不一定直接在 V8 heap 内
ThinString           指向规范化/internalized 字符串的转发对象
InternalizedString   内部化字符串，常用于属性名/字面量等
```

V8 源码里明确区分了 `ONE_BYTE_ENCODING` 和 `TWO_BYTE_ENCODING`，并且 flat string 的内容可以是 one-byte chars 或 two-byte UC16；源码中也能看到 `StringShape` 会判断字符串是否是 sequential、external、cons、sliced、thin、one-byte、two-byte 等形态。([GitHub][2])

所以，写 JS 时要关注的不只是：

```js
const s = 'hello';
```

而是要关心这个字符串在 V8 里可能是：

```txt
连续的？
拼接出来的？
切片出来的？
one-byte 的？
two-byte 的？
是否引用了一个更大的父字符串？
是否会在之后某个操作中被 flatten？
```

---

# 二、最佳实践 1：不要以为所有字符串都是 UTF-16 两字节存储

## 推荐认知

JS 语言语义上按 UTF-16 code unit 建模，但 V8 内部可以把内容都落在 Latin-1 / one-byte 范围内的字符串用 one-byte 存储。V8 源码中 `SeqOneByteString` 的字符类型是 `uint8_t`，`SeqTwoByteString` 的字符类型是 `uint16_t`。([GitHub][2])

所以：

```js
const a = 'hello world';    // 大概率 one-byte
const b = 'abc123';         // 大概率 one-byte
const c = '尝试移除';       // 需要 two-byte
```

## why

one-byte 的内存主体大致是：

```txt
header + length * 1 byte
```

two-byte 的内存主体大致是：

```txt
header + length * 2 bytes
```

对于长字符串，差异非常明显。

例如：

```txt
100 万字符 ASCII 字符串：约 1 MB 主体数据
100 万字符中文字符串：约 2 MB 主体数据
```

还没算对象头、对齐、引用、GC 元数据等额外开销。

---

## 重要误区：`\uXXXX` 不一定能把中文运行时变成 one-byte

你之前问过这种写法：

```js
const s = '\u5C1D\u8BD5\u79FB\u9664';
```

它在源码里看起来是 ASCII 字符，但 **JS 解析后运行时字符串值仍然是中文 code unit**。

也就是说：

```js
'\u5C1D' === '尝'; // true
```

运行时字符的 code unit 是 `0x5C1D`，超过 `0xFF`，所以对于最终字符串内容，V8 不能把它当成 one-byte 顺序字符串存储。

只有下面这种才是 one-byte 字符串：

```js
const s = '\\u5C1D\\u8BD5';
```

但它的语义已经不是中文，而是六个 ASCII 字符：

```txt
\ u 5 C 1 D
```

所以结论是：

> **Unicode escape 可以让源码文件保持 ASCII，但不能让“中文语义的运行时字符串”变成 one-byte。**

---

# 三、最佳实践 2：固定字典、协议字段、枚举值尽量用 ASCII

## 推荐写法

```js
const CellType = {
  TEXT: 'text',
  NUMBER: 'number',
  FORMULA: 'formula',
  DATE: 'date',
};
```

而不是：

```js
const CellType = {
  TEXT: '文本',
  NUMBER: '数字',
  FORMULA: '公式',
  DATE: '日期',
};
```

业务展示可以中文化：

```js
const CellTypeLabel = {
  text: '文本',
  number: '数字',
  formula: '公式',
  date: '日期',
};
```

## why

协议字段、状态码、枚举 key 通常会大量重复出现：

```js
{
  type: 'formula',
  status: 'pending',
  source: 'remote',
}
```

如果这些核心字段都用 ASCII，V8 更容易以 one-byte 形式保存，序列化、比较、哈希、属性访问、日志处理也更轻。

但展示文案本来就是中文，这部分不应该为了 one-byte 强行牺牲可读性。正确分层是：

```txt
内部协议 / 状态 / key：ASCII
用户展示文案：中文
```

---

# 四、最佳实践 3：不要在热路径里反复拼接大字符串

## 不推荐

```js
let html = '';

for (const row of rows) {
  html += `<tr><td>${row.name}</td><td>${row.value}</td></tr>`;
}
```

## 更稳妥的写法

```js
const chunks = [];

for (const row of rows) {
  chunks.push(`<tr><td>${row.name}</td><td>${row.value}</td></tr>`);
}

const html = chunks.join('');
```

## why

V8 对字符串拼接做了优化。很多时候，`a + b` 不会立刻复制所有字符，而是构造 `ConsString`，类似 rope：

```txt
ConsString
  first  -> 'hello'
  second -> 'world'
```

这样短期看很快，因为避免了每次拼接都复制完整字符串。

但问题是：当后续操作需要连续内存时，V8 可能要把 `ConsString` flatten 成顺序字符串。V8 官方在 JSON.stringify 优化文章里也提到，一些内部字符串表示，例如 `ConsString`，在序列化前可能需要 flatten，并且 flatten 可能触发内存分配；JSON.stringify 快路径更适合简单、顺序的字符串。([V8][3])

也就是说：

```txt
拼接阶段省了复制
最终消费阶段可能集中还债
```

风险包括：

```txt
1. flatten 时一次性分配大块连续内存
2. 触发 GC
3. 大字符串进入 Old Space
4. 造成瞬时内存峰值
5. 非 flat string 在某些转换场景下性能变差
```

V8 源码中 `ToCString` 的注释也提到，字符串应该 nearly flat，否则性能可能非常慢，甚至可能出现与长度相关的二次复杂度风险。([GitHub][2])

---

## 但不要绝对化：不是所有 `+=` 都慢

下面这种小规模拼接没必要过度优化：

```js
const msg = 'cell ' + row + ':' + col + ' updated';
```

真正需要注意的是：

```txt
大循环
大文本
日志聚合
HTML/SQL/CSV 生成
JSON 拼接
公式字符串批量生成
```

判断标准：

```txt
拼接次数是否很多？
最终字符串是否很大？
是否处在用户交互热路径？
是否会马上 JSON.stringify / postMessage / encode / 写文件？
```

---

# 五、最佳实践 4：大文本处理优先“分块/流式”，不要一次性变成巨型字符串

## 不推荐

```js
const text = await response.text();
const lines = text.split('\n');

for (const line of lines) {
  process(line);
}
```

## 更推荐

```js
const reader = response.body
  .pipeThrough(new TextDecoderStream())
  .getReader();

let buffer = '';

while (true) {
  const { value, done } = await reader.read();
  if (done) break;

  buffer += value;

  let index;
  while ((index = buffer.indexOf('\n')) !== -1) {
    const line = buffer.slice(0, index);
    process(line);
    buffer = buffer.slice(index + 1);
  }
}

if (buffer) {
  process(buffer);
}
```

## why

一次性 `response.text()` 会产生一个完整大字符串。

然后：

```js
text.split('\n')
```

又会产生大量子字符串和数组元素。

内存峰值可能变成：

```txt
原始大字符串
+ split 结果数组
+ 每一行子字符串
+ 业务处理产生的新字符串
```

如果是几十 MB、几百 MB 的日志、CSV、HTML、表格数据，内存会很危险。

对于大文件、大日志、大 CSV、大 JSONL，最佳策略通常是：

```txt
分块读取
增量解码
逐行消费
及时释放
```

也就是不要让 V8 heap 同时持有所有文本数据。

---

# 六、最佳实践 5：小心 `slice / substring` 长期持有大字符串

## 风险写法

```js
function parseLargeText(text) {
  const results = [];

  for (const match of text.matchAll(/id=(\w+)/g)) {
    results.push(match[1]);
  }

  return results;
}
```

或者：

```js
const huge = loadHugeString();

const token = huge.slice(100, 120);

cache.set('token', token);

// huge 变量看似不用了，但 token 可能仍然间接保留 huge
```

## why

V8 可能用 `SlicedString` 表示子串：

```txt
SlicedString
  parent -> huge string
  offset -> 100
  length -> 20
```

这样创建子串时不用复制数据，很快、也省短期内存。

但如果你只保留一个很小的子串，却让它引用着一个巨大的父字符串，就会出现：

```txt
20 字符的小 token
保活 100 MB 的原始 text
```

V8 当前源码里 `StringShape` 明确存在 `IsSliced()` 这类形态判断；V8 字符串接口也有 `GetUnderlying()`，用于返回 sliced string 的 parent 或 flat cons string 的 first part。([GitHub][2])

---

## 推荐策略 1：长期保存时，避免保存子串对象，保存位置索引

```js
const tokens = [];

for (const match of text.matchAll(/id=(\w+)/g)) {
  tokens.push({
    start: match.index + 3,
    end: match.index + 3 + match[1].length,
  });
}
```

需要时再读取：

```js
function getToken(text, token) {
  return text.slice(token.start, token.end);
}
```

适合：

```txt
原始大文本生命周期明确
结果不需要独立长期保存
解析器、编辑器、代码分析器、日志查看器
```

---

## 推荐策略 2：如果必须长期保存小片段，主动复制成独立字符串

JS 标准没有提供一个明确的“detach substring from parent”的 API。很多民间技巧，比如：

```js
(' ' + s).slice(1)
```

在不同 V8 版本里不一定可靠。

在 Node.js 里，如果确实需要把小字符串从巨大父字符串中脱离，可以考虑：

```js
const detached = Buffer.from(slice).toString();
```

在浏览器里可以用更重的方式：

```js
const detached = new TextDecoder().decode(new TextEncoder().encode(slice));
```

但这会产生编码/解码成本，不适合热路径滥用。

更合理的判断是：

```txt
如果 slice 很小，但 parent 巨大，而且 slice 要长期进入缓存/状态树/索引：
  可以复制一次，换取释放 parent 的机会。

如果 slice 很快消费完：
  不要复制，让 V8 用 SlicedString 反而更省。
```

---

# 七、最佳实践 6：正则匹配大字符串时，不要长期保存 match 对象

## 不推荐

```js
const matches = [];

let m;
const re = /name=(\w+)/g;

while ((m = re.exec(hugeText)) !== null) {
  matches.push(m);
}
```

## why

`match` / `exec` 的结果数组不只是捕获组，它还可能带有 `index`、`input` 等属性，其中 `input` 是原始被解析字符串。MDN 对 `match()` 的说明里也明确提到，返回结果的 `input` 属性是原始字符串。([MDN Web Docs][4])

这意味着你保存的不是单纯的匹配片段，而可能是：

```txt
match array
  input -> hugeText
```

结果是：

```txt
保存了很多小 match
但整个 hugeText 都释放不了
```

---

## 推荐写法

只保存必要字段：

```js
const results = [];

let m;
const re = /name=(\w+)/g;

while ((m = re.exec(hugeText)) !== null) {
  results.push({
    name: m[1],
    index: m.index,
  });
}
```

更进一步，如果担心 `m[1]` 是大字符串切片，也可以保存索引：

```js
const results = [];

let m;
const re = /name=(\w+)/g;

while ((m = re.exec(hugeText)) !== null) {
  const start = m.index + 'name='.length;
  const end = start + m[1].length;

  results.push({ start, end });
}
```

---

# 八、最佳实践 7：不要在热路径频繁创建复合字符串 key

## 常见写法

```js
const key = `${row}:${col}`;
cellMap.set(key, cell);
```

这在普通业务里完全可以。

但在表格、画布、编辑器、依赖图这种高频场景里，可能有问题：

```js
for (const cell of dirtyCells) {
  const key = `${cell.row}:${cell.col}`;
  const node = graphMap.get(key);
}
```

## why

每次模板字符串都会创建新的字符串：

```txt
row number -> string
':' string
col number -> string
拼接结果 ConsString / SeqString
后续 Map hash
```

如果这个逻辑在几十万次循环里执行，会制造大量临时字符串，增加 New Space 分配压力和 Minor GC 频率。

---

## 推荐方案 1：用数字编码 key

```js
const MAX_COL = 20000;

function cellKey(row, col) {
  return row * MAX_COL + col;
}

const key = cellKey(row, col);
cellMap.set(key, cell);
```

适合：

```txt
row / col 范围可控
key 不超过 Number 安全整数
性能敏感
```

注意要保证：

```js
row * MAX_COL + col <= Number.MAX_SAFE_INTEGER;
```

---

## 推荐方案 2：嵌套 Map，避免拼接字符串

```js
const rows = new Map();

function setCell(row, col, cell) {
  let cols = rows.get(row);
  if (!cols) {
    cols = new Map();
    rows.set(row, cols);
  }
  cols.set(col, cell);
}

function getCell(row, col) {
  return rows.get(row)?.get(col);
}
```

适合：

```txt
稀疏二维表
row / col 是数字
避免复合字符串 key
```

---

## 推荐方案 3：大规模数值索引用 TypedArray / 专用索引结构

例如：

```js
const rows = new Int32Array(n);
const cols = new Int32Array(n);
const deps = new Int32Array(n);
```

或者用：

```txt
R-Tree
Interval Tree
Sparse Matrix
Bitmap
Roaring Bitmap
```

对于表格依赖图、条件格式区间、数据验证区间，这通常比字符串 key 更适合。

---

# 九、最佳实践 8：避免在热路径反复 `toLowerCase / trim / normalize`

## 不推荐

```js
for (const item of list) {
  if (item.name.toLowerCase().trim() === keyword.toLowerCase().trim()) {
    return item;
  }
}
```

## 推荐

```js
const normalizedKeyword = keyword.toLowerCase().trim();

for (const item of list) {
  if (item.normalizedName === normalizedKeyword) {
    return item;
  }
}
```

数据进入系统时预处理：

```js
function normalizeItem(item) {
  return {
    ...item,
    normalizedName: item.name.toLowerCase().trim(),
  };
}
```

## why

这些 API 通常会创建新字符串：

```txt
toLowerCase -> 新字符串
trim        -> 新字符串或切片表示
normalize   -> 新字符串
replace     -> 新字符串
```

如果在热路径反复执行，就会制造大量短命字符串。

在搜索、过滤、排序、公式解析、字段匹配场景里，推荐把字符串规范化从：

```txt
每次比较时做
```

变成：

```txt
数据进入系统时做一次
```

也就是：

```txt
write-time normalize
read-time compare
```

---

# 十、最佳实践 9：对用户可见文本，要正确处理 code unit / code point / grapheme

## 风险写法

```js
const chars = str.split('');
```

对 emoji 会出问题：

```js
'😄'.split('');
// 可能得到两个 surrogate code units
```

MDN 明确指出，`split("")` 会按 UTF-16 code unit 切分，可能拆开 surrogate pair；字符串索引也是基于 UTF-16 code unit，而 `[Symbol.iterator]` 是按 Unicode code point 迭代。([MDN Web Docs][5])

---

## 推荐策略

如果你处理的是内部协议、ASCII token、公式标识符：

```js
for (let i = 0; i < str.length; i++) {
  const code = str.charCodeAt(i);
}
```

why：

```txt
charCodeAt 按 code unit 读取，速度直接，适合词法分析、ASCII 协议、公式 token
```

如果你处理 Unicode code point：

```js
for (const ch of str) {
  // ch 是 code point 级别，不会拆 surrogate pair
}
```

如果你处理用户肉眼看到的字符，比如 emoji、组合音标、复杂文字：

```js
const segmenter = new Intl.Segmenter('zh', {
  granularity: 'grapheme',
});

for (const { segment } of segmenter.segment(str)) {
  console.log(segment);
}
```

why：

```txt
用户看到的“一个字符”不一定等于一个 code unit，也不一定等于一个 code point。
```

所以：

| 场景            | 推荐粒度                             |
| ------------- | -------------------------------- |
| 公式 lexer      | code unit / charCodeAt           |
| ASCII 协议解析    | code unit                        |
| Unicode 字符遍历  | code point / for...of            |
| 光标移动、删除一个“字符” | grapheme cluster                 |
| 用户可见长度统计      | grapheme cluster                 |
| 内存估算          | code unit / V8 one-byte/two-byte |

---

# 十一、最佳实践 10：大规模文本扫描优先用 index 游标，不要疯狂切字符串

## 不推荐

```js
while (source.length > 0) {
  const token = source.slice(0, 10);
  process(token);
  source = source.slice(10);
}
```

## 推荐

```js
let pos = 0;

while (pos < source.length) {
  const end = Math.min(pos + 10, source.length);
  processRange(source, pos, end);
  pos = end;
}
```

必要时再切：

```js
function processRange(source, start, end) {
  const token = source.slice(start, end);
  // 只在真正需要字符串对象时创建
}
```

## why

反复 `slice` 会产生大量子字符串对象。即使 V8 用 SlicedString 避免复制字符数据，也仍然会产生很多小对象：

```txt
SlicedString object
parent pointer
offset
length
```

而且如果这些切片被保留，会带来父字符串保活问题。

解析器、词法分析器、CSV 解析、公式解析更推荐：

```txt
source + index + length
```

而不是：

```txt
不断创建 token string
```

例如公式 lexer 可以这样设计：

```js
function nextToken(source, pos) {
  const start = pos;

  while (pos < source.length) {
    const c = source.charCodeAt(pos);
    if (c === 43 /* + */ || c === 45 /* - */) break;
    pos++;
  }

  return {
    type: 'identifier',
    start,
    end: pos,
  };
}
```

最后只在展示或错误提示时 materialize：

```js
const text = source.slice(token.start, token.end);
```

---

# 十二、最佳实践 11：避免把大字符串塞进状态树、响应式系统、日志系统

## 不推荐

```js
store.setState({
  workbook,
  rawCsvText,
  parsedRows,
});
```

或者：

```js
reactiveState.rawHtml = hugeHtml;
```

## why

现代前端状态系统可能会做：

```txt
代理 Proxy
快照 snapshot
diff
devtools 序列化
time-travel
日志打印
持久化缓存
```

如果里面有巨大字符串，会导致：

```txt
1. 状态更新成本变高
2. devtools 卡顿
3. JSON.stringify 巨慢
4. 内存无法释放
5. 历史快照保留多份大字符串
```

V8 的 JSON.stringify 快路径也强调，它更适合满足特定条件的对象和简单字符串；复杂字符串表示可能需要 flatten 并引发分配。([V8][3])

---

## 推荐

大字符串放在独立模块，并且明确生命周期：

```js
class TextBuffer {
  constructor(text) {
    this.text = text;
  }

  dispose() {
    this.text = '';
  }
}

const buffer = new TextBuffer(rawCsvText);
```

状态树只保存引用 ID：

```js
store.setState({
  activeBufferId: buffer.id,
  rowCount,
  colCount,
});
```

更高级一点：

```txt
大文本：TextBuffer / Piece Table / Rope / Chunk Store
状态树：只存元信息和 ID
渲染层：按需读取局部文本
```

这对在线文档、表格、代码编辑器尤其重要。

---

# 十三、最佳实践 12：日志、埋点、错误信息不要无脑拼接大对象

## 不推荐

```js
logger.info('cell updated: ' + JSON.stringify(cell));
```

或者：

```js
throw new Error(`Failed to parse: ${hugeText}`);
```

## 推荐

```js
logger.info('cell updated', {
  row: cell.row,
  col: cell.col,
  valueType: cell.valueType,
});
```

错误信息限制长度：

```js
function preview(str, limit = 200) {
  if (str.length <= limit) return str;
  return `${str.slice(0, limit)}...(${str.length} chars)`;
}

throw new Error(`Failed to parse: ${preview(hugeText)}`);
```

## why

日志字符串常常有三个隐藏问题：

```txt
1. 模板字符串会提前构造完整字符串
2. JSON.stringify 会深度遍历对象并创建巨大字符串
3. 错误对象可能被长期保存，导致大字符串被保活
```

更好的方式是结构化日志：

```txt
记录关键字段
延迟序列化
限制长度
不要把完整大文本塞进 Error.message
```

---

# 十四、最佳实践 13：大量重复字符串要做规范化/字典化

## 场景

表格里可能有大量重复值：

```txt
"未开始"
"进行中"
"已完成"
"张三"
"广州"
"产品部"
```

如果每个 cell 都保存一份字符串引用，短字符串本身可能被 internalized 或共享，但不要完全依赖引擎行为。

对于业务上明显可枚举、可复用的字段，建议做字典化。

---

## 推荐设计

```js
class StringPool {
  constructor() {
    this.idByString = new Map();
    this.stringById = [];
  }

  intern(str) {
    let id = this.idByString.get(str);

    if (id !== undefined) {
      return id;
    }

    id = this.stringById.length;
    this.idByString.set(str, id);
    this.stringById.push(str);
    return id;
  }

  get(id) {
    return this.stringById[id];
  }
}
```

业务存储：

```js
const pool = new StringPool();

const cell = {
  valueId: pool.intern('进行中'),
};
```

读取：

```js
const value = pool.get(cell.valueId);
```

## why

字典化可以把：

```txt
N 个重复字符串
```

变成：

```txt
1 份字符串 + N 个整数 ID
```

对于大型表格、枚举字段、分类字段、状态字段、标签字段，非常有价值。

不过不要滥用。对完全随机、重复率很低的字符串，比如 UUID、随机日志、hash，字典化反而会增加 Map 成本。

适用判断：

```txt
重复率高？
字符串较长？
数量巨大？
生命周期长？
```

满足越多，越适合池化。

---

# 十五、最佳实践 14：字符串比较前，尽量降低比较成本

## 不推荐

```js
items.sort((a, b) => {
  return a.name.localeCompare(b.name);
});
```

如果 `items` 很大，且排序频繁，`localeCompare` 成本可能很高。

## 推荐

简单 ASCII / 协议字段排序：

```js
items.sort((a, b) => {
  return a.key < b.key ? -1 : a.key > b.key ? 1 : 0;
});
```

需要 locale 排序时，复用 `Intl.Collator`：

```js
const collator = new Intl.Collator('zh-Hans-CN', {
  sensitivity: 'base',
  numeric: true,
});

items.sort((a, b) => collator.compare(a.name, b.name));
```

## why

字符串比较可能涉及：

```txt
长度判断
逐 code unit 比较
Unicode 规则
locale 规则
大小写/重音/数字排序
```

如果是内部 key，不要用 locale 级别比较。

如果是用户可见文本排序，要用正确语义，但要复用 collator，避免重复构造。

---

# 十六、最佳实践 15：避免 `new String()` 包装对象

## 不推荐

```js
const s = new String('hello');
```

## 推荐

```js
const s = 'hello';
```

## why

`new String('hello')` 创建的是对象：

```js
typeof 'hello';             // 'string'
typeof new String('hello'); // 'object'
```

它会带来：

```txt
额外对象分配
属性访问复杂化
比较语义容易出错
```

例如：

```js
new String('a') === new String('a'); // false
new String('a') === 'a';             // false
```

在性能和语义上都不推荐。

---

# 十七、最佳实践 16：避免无意义的字符串化/反字符串化

## 不推荐

```js
const cloned = JSON.parse(JSON.stringify(obj));
```

或者：

```js
const key = JSON.stringify(params);
cache.get(key);
```

## why

`JSON.stringify` 会生成完整字符串，`JSON.parse` 又会重新解析创建对象。

如果对象很大，会产生：

```txt
大字符串
临时对象
GC 压力
主线程阻塞
```

对于 clone：

```js
const cloned = structuredClone(obj);
```

对于 cache key：

```js
const key = `${type}:${id}`;
```

如果参数复杂，考虑稳定 hash：

```js
const key = hashParams(params);
```

或者使用嵌套 Map：

```js
let byType = cache.get(type);
if (!byType) {
  byType = new Map();
  cache.set(type, byType);
}
byType.set(id, value);
```

避免为了 key 生成巨大字符串。

---

# 十八、最佳实践 17：字符串缓存必须有上限

## 不推荐

```js
const cache = new Map();

function normalize(str) {
  if (cache.has(str)) return cache.get(str);

  const result = str.trim().toLowerCase();
  cache.set(str, result);
  return result;
}
```

如果输入字符串无限多，这就是内存泄漏。

## 推荐

```js
class LimitedStringCache {
  constructor(limit = 10000) {
    this.limit = limit;
    this.map = new Map();
  }

  get(key, compute) {
    if (this.map.has(key)) {
      const value = this.map.get(key);
      this.map.delete(key);
      this.map.set(key, value);
      return value;
    }

    const value = compute(key);
    this.map.set(key, value);

    if (this.map.size > this.limit) {
      const oldestKey = this.map.keys().next().value;
      this.map.delete(oldestKey);
    }

    return value;
  }
}
```

## why

字符串缓存常用于：

```txt
normalize
format
parse
hash
tokenize
measureText
```

缓存可以提升性能，但无界缓存会长期保活大量字符串，使它们晋升到 Old Space。

缓存一定要有：

```txt
容量上限
失效策略
clear 时机
命中率统计
```

---

# 十九、最佳实践 18：对长文本编辑，不要用单个字符串反复改

## 不推荐

```js
text = text.slice(0, start) + inserted + text.slice(end);
```

如果文本很大，且编辑频繁，每次都可能创建新字符串/新 rope，并带来后续 flatten 成本。

## 推荐数据结构

对于代码编辑器、在线文档、公式编辑器、大文本编辑器：

```txt
Piece Table
Rope
Gap Buffer
Chunked Text Buffer
```

简单一点可以分块：

```js
class ChunkedText {
  constructor(chunks = []) {
    this.chunks = chunks;
  }

  toString() {
    return this.chunks.join('');
  }
}
```

## why

字符串是不可变的。

每次编辑本质是：

```txt
旧字符串
+ 新插入片段
+ 新结果字符串/rope
```

编辑次数一多，会产生大量中间表示。

专业文本编辑器不会把整个文档当成一个 JS string 反复替换，而会使用分块文本结构。

---

# 二十、最佳实践 19：Node.js 场景下，二进制数据不要先转字符串

## 不推荐

```js
const text = fs.readFileSync(path, 'utf8');
const hash = crypto.createHash('sha256').update(text).digest('hex');
```

如果只是做二进制处理、hash、转发：

```js
const buffer = fs.readFileSync(path);
const hash = crypto.createHash('sha256').update(buffer).digest('hex');
```

## why

把 Buffer 转成 string 会产生 JS 字符串对象。

如果数据本质不是文本，或者只是要传输、hash、压缩、加密，就没必要经过 V8 string：

```txt
Buffer / Uint8Array 更适合二进制
String 更适合文本语义
```

这能减少：

```txt
编码转换
字符串分配
GC 压力
two-byte 升级风险
```

---

# 二十一、最佳实践 20：WASM/Native 边界减少字符串来回拷贝

## 不推荐

```js
for (const item of list) {
  wasm.processString(item.text);
}
```

如果每次都 JS string → UTF-8/UTF-16 → WASM memory，会有大量编码和拷贝成本。

## 推荐

批量传输：

```js
const encoded = new TextEncoder().encode(bigText);
wasm.processBuffer(ptr, encoded.length);
```

或者做字符串表：

```txt
JS 侧 StringPool
WASM 侧 ID / offset / length
一次性编码
多次复用
```

## why

JS string 和 WASM linear memory 不是同一个存储。

跨边界通常需要：

```txt
计算编码长度
分配 WASM 内存
编码复制
传 offset/length
WASM 读取
```

频繁小字符串跨边界，会让性能被 glue code 和编码成本吃掉。

表格公式计算、lexer、解析器、文本分析类 WASM 模块，建议：

```txt
批量编码
传 buffer
传 offset/length
避免逐字符串调用
```

---

# 二十二、结合 V8 String 类型的专项建议

## 1. 针对 SeqOneByteString / SeqTwoByteString

目标：

```txt
能 one-byte 就 one-byte
避免无意义 two-byte 升级
```

建议：

```txt
内部 key、状态码、协议字段用 ASCII
展示文案和用户输入保持真实语义
不要指望 \uXXXX 把中文运行时变成 one-byte
```

---

## 2. 针对 ConsString

目标：

```txt
利用拼接优化，但避免最终 flatten 暴雷
```

建议：

```txt
小字符串拼接可以直接 +
大循环生成大文本，用 chunks + join 或流式输出
拼接后马上要 JSON.stringify / encode / postMessage，要警惕 flatten 成本
```

---

## 3. 针对 SlicedString

目标：

```txt
短期切片省复制
长期保存避免保活大父串
```

建议：

```txt
解析大文本时优先保存 start/end
不要长期缓存从 hugeText slice 出来的小 token
必须长期保存小片段时，考虑复制成独立字符串，但要测量成本
```

---

## 4. 针对 ExternalString

目标：

```txt
不要把二进制/外部数据过早变成 JS string
```

建议：

```txt
Node 里 Buffer 优先处理二进制
浏览器里 ArrayBuffer / Uint8Array 优先处理二进制
只有真正需要文本语义时再 decode 成 string
```

V8 的 C++ API 也有 `NewExternalOneByte`、`NewExternalTwoByte`，说明 V8 可以让字符串关联外部资源；其生命周期由 V8 在字符串不再存活时处理外部资源。([v8.github.io][6])

---

# 二十三、表格/文档类业务里的最佳实践

结合你的表格背景，最重要的是这些。

## 1. 单元格坐标不要高频拼字符串 key

不推荐：

```js
const key = `${row}:${col}`;
```

推荐：

```js
const key = row * MAX_COL + col;
```

或者：

```js
Map<row, Map<col, Cell>>
```

why：

```txt
减少临时字符串
减少 hash 字符串成本
降低 GC 压力
```

---

## 2. 单元格文本值做冷热分离

```js
const cell = {
  row,
  col,
  valueId,
  type,
};
```

字符串池：

```js
const stringPool = new StringPool();
```

why：

```txt
重复值只存一份
cell 对象更小
依赖图/渲染层传整数更轻
```

---

## 3. 公式解析器不要每个 token 都切字符串

不推荐：

```js
tokens.push(source.slice(start, end));
```

推荐：

```js
tokens.push({
  type,
  start,
  end,
});
```

需要展示时再：

```js
source.slice(token.start, token.end);
```

why：

```txt
减少 token string 数量
避免 SlicedString 保活 source
降低 parser 热路径分配
```

---

## 4. 公式高亮可用 offset + length 描述引用

```js
{
  type: 'cell-ref',
  start: 3,
  end: 5,
  colorId: 2,
}
```

而不是：

```js
{
  type: 'cell-ref',
  text: 'A1',
  color: '#f00',
}
```

why：

```txt
高亮渲染阶段可以基于原始公式字符串和 token range 工作
不用为每个 token 复制字符串
```

---

## 5. 大表导入 CSV/TSV 要流式

不推荐：

```js
const rows = csv.split('\n').map(line => line.split(','));
```

推荐：

```txt
chunk decode
line parser
row by row append
必要时 worker 化
```

why：

```txt
避免原始 CSV、行数组、字段子串、解析对象同时常驻
```

---


# 最后关键总结

## 1. JS 语义是 UTF-16，但 V8 内部可以 one-byte

```txt
ASCII / Latin-1 内容：可能 one-byte
中文 / 大量非 Latin-1：通常 two-byte
```

所以内部协议字段、枚举、key 尽量用 ASCII。

---

## 2. `\uXXXX` 不能让中文运行时变 one-byte

```js
'\u5C1D' === '尝'
```

源码是 ASCII，不代表运行时字符串是 one-byte。

---

## 3. 大循环拼接字符串要小心 ConsString flatten

```txt
拼接时可能很快
最终消费时可能集中复制和分配
```

大文本生成优先：

```js
chunks.push(part);
chunks.join('');
```

或者直接流式输出。

---

## 4. 大字符串切片长期保存有风险

```txt
小 substring 可能引用巨大 parent string
```

解析器里优先存：

```js
{ start, end }
```

而不是直接存：

```js
source.slice(start, end)
```

---

## 5. 热路径不要频繁创建字符串 key

不推荐：

```js
`${row}:${col}`
```

高性能场景推荐：

```js
row * MAX_COL + col
```

或者：

```js
Map<row, Map<col, value>>
```

---

## 6. 字符串变换要前置和缓存

避免热路径反复：

```js
trim()
toLowerCase()
replace()
normalize()
```

推荐：

```txt
数据进入系统时 normalize
查询比较时直接使用 normalized 字段
```

---

## 7. 大文本不要一次性塞进 V8 heap

大文件、CSV、日志、HTML、JSONL：

```txt
分块读取
流式解码
逐行处理
及时释放
```

---

## 8. 字符串优化的本质

```txt
减少临时字符串
减少大字符串
减少重复字符串
减少 flatten 成本
减少 substring 保活父串
减少热路径编码/解码/规范化
```

最终原则是：

> **短文本让 V8 自己优化；大文本、热路径、高频解析、长期缓存场景，要主动设计字符串生命周期和数据结构。**



下面这份总结可以理解为：**既然 V8 内部通过 HiddenClass / Map、Fast Properties、Elements Kind、Inline Cache、GC 分代等机制来优化对象，那么我们写 JS 时应该尽量“配合 V8 的假设”，让对象形状稳定、访问路径稳定、内存布局紧凑、GC 压力可控。**

---

# 一、核心原则：让对象“长得一样、用得稳定、活得短”

从 V8 视角看，性能好的 JS 对象通常具备几个特征：

| 维度   | 好的写法                      | 差的写法                           |
| ---- | ------------------------- | ------------------------------ |
| 对象形状 | 同一类对象字段固定、顺序一致            | 动态增删字段、字段顺序混乱                  |
| 属性访问 | 访问路径稳定，函数入参对象结构一致         | 同一个函数接收多种形状对象                  |
| 存储模式 | 保持 Fast Properties        | 频繁 `delete` 导致 Dictionary Mode |
| 数组元素 | 保持 Packed / 同类型 Elements  | 稀疏数组、混合类型、洞数组                  |
| 生命周期 | 临时对象快进快出                  | 大量对象被意外长期引用                    |
| 数据结构 | 固定字段用 Object，动态 key 用 Map | 用普通对象模拟高频动态字典                  |

一句话总结：

> **固定结构的数据，用稳定 shape 的 Object；动态 key/high churn 数据，用 Map；数组保持连续、同类型；避免频繁 delete 和动态扩展对象。**

---

# 二、最佳实践 1：初始化时一次性声明完整字段

## 1.1 推荐写法：对象字段完整、顺序稳定

```js
class Cell {
  constructor(row, col, value) {
    this.row = row;
    this.col = col;
    this.value = value;
    this.formula = null;
    this.style = null;
  }
}
```

或者：

```js
function createCell(row, col, value) {
  return {
    row,
    col,
    value,
    formula: null,
    style: null,
  };
}
```

这样做的好处是：

1. 同一类对象会共享相同的 HiddenClass，也就是 V8 内部的 `Map`。
2. 属性偏移稳定，`cell.value` 可以被优化成接近固定偏移读取。
3. Inline Cache 更容易保持 monomorphic，也就是单态缓存。
4. 后续优化编译器更敢内联和消除冗余检查。

---

## 1.2 避免写法：后续不断补字段

```js
const cell = {};
cell.row = row;
cell.col = col;

if (hasFormula) {
  cell.formula = formula;
}

if (hasStyle) {
  cell.style = style;
}
```

问题在于，不同对象可能经历不同的属性添加路径：

```js
const a = {};
a.row = 1;
a.col = 2;
a.value = 3;

const b = {};
b.value = 3;
b.row = 1;
b.col = 2;
```

即使最终字段一样，**添加顺序不同，也可能产生不同 HiddenClass 转换链**。

这会导致：

```js
function getValue(cell) {
  return cell.value;
}
```

这个函数可能接收很多种不同 shape 的对象，Inline Cache 从：

```txt
monomorphic -> polymorphic -> megamorphic
```

一旦变成 megamorphic，属性访问优化效果会明显下降。

---

# 三、最佳实践 2：避免在热路径里动态增删属性

## 2.1 不推荐：频繁 `delete`

```js
delete cell.formula;
delete cell.style;
```

`delete` 对 V8 很不友好。

因为普通对象一开始通常是 Fast Properties：

```txt
Object
  map -> HiddenClass
  properties -> PropertyArray
```

但如果频繁删除属性，对象可能退化成 Dictionary Mode：

```txt
Object
  properties -> NameDictionary
```

Dictionary Mode 的特点是：

1. 属性查询变成哈希表查询。
2. 无法继续享受稳定 HiddenClass 带来的偏移访问。
3. Inline Cache 优化能力变弱。
4. 内存结构更重。

---

## 2.2 推荐：用 `null` 或 `undefined` 标记空值

```js
cell.formula = null;
cell.style = null;
```

这样对象 shape 不变，属性仍然存在，只是值为空。

在业务语义上，如果你需要区分：

```js
没有这个字段
```

和：

```js
字段存在，但没有值
```

可以明确使用：

```js
null      // 显式为空
undefined // 尚未赋值或未初始化
```

但从 V8 对象 shape 稳定性角度看，**保留字段通常比 delete 更好**。

---

# 四、最佳实践 3：动态 key 场景优先使用 Map

## 4.1 普通对象适合固定 schema

适合：

```js
const user = {
  id: 1,
  name: 'Tom',
  age: 18,
};
```

这种对象字段固定，访问方式稳定：

```js
user.name
user.age
```

---

## 4.2 Map 适合动态字典

如果你的数据是动态 key：

```js
const cache = {};

cache[id1] = value1;
cache[id2] = value2;
delete cache[id1];
```

这种场景更适合：

```js
const cache = new Map();

cache.set(id1, value1);
cache.set(id2, value2);
cache.delete(id1);
```

尤其适合这些场景：

| 场景        | 推荐                 |
| --------- | ------------------ |
| key 不固定   | `Map`              |
| 高频增删      | `Map`              |
| key 不是字符串 | `Map`              |
| 需要稳定遍历顺序  | `Map`              |
| 大量缓存表、索引表 | `Map`              |
| 固定业务实体对象  | `Object` / `class` |

例如表格业务里：

```js
// 固定结构：适合 Object
const cell = {
  row,
  col,
  value,
  formula,
};

// 动态索引：适合 Map
const cellMap = new Map();
cellMap.set(`${row}:${col}`, cell);
```

不要把一个普通对象当成万能哈希表使用。

---

# 五、最佳实践 4：让函数入参对象 shape 尽量一致

## 5.1 不推荐：同一个函数接收多种结构

```js
function renderCell(cell) {
  return cell.value;
}

renderCell({ value: 1 });
renderCell({ value: 1, formula: '=A1+B1' });
renderCell({ row: 1, col: 2, value: 1 });
renderCell({ text: 'hello', value: 1 });
```

从 JS 语义上看没问题，因为都有 `value`。

但从 V8 角度看，这些对象可能是不同 HiddenClass。于是：

```js
cell.value
```

这个访问点会变得 polymorphic，甚至 megamorphic。

---

## 5.2 推荐：对数据做规范化

```js
function normalizeCell(raw) {
  return {
    row: raw.row ?? -1,
    col: raw.col ?? -1,
    value: raw.value ?? null,
    formula: raw.formula ?? null,
    style: raw.style ?? null,
  };
}
```

然后热路径只处理规范化之后的数据：

```js
function renderCell(cell) {
  return cell.value;
}
```

这在大型前端系统里非常重要。

比如表格、甘特图、多维表格、在线文档编辑器这类场景，经常会有：

```txt
接口数据 -> 业务模型 -> 渲染模型 -> 视图模型
```

建议在进入核心计算链路前做一次 model normalization，避免核心函数被各种临时结构污染。

---

# 六、最佳实践 5：避免对象字段类型频繁变化

## 6.1 不推荐：同一个字段一会儿是数字，一会儿是对象

```js
const cell = {
  value: 123,
};

cell.value = 'hello';
cell.value = { richText: true };
cell.value = null;
```

JS 允许这样写，但对优化不友好。

V8 的优化编译器会基于历史类型反馈做假设。比如它观察到：

```js
cell.value
```

一直是 number，就可能生成针对 number 的优化路径。

如果后面突然变成 object/string，就可能触发 deopt。

---

## 6.2 推荐：字段语义清晰，类型相对稳定

例如表格单元格可以拆成：

```js
const cell = {
  rawValue: '',
  displayValue: '',
  valueType: 'string',
  richText: null,
  formula: null,
};
```

而不是：

```js
const cell = {
  value: 123,          // number
};

cell.value = 'hello';  // string
cell.value = { ... };  // object
```

如果字段确实是 union type，建议显式加 tag：

```js
const cell = {
  type: 'richText',
  textValue: null,
  numberValue: null,
  richTextValue: richText,
};
```

这不仅有利于 V8 优化，也有利于 TypeScript 类型建模、可维护性和调试。

---

# 七、最佳实践 6：数组保持连续、紧凑、同类型

V8 对数组有 Elements Kind 优化。

常见类型大致包括：

```txt
PACKED_SMI_ELEMENTS       // 连续小整数数组
PACKED_DOUBLE_ELEMENTS    // 连续浮点数数组
PACKED_ELEMENTS           // 连续对象/混合数组

HOLEY_SMI_ELEMENTS        // 有洞的小整数数组
HOLEY_DOUBLE_ELEMENTS     // 有洞的浮点数组
HOLEY_ELEMENTS            // 有洞的对象/混合数组

DICTIONARY_ELEMENTS       // 极度稀疏数组
```

一般来说：

```txt
PACKED > HOLEY > DICTIONARY
```

---

## 7.1 推荐：用 `push` 构建连续数组

```js
const list = [];

for (let i = 0; i < n; i++) {
  list.push(i);
}
```

这样更容易保持 packed array。

---

## 7.2 不推荐：制造洞数组

```js
const list = new Array(1000);

list[0] = 1;
list[999] = 2;
```

这会产生大量 holes。

也不推荐：

```js
delete list[10];
```

因为这会制造 hole。

推荐替代：

```js
list[10] = undefined;
```

注意，这会改变元素类型，但至少不会制造真正的 hole。

如果需要删除并保持数组紧凑，可以用：

```js
list.splice(index, 1);
```

或者用 swap-delete：

```js
list[index] = list[list.length - 1];
list.pop();
```

适合不关心顺序的高性能场景。

---

## 7.3 避免超大稀疏索引

```js
const arr = [];
arr[1000000000] = 1;
```

这种写法可能让数组退化为 dictionary elements。

如果 key 本质是稀疏坐标，例如表格单元格：

```js
row = 100000
col = 20000
```

不要直接用二维稀疏数组：

```js
table[row][col] = cell;
```

更适合：

```js
const cellMap = new Map();
cellMap.set(`${row}:${col}`, cell);
```

或者用空间索引结构，例如 R-Tree、区间树、稀疏矩阵结构。

---

# 八、最佳实践 7：避免数组元素类型频繁升级

## 8.1 不推荐：混合类型数组

```js
const arr = [1, 2, 3];

arr.push(4.5);      // SMI -> DOUBLE
arr.push('hello');  // DOUBLE -> OBJECT/PACKED_ELEMENTS
arr.push({});       // 继续泛化
```

V8 会根据数组内容选择 Elements Kind。

一旦从更具体的类型退化到更通用的类型，通常很难回到更优化的表示。

---

## 8.2 推荐：不同类型数据分开存

```js
const ids = [1, 2, 3, 4];
const names = ['a', 'b', 'c'];
const objects = [{ id: 1 }, { id: 2 }];
```

在性能敏感场景，可以考虑结构化拆分：

```js
const rows = [];
const cols = [];
const values = [];

rows.push(row);
cols.push(col);
values.push(value);
```

这种方式类似数据导向设计，适合大量同构数据。

例如表格大规模单元格计算、图形渲染、布局引擎、依赖图计算等场景。

---

# 九、最佳实践 8：热路径中少创建临时对象

## 9.1 不推荐：循环里大量创建短命对象

```js
for (let i = 0; i < list.length; i++) {
  const point = {
    x: list[i].x,
    y: list[i].y,
  };

  draw(point);
}
```

年轻代对象分配很快，但不是免费。

大量临时对象会带来：

1. New Space 分配压力。
2. Minor GC 更频繁。
3. 对象如果被闭包、数组、缓存引用住，可能晋升到 Old Space。
4. Old Space 压力增大后，Major GC 成本明显更高。

---

## 9.2 推荐：热路径中复用结构，或者传基础值

```js
for (let i = 0; i < list.length; i++) {
  draw(list[i].x, list[i].y);
}
```

或者：

```js
const point = { x: 0, y: 0 };

for (let i = 0; i < list.length; i++) {
  point.x = list[i].x;
  point.y = list[i].y;
  draw(point);
}
```

不过对象复用也要谨慎。

如果 `draw(point)` 内部异步保存了引用：

```js
function draw(point) {
  queue.push(() => {
    console.log(point.x, point.y);
  });
}
```

那复用对象会产生 bug。

所以原则是：

> **同步、局部、无逃逸的热路径可以复用对象；跨异步、跨模块、会被持久引用的对象不要复用。**

---

# 十、最佳实践 9：避免闭包意外持有大对象

## 10.1 不推荐：闭包引用整个大对象

```js
function createHandler(sheet) {
  return function onClick() {
    console.log(sheet.activeCell);
  };
}
```

如果 `sheet` 是一个很大的对象，闭包会让它无法被 GC 回收。

尤其在前端复杂应用里，常见问题是：

```txt
DOM 事件监听器 -> 闭包 -> 大业务对象 -> 大缓存/大数组
```

导致整个链路都无法释放。

---

## 10.2 推荐：只捕获必要字段

```js
function createHandler(sheetId, getActiveCell) {
  return function onClick() {
    console.log(getActiveCell(sheetId));
  };
}
```

或者在销毁时主动解除引用：

```js
element.removeEventListener('click', handler);
handler = null;
largeCache.clear();
```

对于表格这类重型应用，尤其需要注意：

```txt
Workbook
Sheet
CellModel
RenderCache
FormulaGraph
DependencyTree
SelectionModel
HistoryStack
```

这些对象一旦被闭包、事件、定时器、Promise、全局缓存引用，Old Space 会持续增长，最后容易 OOM。

---

# 十一、最佳实践 10：慎用深拷贝和大对象展开

## 11.1 不推荐：频繁展开大对象

```js
const newState = {
  ...oldState,
  activeCell: nextCell,
};
```

如果 `oldState` 很大，这会复制大量属性。

在 React/Zustand/Redux 场景里，浅拷贝是常见做法，但不能滥用在大对象上。

尤其是：

```js
const newSheet = {
  ...sheet,
  cells: {
    ...sheet.cells,
    [cellId]: newCell,
  },
};
```

如果 `cells` 是一个巨大对象，这会非常昂贵。

---

## 11.2 推荐：状态分层，缩小更新粒度

```js
const sheetMeta = {
  id,
  name,
  rowCount,
  colCount,
};

const cellStore = new Map();
const styleStore = new Map();
const formulaStore = new Map();
```

不要把所有东西塞进一个巨大的对象里：

```js
const workbook = {
  sheets: {
    sheet1: {
      cells: {
        A1: {
          value,
          style,
          formula,
          validation,
          conditionalFormat,
        },
      },
    },
  },
};
```

复杂前端应用更适合：

```txt
元信息对象：小、稳定、易拷贝
大数据索引：Map / TypedArray / 专用数据结构
渲染状态：单独维护
历史记录：增量 patch
```

---

# 十二、最佳实践 11：大规模数值数据优先考虑 TypedArray

如果你有大量纯数值数据：

```js
const rows = [];
const cols = [];
const widths = [];
const heights = [];
```

可以考虑：

```js
const rows = new Int32Array(n);
const cols = new Int32Array(n);
const widths = new Float64Array(n);
const heights = new Float64Array(n);
```

TypedArray 的优势：

1. 内存连续。
2. 单元素开销低。
3. 更接近底层数组。
4. GC 压力更小，因为大量数字不需要作为普通 JS 对象管理。
5. 适合 WASM、Canvas、WebGL、计算密集场景。

例如表格列宽、行高、单元格坐标、区间边界、依赖图索引，可以考虑用 TypedArray 或结构化二进制数据。

不过 TypedArray 不适合频繁插入删除中间元素的场景，适合：

```txt
固定长度
批量计算
数值密集
内存敏感
```

---

# 十三、最佳实践 12：不要滥用 Object.freeze / seal / preventExtensions 做性能优化

有些人会认为：

```js
Object.freeze(obj);
```

可以让对象更容易优化。

这不应该作为通用性能手段。

它的主要价值是：

```txt
语义约束、不可变性、防止误修改
```

而不是性能优化。

在一些场景下，冻结对象可能让 V8 知道对象结构不会再变，但它也会带来额外操作成本，并且不一定让你的业务代码更快。

建议原则：

| 目的        | 是否推荐    |
| --------- | ------- |
| 防止配置对象被修改 | 推荐      |
| 保障不可变语义   | 推荐      |
| 试图提升热路径性能 | 不推荐盲目使用 |
| 冻结大量临时对象  | 不推荐     |

例如：

```js
const CONFIG = Object.freeze({
  maxRow: 100000,
  maxCol: 20000,
});
```

可以。

但不要在热循环里：

```js
for (...) {
  Object.freeze(createCell(...));
}
```

---

# 十四、最佳实践 13：避免在热路径使用 Proxy

```js
const proxy = new Proxy(obj, {
  get(target, key) {
    return target[key];
  },
});
```

Proxy 很强大，但对 V8 优化不友好。

因为普通属性访问：

```js
obj.name
```

V8 可以通过 HiddenClass + Inline Cache 优化。

但 Proxy 的语义过于动态，任何属性访问都可能被 trap 拦截：

```js
get
set
has
deleteProperty
ownKeys
getOwnPropertyDescriptor
```

这会让引擎很难做静态假设。

建议：

| 场景           | 建议         |
| ------------ | ---------- |
| 调试、埋点、mock   | 可以用        |
| 响应式系统底层      | 可以用，但要控制范围 |
| 高频数据模型访问     | 谨慎         |
| 大量单元格对象代理化   | 不推荐        |
| 热循环里访问 Proxy | 尽量避免       |

例如表格里如果每个 cell 都包一层 Proxy：

```js
const proxyCell = new Proxy(cell, handlers);
```

当有几十万甚至百万级 cell 时，性能和内存都会很危险。

---

# 十五、最佳实践 14：原型方法优于每个对象挂函数

## 15.1 不推荐：每个对象一份方法

```js
function createCell(row, col) {
  return {
    row,
    col,
    getKey() {
      return `${this.row}:${this.col}`;
    },
  };
}
```

每创建一个 cell，都会创建一个新的函数对象。

---

## 15.2 推荐：方法放到 prototype 上

```js
class Cell {
  constructor(row, col) {
    this.row = row;
    this.col = col;
  }

  getKey() {
    return `${this.row}:${this.col}`;
  }
}
```

这样所有实例共享同一个方法：

```txt
cell1.__proto__.getKey === cell2.__proto__.getKey
```

好处：

1. 减少函数对象数量。
2. 降低内存占用。
3. 实例自身字段更干净。
4. shape 更稳定。

如果是大量模型对象，比如 Cell、Row、Column、Range、FormulaNode，这个收益会比较明显。

---

# 十六、最佳实践 15：不要把所有数据都塞到对象属性上

有些对象会越长越大：

```js
const sheet = {
  id,
  name,
  cells,
  styles,
  formulas,
  validations,
  comments,
  charts,
  filters,
  permissions,
  renderCache,
  layoutCache,
  dependencyGraph,
  historyStack,
};
```

这种超级对象的问题：

1. 生命周期过长。
2. 引用链复杂。
3. 很容易导致大块内存无法释放。
4. 修改任何部分都可能影响整体状态管理。
5. 调试困难。
6. GC 回收粒度差。

更好的做法是拆分：

```js
const sheetMeta = {};
const cellStore = new CellStore();
const formulaStore = new FormulaStore();
const renderCache = new RenderCache();
const historyStore = new HistoryStore();
```

从内存管理角度看，拆分后的好处是：

```txt
哪个模块不用了，就能独立释放
哪个缓存过大了，就能单独 clear
哪个数据需要懒加载，就单独加载
哪个数据需要持久化，就单独序列化
```

这对大型前端系统非常关键。

---

# 十七、最佳实践 16：冷热数据分离

对象里有些字段高频访问，有些字段低频访问。

例如：

```js
const cell = {
  row,
  col,
  value,
  formula,
  style,
  comment,
  validation,
  conditionalFormat,
  permission,
  revisionHistory,
};
```

如果每个 cell 都带这么多字段，内存会爆炸。

更好的方式是：

```js
const cell = {
  row,
  col,
  value,
  flags,
};
```

把低频数据放到外部索引：

```js
const formulaMap = new Map();
const styleMap = new Map();
const commentMap = new Map();
const validationMap = new Map();
```

也就是：

```txt
热数据：放对象本体，字段少，访问快
冷数据：放外部 Map，需要时再查
```

对于表格类业务，这个策略非常重要。

例如：

| 数据                | 访问频率  | 存储建议               |
| ----------------- | ----- | ------------------ |
| row / col / value | 高频    | cell 对象本体          |
| formula           | 中频    | 可放本体或单独索引          |
| style             | 中频/低频 | styleId + styleMap |
| comment           | 低频    | commentMap         |
| validation        | 低频    | validationMap      |
| conditionalFormat | 区间型   | 区间树/R-Tree         |
| permission        | 低频    | 独立权限索引             |
| revision history  | 低频    | 独立历史模块             |

这样既能降低对象大小，也能减少 Old Space 常驻内存。

---

# 十八、最佳实践 17：用 ID 引用代替大对象互相引用

## 18.1 不推荐：对象之间形成复杂引用图

```js
cell.sheet = sheet;
cell.rowObj = rowObj;
cell.colObj = colObj;
cell.style = styleObj;
cell.formulaNode = formulaNode;
```

这种引用图容易导致：

```txt
一个小对象没释放 -> 整个 sheet 被保活
一个事件回调没解绑 -> 整个 workbook 被保活
一个缓存没清理 -> 大量 cell 无法回收
```

---

## 18.2 推荐：用 ID / key 连接

```js
const cell = {
  row,
  col,
  value,
  styleId,
  formulaId,
};
```

外部再维护：

```js
const styleMap = new Map();
const formulaMap = new Map();
```

这样可以降低引用复杂度。

尤其是大型系统里，推荐：

```txt
对象本体轻量化
关系外部索引化
模块边界清晰化
生命周期独立化
```

---

# 十九、最佳实践 18：缓存要有边界，避免无限增长

性能优化经常会加缓存：

```js
const cache = new Map();
```

但缓存如果没有淘汰策略，本质上就是内存泄漏。

推荐至少具备：

```txt
最大容量
过期时间
作用域生命周期
主动 clear
命中率统计
内存占用监控
```

例如：

```js
class LRUCache {
  constructor(limit) {
    this.limit = limit;
    this.map = new Map();
  }

  get(key) {
    const value = this.map.get(key);
    if (value === undefined) return undefined;

    this.map.delete(key);
    this.map.set(key, value);
    return value;
  }

  set(key, value) {
    if (this.map.has(key)) {
      this.map.delete(key);
    }

    this.map.set(key, value);

    if (this.map.size > this.limit) {
      const firstKey = this.map.keys().next().value;
      this.map.delete(firstKey);
    }
  }

  clear() {
    this.map.clear();
  }
}
```

对于前端大型应用，缓存一定要问三个问题：

```txt
什么时候创建？
什么时候复用？
什么时候释放？
```

---

# 二十、最佳实践 19：不要迷信微优化，先定位瓶颈

V8 已经非常聪明，很多微优化未必有效。

例如：

```js
for 循环一定比 map 快吗？
class 一定比 object literal 快吗？
const 一定比 let 快吗？
解构一定慢吗？
```

这些问题没有绝对答案。

真正应该关注的是：

```txt
对象 shape 是否稳定？
数组是否变成 holey？
是否进入 dictionary mode？
是否有大量临时对象？
是否有闭包保活？
是否有意外 deopt？
是否有 GC 抖动？
```

推荐用工具验证：

| 工具                          | 用途              |
| --------------------------- | --------------- |
| Chrome DevTools Performance | 看长任务、JS 执行、渲染耗时 |
| Chrome DevTools Memory      | 看堆快照、对象保留路径     |
| Allocation instrumentation  | 看对象分配热点         |
| Performance Monitor         | 看 JS heap 实时变化  |
| Lighthouse / Web Vitals     | 看用户侧性能          |
| Node `--trace-gc`           | 看 GC 行为         |
| V8 trace/deopt 工具           | 看优化和反优化情况       |

性能优化的正确路径是：

```txt
发现问题 -> 采集数据 -> 定位热点 -> 建立假设 -> 修改代码 -> 对比验证
```

而不是凭感觉改代码。

---

# 二十一、结合 V8 对象存储的代码规范清单

## 1. 对象设计

推荐：

```js
const obj = {
  id: '',
  name: '',
  age: 0,
  extra: null,
};
```

避免：

```js
const obj = {};
obj.id = '';
if (cond) obj.name = '';
if (otherCond) obj.extra = {};
```

---

## 2. 字段删除

推荐：

```js
obj.field = null;
```

谨慎：

```js
delete obj.field;
```

如果是动态字典，改用：

```js
map.delete(key);
```

---

## 3. 动态 key

推荐：

```js
const map = new Map();
map.set(key, value);
```

不推荐：

```js
const obj = {};
obj[key] = value;
delete obj[key];
```

---

## 4. 数组

推荐：

```js
const arr = [];
arr.push(item);
```

避免：

```js
const arr = [];
arr[1000000] = item;
delete arr[10];
```

---

## 5. 热路径对象

推荐：

```js
function updateCell(cell) {
  cell.value = compute(cell.value);
}
```

避免：

```js
function updateCell(cell) {
  return {
    ...cell,
    value: compute(cell.value),
  };
}
```

尤其在大循环里要谨慎对象展开。

---

## 6. 函数入参

推荐：

```js
function renderCell(cell) {
  return cell.value;
}

// 所有 cell 都来自统一 factory
const cell = createCell(raw);
```

避免：

```js
renderCell({ value: 1 });
renderCell({ value: 1, style: {} });
renderCell({ value: 1, formula: '=A1' });
```

---

## 7. 大对象生命周期

推荐：

```js
cache.clear();
removeEventListener('click', handler);
handler = null;
```

避免：

```js
window.handler = () => {
  console.log(largeWorkbook);
};
```

---

# 最后关键总结

## 1. 对象 shape 稳定是第一原则

```txt
字段固定、顺序一致、初始化完整
```

这样 V8 才能共享 HiddenClass，并让属性访问走高效 Inline Cache。

---

## 2. 少用 delete

```txt
delete 可能让对象退化为 Dictionary Mode
```

固定 schema 对象中，优先使用：

```js
obj.field = null;
```

动态 key 场景使用：

```js
Map
```

---

## 3. 热路径函数入参要稳定

同一个函数不要一会儿接收 A shape，一会儿接收 B shape。

```js
function renderCell(cell) {
  return cell.value;
}
```

这里的 `cell` 最好来自统一 factory 或 class。

---

## 4. 数组要连续、紧凑、同类型

避免：

```js
delete arr[i]
arr[1000000] = x
[1, 2, '3', {}, null]
```

推荐：

```js
arr.push(x)
```

大规模数值数据考虑：

```js
TypedArray
```

---

## 5. Object 和 Map 分工明确

```txt
固定字段业务实体：Object / class
动态 key 字典缓存：Map
```

不要用普通对象承载大量动态增删 key。

---

## 6. 少创建、少保活、及时释放

性能问题很多时候不是“访问慢”，而是：

```txt
对象太多
临时对象太多
缓存无限增长
闭包保活大对象
事件监听未解绑
```

---

## 7. 大型前端系统要做冷热数据分离

尤其是表格、文档、画布、编辑器类系统：

```txt
高频字段放对象本体
低频字段放外部索引
大数据放 Map / TypedArray / 专用数据结构
缓存必须有释放策略
```

---

## 8. 不要为了 V8 牺牲代码结构

最佳策略不是写“奇技淫巧”的 JS，而是：

```txt
数据模型稳定
访问路径稳定
生命周期清晰
结构分层合理
性能工具验证
```

真正高质量的 JS 性能优化，本质上是：

> **让业务数据结构更贴近 V8 的优化模型，同时保持代码可维护、可验证、可演进。**

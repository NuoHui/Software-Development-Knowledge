# 引擎与编译

## v8

```javascript
JS代码
 ↓
Parser（解析）
 ↓
AST（抽象语法树）
 ↓
Interpreter（解释执行）
 ↓
Profiler 收集热点
 ↓
JIT 编译
 ↓
优化代码
 ↓
去优化（Deopt）
```

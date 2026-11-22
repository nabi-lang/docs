# 错误码

nabic 的每条诊断都以一个错误码开头：`error[E0040]: mismatched types: ...`。每类错误一个错误码，所以一个错误码涵盖多条消息。消息本身会说明哪里出了问题，通常还会说明应该改写成什么。`nabic --explain E0040`（或 `nabi --explain E0040`）会打印错误码的简短说明，本站对应的页面内容相同，另外附有示例。使用 `--message-format=json` 时，错误码在 `code` 字段中，所属类别在 `kind` 字段中（[9.9.4 诊断](../spec/refs-lang/009-tools/909-debug-tools.html#994-diagnostics--error-messages)）。

错误码按加入的先后顺序编号，从 E0001 开始，中间没有空缺。本索引对应 nabic 0.1.0。大多数页面都给出一个示例和它的修正，示例会在构建文档时与编译器的实际输出核对。

## 词法

| 错误码 | 类别 | 标题 |
|------|------|-------|
| [E0001](E0001.html) | `lexical` | 源代码中有意外的字符 |
| [E0002](E0002.html) | `lexical` | 源代码中有双向控制字符 |
| [E0003](E0003.html) | `lexical` | 格式错误的数字字面量 |
| [E0004](E0004.html) | `lexical` | 字面量超出其类型的范围 |
| [E0005](E0005.html) | `lexical` | 未结束的字面量或注释 |
| [E0006](E0006.html) | `lexical` | 字面量中有无效的转义 |
| [E0007](E0007.html) | `lexical` | 字符字面量只能包含一个字符 |
| [E0008](E0008.html) | `lexical` | 格式错误的 f-string |
| [E0009](E0009.html) | `lexical` | 不能用作原始标识符的名称 |
| [E0147](E0147.html) | `lexical` | 代码中有不可见字符 |
| [E0148](E0148.html) | `lexical` | 只是看起来像词法单元的字符 |

## 语法

| 错误码 | 类别 | 标题 |
|------|------|-------|
| [E0010](E0010.html) | `parse` | 意外的词法单元 |
| [E0011](E0011.html) | `parse` | 嵌套过深 |
| [E0012](E0012.html) | `parse` | 其他语言的语法 |
| [E0013](E0013.html) | `parse` | 块头部的结构体字面量 |
| [E0014](E0014.html) | `parse` | 修饰符顺序错误 |
| [E0015](E0015.html) | `parse` | 没有描述任何内容的文档注释 |
| [E0016](E0016.html) | `parse` | 在没有意义的位置使用可见性 |
| [E0017](E0017.html) | `parse` | 位置错误的项 |
| [E0018](E0018.html) | `parse` | 顶层语句中带值的 `return` |
| [E0019](E0019.html) | `parse` | 需要括号的运算符 |
| [E0020](E0020.html) | `parse` | 尾随闭包中的 `return` |
| [E0021](E0021.html) | `parse` | 用作名称的保留字 |
| [E0022](E0022.html) | `parse` | 没有值的 `let` |
| [E0023](E0023.html) | `parse` | 无效的函数签名 |
| [E0024](E0024.html) | `parse` | 无效的泛型参数 |
| [E0025](E0025.html) | `parse` | 不支持的 ABI |

## 名称与模块

| 错误码 | 类别 | 标题 |
|------|------|-------|
| [E0026](E0026.html) | `resolve` | 名称定义了多次 |
| [E0027](E0027.html) | `resolve` | 找不到值或函数 |
| [E0028](E0028.html) | `resolve` | 找不到类型或 trait |
| [E0029](E0029.html) | `resolve` | 找不到枚举变体 |
| [E0030](E0030.html) | `module` | 无法解析的导入 |
| [E0031](E0031.html) | `resolve` | 有歧义的名称 |
| [E0032](E0032.html) | `module` | 相互冲突的导入 |
| [E0033](E0033.html) | `module` | 无效的重新导出 |
| [E0034](E0034.html) | `visibility` | 项是私有的 |
| [E0035](E0035.html) | `module` | 模块文件有问题 |
| [E0036](E0036.html) | `module` | 无效的程序入口点 |
| [E0037](E0037.html) | `module` | 无效的依赖 |
| [E0038](E0038.html) | `resolve` | 变量与嵌套函数同名 |
| [E0039](E0039.html) | `resolve` | 嵌套项使用了外层的局部变量 |
| [E0128](E0128.html) | `module` | 无效的 WIT |
| [E0135](E0135.html) | `resolve` | 方法所属的 trait 不在作用域中 |
| [E0141](E0141.html) | `resolve` | 方法的泛型参数与所在 impl 的同名 |

## 类型

| 错误码 | 类别 | 标题 |
|------|------|-------|
| [E0040](E0040.html) | `type` | 类型不匹配 |
| [E0041](E0041.html) | `type` | 分支的类型不同 |
| [E0042](E0042.html) | `type` | 需要类型标注 |
| [E0043](E0043.html) | `type` | 没有这个字段 |
| [E0044](E0044.html) | `type` | 字面量缺少字段 |
| [E0045](E0045.html) | `type` | 字段给出了多次 |
| [E0046](E0046.html) | `type` | 没有这个方法或关联项 |
| [E0047](E0047.html) | `type` | 以错误的形式使用项 |
| [E0048](E0048.html) | `type` | 实参个数不对 |
| [E0049](E0049.html) | `type` | 无效的具名参数 |
| [E0050](E0050.html) | `type` | `ref` 参数与实参不匹配 |
| [E0051](E0051.html) | `type` | 无效的展开实参 |
| [E0052](E0052.html) | `type` | 值不可调用 |
| [E0053](E0053.html) | `type` | 无效的强制转换 |
| [E0054](E0054.html) | `type` | 无法应用运算符 |
| [E0055](E0055.html) | `type` | 字面量适合多种类型 |
| [E0056](E0056.html) | `type` | 无效的索引 |
| [E0057](E0057.html) | `type` | 无效的赋值 |
| [E0058](E0058.html) | `type` | 无效的闭包捕获 |
| [E0059](E0059.html) | `type` | 关键字在其上下文之外 |
| [E0060](E0060.html) | `type` | 不发散的 `let .. else` |
| [E0061](E0061.html) | `type` | 操作需要 `unsafe` |
| [E0062](E0062.html) | `type` | 错误未处理 |
| [E0063](E0063.html) | `type` | 抛出的值不是 `Error` |
| [E0064](E0064.html) | `type` | 对非可选值使用可选运算符 |
| [E0065](E0065.html) | `type` | 锁守卫没有由 `using` 释放 |
| [E0066](E0066.html) | `type` | 对没有 `Drop` 的值使用 `using` |
| [E0067](E0067.html) | `type` | 块语句留下了值 |
| [E0068](E0068.html) | `type` | 值不可迭代 |
| [E0069](E0069.html) | `type` | 不满足 trait 约束 |
| [E0070](E0070.html) | `type` | 不是 trait |
| [E0071](E0071.html) | `type` | 把 trait 用作类型 |
| [E0072](E0072.html) | `type` | impl 缺少 trait 项 |
| [E0073](E0073.html) | `type` | impl 项不在 trait 中 |
| [E0074](E0074.html) | `type` | impl 项与 trait 不一致 |
| [E0075](E0075.html) | `type` | 相互冲突的实现 |
| [E0076](E0076.html) | `type` | 为外部类型实现外部 trait |
| [E0077](E0077.html) | `type` | 未实现父 trait |
| [E0078](E0078.html) | `type` | 不允许的 impl |
| [E0079](E0079.html) | `type` | 无法在编译时求值 |
| [E0080](E0080.html) | `type` | 定义依赖自身 |
| [E0081](E0081.html) | `type` | 无效的常量泛型实参 |
| [E0082](E0082.html) | `type` | 泛型实参不对 |
| [E0083](E0083.html) | `type` | 无效的枚举值 |
| [E0084](E0084.html) | `type` | 无效的外部函数用法 |
| [E0085](E0085.html) | `type` | 值不能跨线程 |
| [E0086](E0086.html) | `type` | 无效的联合体 |
| [E0087](E0087.html) | `type` | 有歧义的关联项 |
| [E0088](E0088.html) | `type` | 无效的格式 |
| [E0089](E0089.html) | `type` | 无效的编译器内建调用 |
| [E0090](E0090.html) | `type` | 无效的协程用法 |
| [E0091](E0091.html) | `type` | 无效的广播 |
| [E0092](E0092.html) | `type` | 直接调用 `Drop::drop` |
| [E0093](E0093.html) | `type` | 泛型实例化过深 |
| [E0094](E0094.html) | `type` | 无效的表示 |
| [E0095](E0095.html) | `type` | 类型过大 |
| [E0096](E0096.html) | `type` | 不起作用的 `try` 或 `catch` |
| [E0097](E0097.html) | `type` | 形状不对的 NABION 值 |
| [E0138](E0138.html) | `type` | `pub` 函数的签名没有写全 |
| [E0140](E0140.html) | `type` | 对不是位置的值赋值 |
| [E0142](E0142.html) | `type` | 数组字面量各行长度不同 |
| [E0143](E0143.html) | `type` | 对只沿一个轴索引的类型使用多轴索引 |
| [E0144](E0144.html) | `type` | 整数的负数次幂 |
| [E0145](E0145.html) | `type` | 具名 `some` 类型不是由其模块定义的 |
| [E0146](E0146.html) | `type` | 把具名 `some` 类型当作其具体类型使用 |

## 模式

| 错误码 | 类别 | 标题 |
|------|------|-------|
| [E0098](E0098.html) | `pattern` | 不穷尽的 match |
| [E0099](E0099.html) | `pattern` | 不可达的 match 分支 |
| [E0100](E0100.html) | `pattern` | 模式与值的类型不符 |
| [E0101](E0101.html) | `pattern` | 模式中的字段个数不对 |
| [E0102](E0102.html) | `pattern` | 或模式的各个选项绑定不同 |
| [E0103](E0103.html) | `pattern` | 看起来像变体的绑定 |
| [E0104](E0104.html) | `pattern` | 无效的剩余模式 |

## 移交与资源

| 错误码 | 类别 | 标题 |
|------|------|-------|
| [E0105](E0105.html) | `handoff` | 值移交之后又被使用 |
| [E0106](E0106.html) | `handoff` | 值在其锁释放之后又被使用 |
| [E0107](E0107.html) | `handoff` | 移交了共享的值 |
| [E0133](E0133.html) | `handoff` | 资源在移动或关闭之后又被使用 |
| [E0134](E0134.html) | `handoff` | 资源没有关闭 |
| [E0136](E0136.html) | `handoff` | 线程局部值在 `with` 之后仍被保留 |
| [E0137](E0137.html) | `handoff` | 列表视图在改变大小之后又被使用 |
| [E0139](E0139.html) | `handoff` | 流视图在其流之后仍被使用 |

## 属性与元系统

| 错误码 | 类别 | 标题 |
|------|------|-------|
| [E0108](E0108.html) | `meta` | 无法派生 |
| [E0109](E0109.html) | `meta` | 格式错误的属性 |
| [E0110](E0110.html) | `meta` | 属性用在了错误的项上 |
| [E0111](E0111.html) | `meta` | 无效的 `#[lang]` 项 |
| [E0112](E0112.html) | `meta` | 无效的运算符声明 |
| [E0113](E0113.html) | `meta` | 无效的 `#[keyword]` 声明 |
| [E0114](E0114.html) | `meta` | 无效的字面量处理器 |
| [E0115](E0115.html) | `meta` | 无效的 `#[intrinsic]` |
| [E0116](E0116.html) | `meta` | 格式错误的宏定义 |
| [E0117](E0117.html) | `meta` | 没有匹配的宏规则 |
| [E0118](E0118.html) | `meta` | 无效的元模板 |
| [E0119](E0119.html) | `meta` | 无效的元块 |
| [E0120](E0120.html) | `meta` | 递归展开 |
| [E0121](E0121.html) | `meta` | 无效的 `cfg` 谓词 |
| [E0122](E0122.html) | `meta` | 文件包含失败 |
| [E0123](E0123.html) | `meta` | 注册了两次 |
| [E0124](E0124.html) | `meta` | 无法解析的元名称 |

## 构建与后端

| 错误码 | 类别 | 标题 |
|------|------|-------|
| [E0125](E0125.html) | `other` | 无法读取或写入文件 |
| [E0126](E0126.html) | `linkage` | 链接失败 |
| [E0127](E0127.html) | `linkage` | 外部工具失败 |
| [E0129](E0129.html) | `unsupported` | 此后端不支持 |
| [E0130](E0130.html) | `internal` | 编译器内部错误 |
| [E0132](E0132.html) | `other` | 无效的编译器选项 |
| [E0149](E0149.html) | `type` | 运行中的程序无法接受的修改 |

## Lint

| 错误码 | 类别 | 标题 |
|------|------|-------|
| [E0131](E0131.html) | `lint` | lint |

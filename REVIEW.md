# xyz 规范评审清单

评审范围：`01-types.md` … `12-macros.md`。
流程：逐条讨论定稿，定了的打勾并写入结论，最后按结论统一改正文。

状态标记：`待定` / `已定` / `已修`

## A. 必修矛盾

- [x] **A1 `@field` 编译期字段名 vs 运行时 `f.name`**
      `08-reflection.md:238` 要求名字编译期已知；
      `10-iteration.md:363` 与 `12-macros.md:22` 在运行时循环里传 `f.name`。
      结论：引入 `comptime for` —— 被迭代对象必须编译期已知，循环在编译期展开，
      循环变量随之成为编译期常量。已落地：06（Field access 段 + `comptime` 定义句）、
      08（新增 Comptime For 小节 + Serialization 示例）、10（示例）。
      状态：已定

- [x] **A2 方法调用隐式取地址 vs「没有隐式借用」**
      `05-traits.md:22-24`、`03-move.md:107-108` 声明无隐式借用；
      但 `p.len()`（`05:63-67`）、`it.next()`（`08:186`）、`self.iter.next()`（`08:336`）都在隐式取址。
      结论：方法调用是糖 —— 接收者按方法声明的 `self` 适配（`*Self`→`&x`，`*mut Self`→`&mut x`，
      按值则不取址）；`&mut` 仍要求 `mut` 槽位；「无隐式借用」收缩为只管实参。
      已落地：`05-traits.md` 该段改写。
      状态：已定

- [x] **A3 切片字段可写性两种说法**
      `01-types.md:100-102`（由绑定的 `mut` 决定）vs `01-types.md:22-27,49-51`（由字段 `mut` 决定）；
      `10-iteration.md:45-47` 通过 `*mut Self` 写 `self.ptr`/`self.len`，两种规则下都不成立。
      结论：统一到字段级 `mut`；切片的 `ptr`/`len` 天生是 `mut` 字段（推进视窗不写被借数据，
      与「`[]T` 元素不可写」不冲突）；整体赋值仍由绑定的 `mut` 决定。
      已落地：`01-types.md` 该段改写、`10-iteration.md` 的 `Enumerate` 字段补 `mut`。
      状态：已定

- [x] **A4 `@take` 示例违反前提**
      `03-move.md:200` `take(x) = @take(&x)` 应为 `&mut x`；
      `03-move.md:187-189` `let a` 不是 `mut` 却取 `&mut a`（违反 `01:320-323`）。
      结论：`@take` 收 `*mut T`（与 `@field` 返回地址一致，不破坏「无隐式借用」）。
      已落地：03（`let mut a`、`@take(&mut x)`、`@take(&mut u.f)` 及说明）、
      01（Union 段补「`let mut u` ⇒ 可对字段取 `&mut`」）。
      状态：已定

- [x] **A5 拥有式数组迭代 = 从索引位置移动**
      `10-iteration.md:240,123-127` 产出 `T`；`03-move.md:50-53` 禁止从位置移出。
      未说明元素如何离开数组（`@take`？）。
      结论：`ArrayIter` 用 `@take` 逐元素取出；为此允许「移动到新槽位时提升可写性」
      （`[N]T` → `[N]mut T`），理由是新槽位无别名。
      已落地：01（mutability 段补例外）、08（写出 `ArrayIter` 及其 `Iterator` 实现）。
      状态：已定

- [x] **A6 `mut Self` vs「没有独立的 `mut T`」**
      `05-traits.md:128` `fn drop(self: mut Self)`；`01-types.md:62-63` 禁止独立 `mut T`。
      结论：写成 `mut self: Self` —— 参数是槽位，与「`mut` 标记紧随其后的槽位」一致；
      「不存在独立 `mut i32` 类型」原样成立。顺带纠正：所有权来自按值传参，不来自 `mut`。
      已落地：05（两处签名 + 解释句）、01（mutability 段补「参数槽位」）。
      状态：已定

- [x] **A7 部分移动示例结论矛盾**
      `03-move.md:77-78`：`let y = w.big` 被拒后，`let z = w` 不应再报「w 不可完整使用」。
      结论：`let z = w` 改为 ✅（上一行的移动被拒，`w` 仍完整）；
      并补一句「绑定要么完整要么已死」——这正是静态插入析构的前提。
      已落地：`03-move.md` 示例注释 + 新段落。
      状态：已定

- [x] **A8 `#if` 是否统一为 `comptime if`**（讨论中）
      A1 落地后 `comptime` 成为「要求编译期已知」的唯一标记，可顺势把 `#if` 改为
      `comptime if`（顺带删掉「无括号、条件取到块为止」的解析特例），
      第十章的内置 `#` 构造随之由三个减为两个（`#repr`、`#assert`）。
      结论：采用。`comptime` 一个标记三个位置（参数 / `if` / `for`），06 给出表；
      普通 `if`/`for` 输入恰好编译期已知时也折叠，`comptime` 只是把它变成硬约束。
      第十章只做事实性改动（Three→Two、删 `#if` 行），立论与用词不动——
      宏的设计尚未定，措辞保持开放。
      已落地：04、06、08、10。
      状态：已定

## B. 缺口

- [x] **B1 `Copy` 覆盖面不全** — `03-move.md:36` 未提切片、元组、`?T`/`Option<T>`；第六、八章默认切片是 `Copy`。
      结论：补切片（`[]mut T` 也是）、元组、以及「枚举跟随载荷」——
      原句会让 `Option<File>` 成为 `Copy`，与 `Copy`/`Drop` 互斥冲突（double free）。
      已落地：`03-move.md` Copy 段改写。
- [x] **B2 特异性排序缺 `mut`** — `04-generics.md:117-123`；依赖点 `08:80-82`、`08:135-149`。
      结论：表里补 `*mut T`/`*T`、`[]mut T`/`[]T`、`*mut [N]T`/`*[N]T` 三行，
      并补一句「可写更具体，因为可写类型能用在不可写处，反之不行」。
      已落地：`04-generics.md` 形状模式表。
- [x] **B3 `EnumField` 无载荷类型** — `08-reflection.md:103-106`；无法描述 `Some(T)`；`TypeInfo` 缺 `type`、`?[]T`。
      结论：`EnumField` 加 `type: ?type`（`None` = 无载荷，与 `Field.type` 对齐）；
      `Slice` 加 `optional: bool`（与 `Pointer` 对齐，兑现 06 那句「nullable 在指针和切片上可见」）。
      `type` 自身不加 `TypeInfo` 变体。
      已落地：`08-reflection.md` 的 `TypeInfo` 与 `EnumField`。
- [x] **B4 反射数据的编译期性质未声明** — 与 A1 联动。
      结论：已由 A1 的 `comptime for` 覆盖（06 的 Field access 段已说明遍历方式是 `comptime for`）。
- [x] **B5 无 builtin 总表** — `@if`/`@cast`/`@take`/`@print`/`@close` 未汇总（`06:273-281` 只覆盖一半）。
      结论：06 新增 `## Builtins` 总表（十一个内置，按用途分组，标出定义章节），
      原 Queries 表降级为 `### Queries` 子节；`@print`/`@close` 改为 `std::io` 的普通函数。
      已落地：06（Builtins 节）、05（示例改用 `print`/`close` 并加说明）。
- [x] **B6 无结构体声明未引入** — `struct is_same<A, B>;`（`04:134`、`05:76`、`06:251`）。
      结论：统一成 `struct is_same<A, B> {}` —— 不引入 `struct X;` 这第四种形式
      （`;` 在 C 里是前向声明，xyz 没有声明/定义分离；特化信息在参数模式里，不在函数体）。
      另补一句「struct 的类型参数不必出现在字段里」。
      已落地：04（`{}` + 参数说明）、05、06。
- [x] **B7 第七章 open items 过期** — `09-match.md:144-149` 仍说 `return` 未定义（`08:257` 已定义）。
      结论：删掉该条目，开头指路句改为「Loops and `return` come in `10-iteration.md`」。
      已落地：`09-match.md`（开头 + open items）。
- [x] **C9 与 B7 重复** — `09-match.md:144-149` 与 `08` 的交叉引用问题，已随 B7 解决。
- [x] **B8 `if` 是否为表达式** — 第四章的 `#if` 出现在值位置；若按 A8 改为 `comptime if`，需明确 `if` 是表达式（与 `match` 对齐）。
      结论：是。「值是所取分支的值，两分支类型须一致」；无 `else` 时未取路径产出 `()`，
      故只作语句用。已落地：`10-iteration.md` 新增 `## If` 节。

## C. 文字问题

- [x] C1 `04-generics.md` 断句 —— 改为 "for every concrete type it is called with:"
- [x] C2 `04-generics.md`「neither bound」—— 改为 "A type that is neither `Copy` nor `File`"
- [x] C3 `08-reflection.md` `twice(twice(1))` —— 运行时那半改用 `io::read_u32()`，使其真为运行时
- [x] C4 `01-types.md`「nor arithmetic on」—— 改为 "nor used in pointer arithmetic"
- [x] C5 `01-types.md:3` `you‘ve` —— 左单引号 U+2018 改为 `you've`
- [x] C6 `02-layout.md` 图表列宽对齐；`#repr(align(16))` 行尾空格删除
- [x] C7「module」→「namespace」—— `05-traits.md` 与 `04-generics.md` 两处
- [x] C8「枚举是变体的命名空间」—— 在 `09-match.md` 的 Patterns 段补写，而非改 09 的表述
- [x] C9 与 B7 重复，已随 B7 解决

## 进度

当前讨论：**全部完成**（A 类 8 条、B 类 8 条、C 类 9 条均已处理）
剩余开放项：宏的设计（第十章立论与用词刻意未动）。

## 备注

- 宏的设计尚未定，第十章的立论与「macro」用词刻意未动，只做了 `#if` 相关的事实性删改。
- `01-types.md:3`「`#xxx` is a macro」保持原样，同上。

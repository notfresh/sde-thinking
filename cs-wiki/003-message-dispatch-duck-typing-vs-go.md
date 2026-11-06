# 消息派发与鸭子类型：Go 隐式接口的对比

> cs-wiki·第 003 篇。一篇笔记，三句话：消息派发（Ruby）和隐式接口（Go）都免去了显式继承，但一个在运行时判定"这条消息有没有鸭子接"，一个在编译期就锁死"你的方法集像不像鸭子"。

## 先把名词固定

| 名字 | 是什么 |
|---|---|
| **消息派发（message dispatch）** | 方法调用被理解为"向接收者发送一条消息"，接收者在运行时决定能否处理这条消息，类型系统不参与决策 |
| **鸭子类型（duck typing）** | "如果它走起来像鸭子、叫起来像鸭子，那它就是鸭子"——对象能否响应某条消息决定了一切，不看它继承了哪个类 |
| **隐式接口** | Go 的接口满足机制：类型不用写 `implements`，方法集匹配就自动算实现 |
| **结构化类型系统** | 通过"类型拥有哪些方法"而不是"类型继承自谁"来判定类型关系的系统，Go 属于这一类 |

一句话关系：**消息派发问的是"你能做什么"，而不是"你是谁"**——鸭子类型是它的直接推论。

---

## Ruby 例子：纯消息派发 + 运行时鸭子类型

```ruby
class Duck
  def quack; "Quack!"; end
end

class Person
  # 完全没有继承 Duck，也没有共同父类
  def quack; "I'm imitating a duck: quack quack!"; end
end

class Robot
  def quack; "BEEP BOOP: QUACK"; end
end

# 不关心参数类型，只关心能否响应 quack 消息
def make_it_quack(thing)
  thing.quack
end

make_it_quack(Duck.new)   # => "Quack!"
make_it_quack(Person.new) # => "I'm imitating a duck: quack quack!"
make_it_quack(Robot.new)  # => "BEEP BOOP: QUACK"
```

要点：

- `Duck` / `Person` / `Robot` **互不继承**，无共同基类
- `make_it_quack` 不做任何类型声明或检查，运行时直接发 `quack` 消息
- 能响应就工作，否则运行时报 `NoMethodError`

对比静态类型（Java）：必须显式 `implements Quacker`，类型关系是**声明出来的**，编译期强制。

---

## 和 Go 的相似：都不要求显式声明"我实现了它"

Go 接口是**隐式满足**的，类型不需要写 `implements`，方法集匹配就自动算实现：

```go
type Quacker interface {
	Quack() string
}

type Duck struct{}
func (d Duck) Quack() string { return "Quack!" }

type Robot struct{}
func (r Robot) Quack() string { return "BEEP BOOP: QUACK" }

func MakeItQuack(q Quacker) string { return q.Quack() }

MakeItQuack(Duck{})  // OK
MakeItQuack(Robot{}) // OK，Robot 从未声明实现 Quacker
```

---

## 关键区别：检查时机与机制

| | Ruby（消息派发） | Go（隐式接口） |
|---|---|---|
| 检查时机 | **运行时**，发消息时才查 | **编译期**，静态验证方法集 |
| 判定依据 | 对象此刻能否响应这条消息 | 类型方法集是否**完整匹配**接口签名 |
| 失败表现 | 运行时 `NoMethodError` | 编译错误，编译不过 |
| 动态性 | 可运行时加方法（单例方法、`method_missing`） | 方法集编译期固定，不能运行时增删 |
| 典型能力 | `respond_to?`、`method_missing`、动态代理 | 无（接口是静态契约） |

---

## 最能体现差异的两个点

### 1. 运行时动态性

Ruby 可让对象动态响应任意消息：

```ruby
class Ghost
  def method_missing(name, *args)
    name.to_s.start_with?("quack") ? "ghostly quack" : super
  end
end

make_it_quack(Ghost.new) # => "ghostly quack"，Ghost 根本没有 quack 方法
```

Go 做不到：`Ghost` 没有 `Quack()` 方法就**编译不过**。Go 的"鸭子"是编译期就认定好的。

### 2. 签名必须精确匹配

Go 接口要求方法签名完全一致（如 `Quack() string` 写成 `Quack() (string, error)` 就不满足）；Ruby 只看消息有没有人接，参数、返回值不做静态约束。

---

## 一句话总结

> **静态继承**问的是"你是谁（哪个类）"；**消息派发**问的是"你能做什么（能否响应这条消息）"。
>
> Go 是"**编译期的鸭子类型**"：靠隐式接口在编译时静态判定"长得像不像鸭子"，本质是**结构化类型系统**；
> Ruby 是"**运行时的鸭子类型**"：靠消息派发在运行时判定"这条消息有没有鸭子接"，本质是**动态消息机制**。
>
> 两者都免去了显式继承/声明，但一个在编译期锁死契约，一个在运行时保持开放。所以 Go 常被说"像鸭子类型"，而 Ruby 才是"消息派发 + 鸭子类型"的原教旨版本。

**一句话总结**：Go 的隐式接口 = 结构化类型 + 编译期检查；Ruby 的鸭子类型 = 消息派发 + 运行时检查。理解"检查发生在什么时刻"，是区分两者的关键。

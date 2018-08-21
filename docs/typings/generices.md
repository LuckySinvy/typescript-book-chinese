# 范型

设计范型的关键目的是在成员之间提供有意义的约束，这些成员可以是：

- 类的实例成员
- 类的方法
- 函数参数
- 函数返回值

## 动机和示例

考虑如下简单的 `Queue` （先进先出）数据结构实现，一个在 `TypeScript` 和 `JavaScript` 中的简单实现：

```ts
class Queue {
  private data = []
  push = item => this.data.push(item)
  pop = () => this.data.shift
}
```

在上述代码中存在一个问题，它允许你推入任何类型至队列中，推出的时候也是任意类型，如下所示，但一个人推入一个 `string` 类型至队列中，但是使用者可能会认为队列里只有 `number` 类型：

```ts
class Queue {
  private data = []
  push = item => this.data.push(item)
  pop = () => this.data.shift
}


const queue = new Queue()

queue.push(0)
queue.push('1') // Oops，一个错误

// 一个使用者，走入了误区
console.log(queue.pop().toPrecision(1))
console.log(queue.pop().toPrecision(1))  // RUNTIME ERROR
```

一个解决的办法（事实上，这也是不支持范型类型的唯一解决办法）是为这些约束创建特殊类，如快速创建数字类型的队列：

```ts
class QueueNumber {
  private data = []
  push = (item: number) => this.data.push(item)
  pop = (): number => this.data.shift
}


const queue = new QueueNumber()

queue.push(0)
queue.push('1') // Error: 不能推入一个 `string` 类型，只能是 `number` 类型

// 如果该错误得到修复，其他将不会出现问题
```

当然，快速意为着痛苦的。例如但你想创建一个字符串的队列时，你将不得不再次修改相当大的代码。我们真正想要的一种方式是无论什么类型被推入队列，被推出的类型都与推入类型一样。当你使用范型时，这会很容易：

```ts
// 创建一个范型类
class Queue<T> {
  private data = []
  push = (item: T) => this.data.push(item)
  pop = (): T => this.data.shift()
}

// 简单的使用
const queue = new Queue<number>()
queue.push(0)
queue.push('1') // Error：不能推入一个 `string`，只有 number 类型被允许
```

另外一个我们见过的例子：一个 `reverse` 函数，现在在这个函数里提供了函数参数与函数返回值的约束：

```ts
function reverse<T> (items: T[]): T[] {
  const toreturn = []
  for (let i = items.length - 1; i >= 0; i--) {
    toreturn.push(items[i])
  }
  return toreturn
}

const sample = [1, 2, 3]
const reversed = reverse(sample)

reversed[0] = '1'         // Error
reversed = ['1', '2']     // Error

reversed[0] = 1           // ok
reversed = [1, 2]          // ok
```

在此章节中，你已经了解在*类*和*函数*上使用范型的例子。一个值得补充一点的是，你可以为创建的成员函数添加范型：

```ts
class Utility {
  reverse<T>(items: T[]): T[] {
    const toreturn = []
    for (let i = items.length; i >= 0; i--) {
      toreturn.push(items[i])
    }
    return toreturn
  }
}
```

::: tip
你可以随意调用范型参数，当你使用简单的范型时，范型常用 `T`、`U`、`V` 表示。如果在你的参数里，不止拥有一个范型，你应该使用一个更语义化名字，如 `TKey` 和 `TValue` （通常情况下，以 `T` 做为范型前缀也在如 C++ 的其他语言里做为模版。）
:::

## 误用的范型

我见过人们使用范型仅仅是为了它的 hack。我想问的问题是，你想用它来提供什么样的约束。如果你不能很好的回答它，你可能会误用范型，如：

```ts
declare function foo<T>(arg: T): void
```

在这里，范型完全没有必要使用，因为它仅用于当个参数的位置，使用如下方式可能更好：

```ts
declare function foo (arg: any): void
```

## 设计模式：方便通用

考虑如下函数：

```ts
declare function parse<T>(name: string): T
```

在这种情况下，范型 `T` 只在一个地方被使用了，它并没有在成员之间提供约束 `T`。这相当于一个如下的类型断言：

```ts
declare function parse(name: string): any

const something = parse('something') as TypeOfSomething
```

仅使用一次的范型并不比一个类型断言来的安全。它们都给你使用 API 提供了便利。

另一个明显的例子是，一个用于加载 json 返回值函数，它返回你任何传入类型的 `Promise`：

```ts
const getJSON = <T>(config: {
  url: string,
  headers?: { [key: string]: string },
}): Promise<T> => {
  const fetchConfig = ({
    method: 'GET',
    'Accept': 'application/json',
    'Content-Type': 'application/json',
    ...(config.headers || {})
  })
  return fetch(config.url, fetchConfig)
    .then<T>(response => response.json())
}
```

请注意，你仍然需要明显的注释任何你需要的类型，但是 `getJSON<T>` 的签名 `config => Promise<T>` 能够减少你一些关键的步骤（你不需要注释 `loadUsers` 的返回类型，因为它能够被推出来）：

```ts
type LoadUserResponse = {
  user: {
    name: string,
    email: string
  }[]
}

function loaderUser () {
  return getJSON<LoadUserResponse>({ url: 'https://example.com/users' })
}
```

与此类似：使用 `Promise<T>` 做为一个函数的返回值比一些如：`Promise<any>` 的备选方案要好很多。
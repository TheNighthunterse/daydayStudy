#### 构造函数与普通函数的区别

结论：`new` 会开启“工厂模式”，创造出一个新对象；而不写 `new` 就是一次普通的执行，往往会掉进全局变量的坑里

##### 核心差异对比

```javascript
function Person(name) {
    this.name = name;
    this.sayHi = function() { console.log(this.name); };
}
```

| **特性**        | **const p = new Person('A')** | **const p = Person('A')**                       |
| --------------- | ----------------------------- | ----------------------------------------------- |
| **执行性质**    | **构造函数调用**              | **普通函数调用**                                |
| **`this` 指向** | 指向**新创建的实例对象**      | 指向**全局对象**（浏览器里是 `window`）         |
| **返回值**      | 默认返回那个**新对象**        | 取决于函数里的 `return`（没写就是 `undefined`） |
| **后果**        | 成功创建一个 `Person` 实例    | 意外地给全局增加了 `name` 属性                  |

##### 写 `new` 时发生了什么？

当写下 `new Person('A')` 时，后台悄悄执行了四件事：

1. **开辟空间**：创建一个全新的空对象 `{}`。
2. **链接原型**：将这个空对象的 `__proto__` 指向 `Person.prototype`（这样新对象就能“借”原型上的方法了）。
3. **绑定 `this` 并执行**：执行函数，并将函数内部的 `this` 强行绑定到这个新对象上（所以 `this.name = name` 实际上是在给新对象装配属性）。
4. **自动返回**：如果函数没有显式返回其他对象，就自动返回这个新对象。

##### 没有`new`的时候

当直接 `const p = Person('A')` 时：

- **没有新对象产生**：它只是一个普通的函数运行。
- **`this` 乱跑**：在非严格模式下，`this` 指向 `window`。于是 `this.name = 'A'` 变成了 `window.name = 'A'`，你无意中污染了全局变量。
- **结果为空**：因为 `Person` 函数内部没有 `return` 语句，所以变量 `p` 的值是 `undefined`。

##### 实际应用中的“防错机制”

为了防止别人调用构造函数时忘记写 `new`，我们可以写一个“强迫症”逻辑：

```javascript
function SmartPerson(name) {
    // 判断 this 是不是 SmartPerson 的实例
    if (!(this instanceof SmartPerson)) {
        // 如果不是，说明你没写 new，那我就帮你 new 一个返回去
        return new SmartPerson(name);
    }
    this.name = name;
}

const p1 = SmartPerson('小明'); // 没写 new，但依然能工作！
```

##### 普通函数 vs 构造方法的“本质分工”

| **维度** | **普通函数 (Function)**                | **构造方法 (Constructor / Class)**     |
| -------- | -------------------------------------- | -------------------------------------- |
| **角色** | **加工厂的流水线**                     | **生产产品的模具**                     |
| **目的** | 执行一段逻辑，给一个输入，得一个输出。 | 创建一个持久存在的“对象个体”。         |
| **状态** | 随借随到，运行完变量通常就销毁了。     | 产生一个带有“记忆”（属性）的实体。     |
| **例子** | `calculate(a, b)`、`formatDate(date)`  | `new User('admin')`、`new WebSocket()` |

##### 为什么现在“普通函数”用得更多？

1. **React Hooks 的流行**：以前 React 强推 Class 组件（需要 `this` 和构造函数），现在推崇 Function Component。我们不再通过 `this.state` 存状态，而是通过 `useState`。
2. **函数式编程偏好**：现代开发倾向于“数据与逻辑分离”。数据就是简单的 JSON 对象，逻辑就是独立的工具函数，不再强行把它们揉在一个 Class 里。
3. **单例模式**：很多时候你的应用里某个功能（如：全局配置管理器）只需要一个，直接写个对象或函数就够了，不需要 `new` 很多个出来。

##### 深度思考：即便不写 `new`，也一直在用它

虽然你可能很少自己写 `function Person() {}`，但 JavaScript 的底层全是它们：

- 当写 `const arr = []` 时，底层其实是 `new Array()`。
- 当写 `const obj = {}` 时，底层其实是 `new Object()`。
- 当发送请求 `new Promise()` 或 `new Date()` 时

**这些内置工具之所以设计成构造方法，就是为了能同时开启多个倒计时、多个网络请求、多个数组，而它们互不干扰**

------

##### **手写new**

1. **创建一个新的空对象**。
2. **建立原型链**：将新对象的 `__proto__` 属性指向构造函数的 `prototype` 对象。
3. **绑定 this 并执行**：将构造函数的 `this` 绑定到这个新对象上，并执行构造函数。
4. **处理返回值**：如果构造函数返回的是一个**对象**，则返回该对象；否则，返回我们创建的新对象。

```javascript
function myNew(constructor, ...args) {
  // 1. 创建一个空的简单 JavaScript 对象
  const obj = {};

  // 2. 将该对象的原型指向构造函数的 prototype
  // 这样新对象就可以访问构造函数原型上的属性和方法
  Object.setPrototypeOf(obj, constructor.prototype);

  // 3. 将步骤 1 新创建的对象作为 this 的上下文，并传入参数执行构造函数
  const result = constructor.apply(obj, args);

  // 4. 如果构造函数返回的是对象，则返回该对象；否则返回我们创建的 obj
  return result instanceof Object ? result : obj;
}
//测试
function Person(name, age) {
  this.name = name;
  this.age = age;
}

Person.prototype.sayHi = function() {
  console.log(`你好，我是 ${this.name}`);
};

// 使用手写的 myNew
const p1 = myNew(Person, "小明", 20);

console.log(p1.name);     // "小明"
console.log(p1.age);      // 20
p1.sayHi();               // "你好，我是 小明"
console.log(p1.__proto__ === Person.prototype); // true
```

- **`Object.setPrototypeOf` vs `__proto__`**：虽然直接设置 `obj.__proto__` 也可以，但在现代开发中推荐使用 `Object.setPrototypeOf` 或 `Object.create(constructor.prototype)`，这样性能和规范性更好。

- **返回值判断**：JavaScript 的 `new` 比较特殊，如果构造函数显式返回了一个对象（比如 `return {a: 1}`），那么 `new` 出来的结果就是这个返回的对象，而不是实例本身。

  ------

##### new在构造函数返回不同类型时的表现

> “覆盖机制”

如果构造函数返回的是一个“对象（Object）”，则 `new` 的结果就是这个对象；否则（返回基础类型或没有返回值），`new` 的结果就是原本创建的实例，这也是在手写`new`时最后需要判断是否为对象的原因。

| **构造函数返回值类型**          | **new 的最终结果**     |
| ------------------------------- | ---------------------- |
| `undefined` (默认)              | 正常创建的实例对象     |
| `null`                          | 正常创建的实例对象     |
| `number` / `string` / `boolean` | 正常创建的实例对象     |
| **`Object` (如 `{}`)**          | **函数返回的那个对象** |
| **`Array` (如 `[]`)**           | **函数返回的那个数组** |
| **`Function`**                  | **函数返回的那个函数** |

`tips`：虽然 `typeof null` 是 `"object"`，但在 `new` 的逻辑里它被视为非对象）时，`new` 会**忽略**这些返回值。

------

**如果构造函数显式返回了一个新的对象，那么 `new` 出来的结果就是这个“新对象”。而这个新对象通常是直接通过 `{}` 或者 `new Object()` 创建的，它的原型指向的是 `Object.prototype`，而不是 `Person.prototype`。**

```javascript
function Person(name) {
  this.name = name;
  
  // 情况 A：显式返回一个全新的对象
  return {
    message: "我是一个乱入的对象"
  };
}

Person.prototype.sayHi = function() {
  console.log("你好！");
};

const p1 = new Person("小明");

console.log(p1.message); // "我是一个乱入的对象"
console.log(p1.name);    // undefined (Person 内部的 this 赋值失效了)

// 关键点：
p1.sayHi(); // ❌ Uncaught TypeError: p1.sayHi is not a function
```

**原因:**

1. 原本`new`准备了一个原型指向 `Person.prototype` 的对象
2. 执行构造函数时手动 `return` 了一个 `{ message: "..." }`。
3. **结果**：`new` 丢弃了之前准备好的 `对象`，转而把这个 `{ message: "..." }` 交给了 `p1`。
   - **`p1` 的原型链**：`p1` -> `Object.prototype` -> `null`
   - **`Person.prototype` 的位置**：它依然存在，但 `p1` 并没有链接到它上面。

------

如果非要让返回的对象也能调用 `sayHi` :

```javascript
function Person(name) {
  const result = { name: name };
  // 手动关联原型（不推荐这样写，违背了 new 的初衷）
  Object.setPrototypeOf(result, Person.prototype);
  return result;
}
```

**总结**:当构造函数返回对象时，**实例与构造函数原型之间的“自动链接”会失效**。这也是为什么在 JavaScript 模拟类继承时，必须非常小心处理构造函数返回值的缘故。

------

##### **ES6 Class** 中`constructor`返回对象时的表现和普通的 `function` 构造函数基本一致

在 ES6 `class` 中，`constructor` 的行为遵循与普通构造函数相同的逻辑：如果显式返回一个**对象**，则该对象会取代 `new` 本应生成的实例。

不过，由于 `class` 引入了**继承（extends）**，情况会变得稍微复杂一些，尤其是在处理 `super()` 的时候。

1. 基础表现：与普通函数一致

   在没有继承的情况下，`class` 的表现和 `function` 完全一样：

   ```javascript
   class Person {
     constructor(name) {
       this.name = name;
       return { greeting: "I am a spy!" }; // 显式返回对象
     }
   
     sayHi() {
       console.log("Hi!");
     }
   }
   
   const p = new Person("小明");
   console.log(p.name);     // undefined
   console.log(p.greeting); // "I am a spy!"
   console.log(p instanceof Person); // false (原型链断了)
   // p.sayHi();            // ❌ TypeError: p.sayHi is not a function
   ```

2. 继承中的“回马枪”：super() 的影响

   在子类的 `constructor` 中，规则依然适用，但你必须先处理 `super()`。

   根据 ES6 规范，子类的 `this` 是由父类生成的。如果你在子类构造函数中返回一个对象，那么这个对象将**彻底取代**父类生成的实例。

   ```javascript
   class Animal {}
   
   class Dog extends Animal {
     constructor() {
       super(); 
       return { bark: "woof" }; // 返回一个新对象
     }
   }
   
   const myDog = new Dog();
   console.log(myDog instanceof Dog);    // false
   console.log(myDog instanceof Animal); // false
   ```

   **有趣的一点：** 如果在子类中，你连 `super()` 都不调，但**直接返回一个对象**，代码**不会报错**。因为 `new` 最终拿到了一个合法的对象作为实例。

   ```javascript
   class Dog extends Animal {
     constructor() {
       // 没有 super()！
       return { bark: "woof" }; // 只要返回了对象，就不报错
     }
   }
   const d = new Dog(); // 正常运行
   ```

   **注意**：如果你不写 `return` 也不写 `super()`，JavaScript 就会报错，因为它发现 `this` 没被初始化。

   `super()`就是初始化

3. class 与 function 的细微差别

   虽然返回值逻辑一致，但 `class` 在语法层面有更严格的限制：

   | **特性**                        | **普通 Function**        | **ES6 Class**                     |
   | ------------------------------- | ------------------------ | --------------------------------- |
   | **显式返回基础类型** (如 `123`) | 忽略，返回实例           | 忽略，返回实例                    |
   | **显式返回 null**               | 忽略，返回实例           | 忽略，返回实例                    |
   | **不写 return**                 | 返回实例                 | 返回实例                          |
   | **调用限制**                    | 可以直接当作普通函数调用 | **必须**配合 `new` 调用，否则报错 |


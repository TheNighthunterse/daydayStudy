#### 原型与原型链

##### call、apply、bind

​	这三个方法的核心目的只有一个：**改变函数执行时的 `this` 指向**。

###### 	call` 和 `apply`：雷厉风行的“执行派”

- `call` 适合参数数量确定的场景。

- `apply` 适合参数数量不确定，或者参数本来就在数组里的场景。

```javascript
function greet(greeting, punctuation) {
    console.log(`${greeting}, I am ${this.name}${punctuation}`);
}

const user = { name: 'Gemini' };

// call: 逐个传入
greet.call(user, 'Hello', '!'); // 输出: Hello, I am Gemini!

// apply: 数组传入
greet.apply(user, ['Hi', '...']); // 输出: Hi, I am Gemini...
```

###### 		`bind`：运筹帷幄的“预谋派”

- `bind` 不会立即执行函数，而是返回一个**永久绑定**了特定 `this` 的新函数。它常用于回调函数（如 `setTimeout`）或 React 的事件处理中。

```javascript
const boundGreet = greet.bind(user, 'Hey'); 
// 此时不会有输出，它只是创建了一个新函数

boundGreet('?'); // 输出: Hey, I am Gemini?
```

######  应用场景:

- **借用原型方法**： 最经典的例子是将“类数组”（如 `arguments` 或 `NodeList`）转换成真正的数组。 `Array.prototype.slice.call(arguments)`

- **获取数组最值**： 利用 `apply` 可以接收数组的特性，配合 `Math.max`。 `Math.max.apply(null, [1, 5, 10, 2])` // 相当于 Math.max(1, 5, 10, 2)

- **配合闭包（如防抖）**： 在实现防抖（Debounce）或节流（Throttle）时，为了确保原函数的 `this` 指向正确，必须使用 `apply` 来传递上下文。

###### 面试题：如果 call/apply 的第一个参数传了 null 或 undefined 会怎样？

​	***在非严格模式下，`this` 会指向全局对象（浏览器里是 `window`，Node.js 里是 `global`）。在严格模式下，`this` 还是你传的那个 `null` 或 `undefined`***

###### 手写call

**思路：既然 `this` 总是指向调用函数的那个对象，那我们就把函数临时“挂载”到目标对象上**

实现方法：

1. **将函数设为对象的属性。**
2. **通过该对象调用这个函数（此时 `this` 自动指向该对象）**
3. **删除这个临时属性，并返回结果**

```javascript
Function.prototype.myCall = function(context, ...args) {
    // 1. 处理 context 为 null 或 undefined 的情况（指向 window）
    context = context || window;

    // 2. 使用 Symbol 创建一个唯一的键，防止覆盖对象原有的属性
    const fnSymbol = Symbol('fn');

    // 3. 这里的 this 就是我们要执行的那个函数
    // 将函数赋值给 context 的一个临时属性
    context[fnSymbol] = this;

    // 4. 执行函数并传入参数，获取返回值
    const result = context[fnSymbol](...args);

    // 5. 删除临时属性，深藏功与名
    delete context[fnSymbol];

    // 6. 返回函数执行的结果
    return result;
};

// --- 测试一下 ---
const person = { name: '小明' };
function sayHi(age, city) {
    console.log(`我是${this.name}，今年${age}岁，来自${city}`);
}

sayHi.myCall(person, 25, '杭州'); 
// 输出: 我是小明，今年25岁，来自杭州
```

- **`Symbol()` 的使用**：在老派的手写代码里，大家常用 `context.fn = this`。但如果 `context` 本来就有一个属性叫 `fn`，就会被覆盖。用 `Symbol` 可以保证这个临时属性是唯一的。
- **`...args` 剩余参数**：这是 ES6 的写法，非常简洁。如果要用 ES5，需要用 `arguments` 结合 `eval` 来构造参数字符串。
- **返回值处理**：记得要把函数执行的结果 `result` 存下来并返回，否则 `myCall` 永远返回 `undefined`
- **`apply` 的模拟**：和上面几乎一样，只是不需要 `...args`，直接接收一个数组参数即可。
- **`bind` 的模拟**：难度稍高，因为它需要返回一个**闭包**，并且要处理 `new` 调用的特殊情况。

##### 原型与原型链

1. ###### 三个核心概念（铁三角)

   **`prototype`**：**显式原型**。只有**函数**才有的属性。它定义了由该函数创建的实例共享的方法和属性。
   **`__proto__`**：**隐式原型**。每个**对象**（包括函数）都有的属性。它指向创建该对象的构造函数的 `prototype`。
   **`constructor`**：**构造函数**。原型对象上的一个属性，指向关联的构造函数。

2. ###### 什么是原型链？

   访问一个对象的属性时，先在**对象自身**找->顺着 **`__proto__`** 去它的**原型对象**找->**原型的原型**找->一直找到 **`Object.prototype`** 为止。如果最后还是找不到，就返回 `undefined`
   ***这种由 `__proto__` 层层连接起来的链路，就叫原型链。***

3. ###### 代码演示：直观感受链条

   ```javascript
   function Cat(name) {
       this.name = name;
   }
   
   // 在原型上加一个方法，所有猫都能用
   Cat.prototype.eat = function() {
       console.log(`${this.name} 正在吃小鱼干`);
   };
   
   const myCat = new Cat('咪咪');
   
   myCat.eat(); // 成功执行！
   
   // 查找过程分析：
   // 1. myCat 自身有 eat 吗？没有。
   // 2. 去 myCat.__proto__ (即 Cat.prototype) 找。
   // 3. 找到了！执行它。
   ```

4. ###### 原型链的尽头在哪里？原型链不是无限循环的，它有一个终点：

   `myCat` → `Cat.prototype` → `Object.prototype` → **`null`**

   > **知识点**：`Object.prototype` 是 JavaScript 中绝大多数对象的顶层原型。它的 `__proto__` 是 `null`，这代表了链条的终结。

5. 为什么会有这个设计？
   原型链的核心意义在于**属性共享**和**节省内存**。

   - **减少开销**：如果我有 100 只猫，我不需要在每只猫身上都存一遍 `eat` 方法。我只需要在 `Cat.prototype` 上存一份，所有猫共用即可。
   - **实现继承**：JavaScript 的继承不像 Java 那样通过类拷贝，而是通过原型链“顺藤摸瓜”来实现的。

6. 经典的面试题
   **Q：为什么 `Object.prototype.toString.call([])` 可以检测类型，而直接 `[].toString()` 不行？**
   **A：因为 `Array.prototype` 上重写了（Override）自己的 `toString` 方法，原型链查找到 `Array.prototype` 就停止了。而通过 `.call`，我们是强行跳过了数组的原型，直接去 `Object.prototype` 上调用了那个最原始的方法**

   **Q:万物皆对象，函数也是对象**

   **A：既然函数也是对象，那函数就有 `__proto__`**

   - `Robot.__proto__ === Function.prototype` (因为 Robot 是由 `new Function` 产生的)

   - **最绕的一点：** `Function.__proto__ === Function.prototype`

   - *解释：* 在 JS 的底层设计中，`Function` 既是构造函数也是自己的实例。

   **Q：原型链的终点**

   **A:`Object.prototype.__proto__ === null`**

7. 总结

   - **原型**：是一个对象，它是其他对象的“模板”。
   - **原型链**：是对象之间通过 `__proto__` 联系起来的路径。
   - **本质**：一种实现属性共享和继承的机制。

###### 常见的误区

- **误区一：所有对象都有 `prototype`。** **错！** 只有函数才有 `prototype`。普通对象只有 `__proto__`。
- **误区二：`__proto__` 是标准写法。** **不完全对。** 虽然主流浏览器都支持，但它最初是私有属性。现在标准建议使用 `Object.getPrototypeOf(obj)` 来获取原型。
- **误区三：修改 `prototype` 会影响已经创建的实例。** **对！** 因为实例只是通过指针（`__proto__`）引用了这个仓库。你往仓库里加东西，已经出生的孩子也能立马用到。
- **函数通过 `prototype` 存东西，实例通过 `__proto__` 找东西，而 `constructor` 则是证明身份的标签。**

```javascript
function Foo() {}
const f = new Foo();

console.log(f instanceof Foo);      // true (f 的原型链上有 Foo.prototype)
console.log(f instanceof Object);   // true (f 的原型链上也有 Object.prototype)
console.log(Foo instanceof Function); // true (函数本身是 Function 的实例)
```

##### ES6 Class —— 原型的“语法糖”

​	**ES5**

```javascript
function Animal(name) {
    this.name = name;
}
Animal.prototype.eat = function() { console.log('eating...'); };
```

​	**ES6**

```javascript
class Animal {
    constructor(name) {
        this.name = name;
    }
    eat() {
        console.log('eating...');
    }
}
```

**底层逻辑依然是原型**

| **维度**     | **原型（ES5）**            | **类（ES6 Class）**                  |
| ------------ | -------------------------- | ------------------------------------ |
| **定义方式** | 构造函数 + `prototype`     | `class` 关键字 + `constructor`       |
| **方法位置** | 手动挂载到 `prototype`     | 直接写在类块里（本质也是 prototype） |
| **继承实现** | 复杂的 `call` + 修改原型链 | 简单的 `extends` + `super()`         |
| **底层核心** | **原型链**                 | **原型链（换了个壳）**               |
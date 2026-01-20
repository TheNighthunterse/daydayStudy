#### Object.groupBy()允许我们按照特定属性将数组中的**对象分组**

```javascript
const pets = [
  { gender: 'man', name: '张三' },
  { gender: 'woman', name: '李四' },
  { gender: 'man', name: '王五' }
];

const res = Object.groupBy(pets, pet => pet.gender);
console.log(res);
// 输出:
// {
//   woman: [{ gender: '女', name: '李四' }]
//   man: [{ gender: 'man', name: '张三' }, { gender: 'man', name: '王五' }],
// }

```

#### 防抖

------



```javascript
function debounce(fn, delay) {
  let timer = null;

  // 这个返回的匿名函数，就是你在外部真正调用的“新函数”
  return function(...args) { 
    if (timer) clearTimeout(timer);
    
    timer = setTimeout(() => {
      // 核心行：fn.apply(this, args)
      fn.apply(this, args); 
    }, delay);
  };
}

// 调用示例
const handleSearch = debounce((text) => { console.log(text); }, 500);
handleSearch("Hello"); // 这里的 "Hello" 就是 text
```

#### 闭包

------

函数内部嵌套函数 外部函数定义的变量被内部使用的情况下内部函数被外面调用会导致变量被保存

形成条件：

**1.嵌套函数、**

**2.内部函数被传到了外面(通过 `return`、赋值给全局变量、或者传给定时器)**

**3.内部函数被存放在了某个地方**

```javascript
//return
function mother() {
  let food = "苹果";
  return function child() {
    console.log(food); 
  };
}

const eat = mother(); // child 函数“逃”到了外面，被 eat 变量接住了
// 此时，food 变量必须被保留，否则以后执行 eat() 时找不到苹果。


//赋值给外部变量
let globalChild;
function outer() {
  let data = "秘密";
  globalChild = function() { console.log(data); }; 
}
outer(); // 执行完后，data 依然活着，因为 globalChild 盯着它呢

//异步回调
function timerDemo() {
  let msg = "3秒后见";
  setTimeout(function() {
    console.log(msg); // 闭包：即使 timerDemo 运行完了，msg 也会保留到 3 秒后
  }, 3000);
}
timerDemo();

//事件监听
function setup() {
  let count = 0;
  document.getElementById('btn').onclick = function() {
    count++; // 只要按钮还在，count 就会一直保留在内存里
    console.log(count);
  };
}
```

##### 如果有多层嵌套：就近原则

```javascript
function grandFather() {
  let name = "爷爷";
  let treasure = "古董";

  return function father() {
    let name = "爸爸"; // 这里把爷爷的 name 覆盖了（遮蔽效应）
    
    return function child() {
      // child 访问了 father 的 name，访问了 grandFather 的 treasure
      console.log(`我是${name}，我继承了${treasure}`);
    };
  };
}

const action = grandFather()(); 
action(); // 输出: 我是爸爸，我继承了古董
```

##### 同级函数使用同一个变量：共享与影响

```javascript
function bankAccount() {
  let balance = 100; // 被共享的变量

  return {
    deposit: function(amount) {
      balance += amount;
      console.log(`存入后余额: ${balance}`);
    },
    withdraw: function(amount) {
      balance -= amount;
      console.log(`取款后余额: ${balance}`);
    }
  };
}

const myAccount = bankAccount();

myAccount.deposit(50);  // 输出: 150
myAccount.withdraw(20); // 输出: 130（它看到的是被 deposit 修改后的 150）
```

##### 在 for 循环里用 var 定义函数导致的闭包错误

```javascript
for (var i = 0; i < 3; i++) {
  setTimeout(function() {
    console.log(i);
  }, 1000);
}

// 实际结果：一秒后连着打印了三个 3
```

***解决方案：***

> ###### **利用闭包“锁住”参数（老派做法）**

```javascript
for (var i = 0; i < 3; i++) {
  // 通过 IIFE 创建一个独立的小房间
  (function(currentI) {
    setTimeout(function() {
      console.log(currentI); // 这个闭包引用的不再是那个全局的 i，而是小房间里的参数 currentI
    }, 1000);
  })(i); // 把当前的 i 传给 currentI
}
// 结果：0, 1, 2
```

**原理：** 每次循环都执行了一个新函数，产生了一个新的闭包环境。每个环境里的 `currentI` 都是独立的快照（0, 1, 2）。

> ###### **使用 `let`（现代标准做法）**

```javascript
for (let i = 0; i < 3; i++) {
  setTimeout(function() {
    console.log(i);
  }, 1000);
}
// 结果：0, 1, 2
```

**原理（非常精彩）：**

- `let` 具有 **块级作用域**。
- 在 `for` 循环中使用 `let` 时，JavaScript 引擎在底层不仅仅是声明了一个变量。实际上，**每一次循环迭代，都会产生一个新的 `i` 变量**。
- 每一个 `setTimeout` 的闭包都捕捉到了属于它那一轮循环的那个独立的 `i`。

> ######  **一个有趣的变体：如果用 `setTimeout` 的第三个参数**

```javascript
for (var i = 0; i < 3; i++) {
  // 第三个参数会被作为参数传给第一个回调函数
  setTimeout(function(val) {
    console.log(val);
  }, 1000, i); 
}
```

###### 总结

   		这个问题的核心教训是：**闭包引用的外部变量是动态的，除非你通过函数参数传参或者使用块级作用域（let），否则闭包只会看到变量最终的状态**
















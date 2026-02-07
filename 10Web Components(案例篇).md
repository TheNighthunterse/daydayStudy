### 基本写法

#### 核心骨架：HTML 模板

使用 `<template>` 标签。它的特点是：**里面的内容在被 JS 激活前不会被渲染，且不占资源（图片不会预加载）**

```html
<template id="my-card-template">
  <style>
    /* 这里的样式是隔离的，不会影响外部 */
    .card {
      border: 1px solid #ddd;
      border-radius: 8px;
      padding: 16px;
      width: 200px;
      transition: box-shadow 0.3s;
    }
    .card:hover { box-shadow: 0 4px 12px rgba(0,0,0,0.1); }
    ::slotted(h2) { color: #007aff; } /* 专门为 slot 传入的元素设置样式 */
  </style>

  <div class="card">
    <slot name="title">默认标题</slot>
    <hr>
    <slot name="content">默认内容...</slot>
    <button id="btn">点击交互</button>
  </div>
</template>
```

------

#### 逻辑封装：自定义类

写一个 JS 类来继承原生的 `HTMLElement`，这是组件的大脑

```javascript
class MyCard extends HTMLElement {
  constructor() {
    super();
    // 1. 开启影子 DOM
    const shadow = this.attachShadow({ mode: 'open' });

    // 2. 获取并克隆模板内容
    const template = document.getElementById('my-card-template');
    const content = template.content.cloneNode(true);

    // 3. 将克隆的内容挂载到影子 DOM 上
    shadow.appendChild(content);
  }

  // 相当于 Vue 的 mounted 或 React 的 useEffect
  connectedCallback() {
    console.log('组件已挂载');
    this.shadowRoot.querySelector('#btn').addEventListener('click', () => {
      alert('原生组件响应成功！');
    });
  }

  // 监听属性变化,必须声明你要监听哪些属性，否则下面的回调不会触发
  static get observedAttributes() {
    return ['data-color'];
  }
  // 当属性变化时，浏览器自动调用这个方法
  attributeChangedCallback(name, oldValue, newValue) {
    if (name === 'data-color') {
      this.shadowRoot.querySelector('.card').style.borderColor = newValue;
    }
  }
}

// 4. 注册组件
customElements.define('my-card', MyCard);
```

在 `class MyComponent extends HTMLElement` 中：

- **在 `constructor` 中：** `this` 代表“即将诞生”的组件。你可以给它装载影子 DOM（`this.attachShadow`）。

- **在方法中：** `this` 就是这个 **DOM 节点本身**。这意味着你可以像操作原生 `div` 一样操作 `this`。

  - `this.style.color = 'red'` (改颜色)
  - `this.getAttribute('name')` (拿属性)
  - `this.innerHTML` (换内容)

- **四大生命周期**

  | **钩子函数**                     | **触发时机**          | **相当于 Vue / React**         | **常见用途**                  |
  | -------------------------------- | --------------------- | ------------------------------ | ----------------------------- |
  | **`constructor()`**              | 实例被创建时          | `setup` / `constructor`        | 初始化数据、创建 Shadow DOM。 |
  | **`connectedCallback()`**        | 组件**插入**到 DOM 时 | `mounted` / `useEffect`        | 发起网络请求、绑定全局事件。  |
  | **`disconnectedCallback()`**     | 组件从 DOM **移除**时 | `unmounted`                    | 清除定时器、解绑事件。        |
  | **`attributeChangedCallback()`** | 监听的**属性变化**时  | `watch` / `componentDidUpdate` | 根据参数变化更新视图。        |

------

#### 使用组件

```html
<my-card data-color="blue">
  <h2 slot="title">哈罗德</h2>
  <p slot="content">这是一个完全基于原生 Web Components 编写的组件。</p>
</my-card>
```

------

#### 深度解析：

##### A. Shadow DOM 的边界感

在 `constructor` 里执行 `this.attachShadow({ mode: 'open' })` 之后，组件就拥有了一个独立的“沙盒”。

- **外部 CSS 进不来：** 页面上的 `div { background: red; }` 不会影响组件。
- **内部 CSS 出不去：** 组件里的样式也不会污染全局。

##### B. 声明式 Slot（插槽）

`::slotted` 选择器非常神奇。它允许组件开发者为用户传进来的 HTML 预设样式，但同时保留了内容的灵活性。

##### C. 生命周期钩子

原生 API 提供了完整的生命周期，足以覆盖大多数逻辑场景：

- `connectedCallback`: 插入 DOM 时调用（初始化事件、拉取数据）。
- `disconnectedCallback`: 从 DOM 移除时调用（清除定时器、解绑事件）。
- `attributeChangedCallback`: 属性变化时调用（实现响应式更新）

------

### 模拟一个 Element 风格的对话框（Dialog）

```html
<!doctype html>
<html lang="zh-CN">
  <head>
    <meta charset="UTF-8" />
    <title>浏览器系统 API 探测器</title>
    <style></style>
  </head>
  <body>
    <template id="dialog-template">
      <style>
        /* 遮罩层：全屏固定，半透明背景 */
        .overlay {
          position: fixed;
          top: 0;
          left: 0;
          width: 100%;
          height: 100%;
          background: rgba(0, 0, 0, 0.5);
          display: flex;
          align-items: center; /* 垂直居中 */
          justify-content: center; /* 水平居中 */
          opacity: 0;
          visibility: hidden;
          transition: all 0.3s;
          z-index: 2000;
        }

        /* 激活状态 */
        :host([visible]) .overlay {
          opacity: 1;
          visibility: visible;
        }

        /* 对话框主体 */
        .dialog {
          background: #fff;
          border-radius: 4px;
          box-shadow: 0 1px 3px rgba(0, 0, 0, 0.3);
          width: 50%;
          max-width: 600px;
          transform: translateY(-20px);
          transition: transform 0.3s;
        }

        :host([visible]) .dialog {
          transform: translateY(0);
        }

        /* 头部、主体、尾部布局 */
        .header {
          padding: 20px 20px 10px;
          display: flex;
          justify-content: space-between;
        }
        .title {
          font-size: 18px;
          color: #303133;
        }
        .close-btn {
          cursor: pointer;
          color: #909399;
          font-size: 20px;
          border: none;
          background: none;
        }
        .close-btn:hover {
          color: #409eff;
        }

        .body {
          padding: 30px 20px;
          color: #606266;
          font-size: 14px;
          word-break: break-all;
        }

        .footer {
          padding: 10px 20px 20px;
          text-align: right;
          box-sizing: border-box;
        }

        /* 模拟 Element 按钮样式 */
        ::slotted(button) {
          padding: 9px 15px;
          font-size: 12px;
          border-radius: 3px;
          cursor: pointer;
          border: 1px solid #dcdfe6;
          margin-left: 10px;
        }
      </style>

      <div class="overlay">
        <div class="dialog">
          <div class="header">
            <span class="title"><slot name="title">提示</slot></span>
            <button class="close-btn">&times;</button>
          </div>
          <div class="body">
            <slot>这是对话框内容</slot>
          </div>
          <div class="footer">
            <slot name="footer"></slot>
          </div>
        </div>
      </div>
    </template>
    <button onclick="document.querySelector('x-dialog').setAttribute('visible', '')">打开原生对话框</button>

    <x-dialog>
      <span slot="title">删除确认</span>
      <div>确认要删除这条重要数据吗？此操作不可撤销。</div>
      <div slot="footer">
        <button onclick="document.querySelector('x-dialog').removeAttribute('visible')">取消</button>
        <button style="background: #409eff; color: #fff; border: none">确定</button>
      </div>
    </x-dialog>
    <script>
      class XDialog extends HTMLElement {
        constructor() {
          super();
          this.attachShadow({ mode: 'open' });
          const template = document.getElementById('dialog-template');
          this.shadowRoot.appendChild(template.content.cloneNode(true));
        }

        connectedCallback() {
          // 点击关闭按钮
          this.shadowRoot.querySelector('.close-btn').onclick = () => this.handleClose();
          // 点击遮罩层关闭
          this.shadowRoot.querySelector('.overlay').onclick = (e) => {
            if (e.target.className === 'overlay') this.handleClose();
          };
        }

        // 监听 visible 属性
        static get observedAttributes() {
          return ['visible'];
        }

        attributeChangedCallback(name, oldVal, newVal) {
          if (name === 'visible') {
            // 可以在这里触发 open/close 事件
            const eventName = newVal !== null ? 'open' : 'close';
            this.dispatchEvent(new CustomEvent(eventName));
          }
        }

        handleClose() {
          this.removeAttribute('visible'); // 触发隐藏动画
        }
      }

      customElements.define('x-dialog', XDialog);
    </script>
  </body>
</html>

```

#### 通过案例体现web components的特点、

1. **真正的 CSS 隔离：** 你在外面给 `button` 写的全局样式，绝对不会影响对话框里的 `close-btn`。这种稳定性是大型项目最需要的。
2. **声明式交互：** 看到那两个 `slot` 了吗？这和 Vue 的插槽几乎一模一样。浏览器原生支持这种“内容投影”，让组件的使用者可以自由定义对话框的头和尾。
3. **无框架依赖：** 这段代码放在任何 HTML 页面都能跑，甚至是 20 年前的老网页（只要浏览器更新到现代版本），这便是**原生组件**的生命力。
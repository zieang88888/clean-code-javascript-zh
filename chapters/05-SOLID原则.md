# SOLID 原则（5 条规范）

> SOLID 是面向对象设计的五条核心原则，缩写自 SRP / OCP / LSP / ISP / DIP。它们共同回答一个问题：怎么把类切小、切对，让代码好扩展、好替换、好维护。本章逐条给出可落地的判断标准。

## 单一职责原则（Single Responsibility Principle，SRP）

《Clean Code》里那句话："一个类存在的修改理由不应该超过一个。"给一个类塞太多功能，就像出差只带一个行李箱——全塞进去后概念上不再内聚，改任何一处都可能波及其它依赖模块。落地方法：按"变更原因"拆类，认证归认证、设置归设置，类之间通过组合协作。

❌ **坏例子**

```javascript
class UserSettings {
  constructor(user) {
    this.user = user;
  }

  changeSettings(settings) {
    if (this.verifyCredentials()) {
      // ...
    }
  }

  verifyCredentials() {
    // ...
  }
}
```

✅ **好例子**

```javascript
class UserAuth {
  constructor(user) {
    this.user = user;
  }

  verifyCredentials() {
    // ...
  }
}

class UserSettings {
  constructor(user) {
    this.user = user;
    this.auth = new UserAuth(user);
  }

  changeSettings(settings) {
    if (this.auth.verifyCredentials()) {
      // ...
    }
  }
}
```

> 💡 学习提示：问自己"这个类为什么要改？"——如果答案能列出两三个互不相关的理由（改字段校验、改登录逻辑、改存储方式），就说明该拆了。

## 开闭原则（Open/Closed Principle，OCP）

Bertrand Meyer 提出："软件实体应对扩展开放，对修改关闭。"意思是加新功能时应该是新增代码（新类、新实现），而不是回头去改老代码里的 `if/else` 分支。落地手段是面向抽象编程：让依赖方只认抽象约定，具体实现各写各的，新增一种适配器就新增一个类。

❌ **坏例子**

```javascript
class AjaxAdapter extends Adapter {
  constructor() {
    super();
    this.name = "ajaxAdapter";
  }
}

class NodeAdapter extends Adapter {
  constructor() {
    super();
    this.name = "nodeAdapter";
  }
}

class HttpRequester {
  constructor(adapter) {
    this.adapter = adapter;
  }

  fetch(url) {
    if (this.adapter.name === "ajaxAdapter") {
      return makeAjaxCall(url).then(response => {
        // transform response and return
      });
    } else if (this.adapter.name === "nodeAdapter") {
      return makeHttpCall(url).then(response => {
        // transform response and return
      });
    }
  }
}

function makeAjaxCall(url) {
  // request and return promise
}

function makeHttpCall(url) {
  // request and return promise
}
```

✅ **好例子**

```javascript
class AjaxAdapter extends Adapter {
  constructor() {
    super();
    this.name = "ajaxAdapter";
  }

  request(url) {
    // request and return promise
  }
}

class NodeAdapter extends Adapter {
  constructor() {
    super();
    this.name = "nodeAdapter";
  }

  request(url) {
    // request and return promise
  }
}

class HttpRequester {
  constructor(adapter) {
    this.adapter = adapter;
  }

  fetch(url) {
    return this.adapter.request(url).then(response => {
      // transform response and return
    });
  }
}
```

> 💡 学习提示：一旦在业务代码里看到 `if (xxx.name === "A") ... else if (xxx.name === "B")` 这种按类型分支，多半就是 OCP 在报警——把分支收进每个实现类自己的 `request` 方法里。

## 里氏替换原则（Liskov Substitution Principle，LSP）

名字吓人，意思很朴素：子类必须能在父类出现的任何地方原样替换，而不破坏程序的正确性。经典反例是"正方形继承长方形"——数学上正方形是长方形，但一旦让 `Square extends Rectangle`，`setWidth` / `setHeight` 就得互相牵连，外部按长方形的用法调用就会算出错误面积。正确做法是让两者共同继承一个更抽象的 `Shape`。

❌ **坏例子**

```javascript
class Rectangle {
  constructor() {
    this.width = 0;
    this.height = 0;
  }

  setColor(color) {
    // ...
  }

  render(area) {
    // ...
  }

  setWidth(width) {
    this.width = width;
  }

  setHeight(height) {
    this.height = height;
  }

  getArea() {
    return this.width * this.height;
  }
}

class Square extends Rectangle {
  setWidth(width) {
    this.width = width;
    this.height = width;
  }

  setHeight(height) {
    this.width = height;
    this.height = height;
  }
}

function renderLargeRectangles(rectangles) {
  rectangles.forEach(rectangle => {
    rectangle.setWidth(4);
    rectangle.setHeight(5);
    const area = rectangle.getArea(); // BAD: Returns 25 for Square. Should be 20.
    rectangle.render(area);
  });
}

const rectangles = [new Rectangle(), new Rectangle(), new Square()];
renderLargeRectangles(rectangles);
```

✅ **好例子**

```javascript
class Shape {
  setColor(color) {
    // ...
  }

  render(area) {
    // ...
  }
}

class Rectangle extends Shape {
  constructor(width, height) {
    super();
    this.width = width;
    this.height = height;
  }

  getArea() {
    return this.width * this.height;
  }
}

class Square extends Shape {
  constructor(length) {
    super();
    this.length = length;
  }

  getArea() {
    return this.length * this.length;
  }
}

function renderLargeShapes(shapes) {
  shapes.forEach(shape => {
    const area = shape.getArea();
    shape.render(area);
  });
}

const shapes = [new Rectangle(4, 5), new Rectangle(4, 5), new Square(5)];
renderLargeShapes(shapes);
```

> 💡 学习提示：子类方法的行为契约必须"不弱于"父类——可以强化前置条件，但不能削弱后置条件。否则一旦把子类塞回父类列表里跑，结果就会悄悄出错。

## 接口隔离原则（Interface Segregation Principle，ISP）

"不应该强迫客户端依赖它用不到的方法。"JS 没有显式接口，但鸭子类型下接口是隐式约定。典型场景是大而全的配置对象：调用方明明只想遍历 DOM，却被迫要传一个动画模块进来。把配置拆细、把可选项做成"需要才配置"，就能避免肥大接口。

❌ **坏例子**

```javascript
class DOMTraverser {
  constructor(settings) {
    this.settings = settings;
    this.setup();
  }

  setup() {
    this.rootNode = this.settings.rootNode;
    this.settings.animationModule.setup();
  }

  traverse() {
    // ...
  }
}

const $ = new DOMTraverser({
  rootNode: document.getElementsByTagName("body"),
  animationModule() {} // Most of the time, we won't need to animate when traversing.
  // ...
});
```

✅ **好例子**

```javascript
class DOMTraverser {
  constructor(settings) {
    this.settings = settings;
    this.options = settings.options;
    this.setup();
  }

  setup() {
    this.rootNode = this.settings.rootNode;
    this.setupOptions();
  }

  setupOptions() {
    if (this.options.animationModule) {
      // ...
    }
  }

  traverse() {
    // ...
  }
}

const $ = new DOMTraverser({
  rootNode: document.getElementsByTagName("body"),
  options: {
    animationModule() {}
  }
});
```

> 💡 学习提示：看构造函数要的参数列表——如果客户端经常要传 `undefined` 或空函数占位，说明接口太胖，该按使用场景拆成多个小配置。

## 依赖倒置原则（Dependency Inversion Principle，DIP）

两句话：高层模块不该依赖低层模块，二者都该依赖抽象；抽象不该依赖细节，细节该依赖抽象。落到 JS 里就是——别在类内部 `new` 具体实现，而是把实现作为参数从外面注入进来（依赖注入）。这样换一种请求方式（HTTP 换 WebSocket）只需在外部换一个实例，业务代码一行都不用改。

❌ **坏例子**

```javascript
class InventoryRequester {
  constructor() {
    this.REQ_METHODS = ["HTTP"];
  }

  requestItem(item) {
    // ...
  }
}

class InventoryTracker {
  constructor(items) {
    this.items = items;

    // BAD: We have created a dependency on a specific request implementation.
    // We should just have requestItems depend on a request method: `request`
    this.requester = new InventoryRequester();
  }

  requestItems() {
    this.items.forEach(item => {
      this.requester.requestItem(item);
    });
  }
}

const inventoryTracker = new InventoryTracker(["apples", "bananas"]);
inventoryTracker.requestItems();
```

✅ **好例子**

```javascript
class InventoryTracker {
  constructor(items, requester) {
    this.items = items;
    this.requester = requester;
  }

  requestItems() {
    this.items.forEach(item => {
      this.requester.requestItem(item);
    });
  }
}

class InventoryRequesterV1 {
  constructor() {
    this.REQ_METHODS = ["HTTP"];
  }

  requestItem(item) {
    // ...
  }
}

class InventoryRequesterV2 {
  constructor() {
    this.REQ_METHODS = ["WS"];
  }

  requestItem(item) {
    // ...
  }
}

// By constructing our dependencies externally and injecting them, we can easily
// substitute our request module for a fancy new one that uses WebSockets.
const inventoryTracker = new InventoryTracker(
  ["apples", "bananas"],
  new InventoryRequesterV2()
);
inventoryTracker.requestItems();
```

> 💡 学习提示：五原则记忆口诀——"一个类一件事（SRP）、加功能不改老代码（OCP）、子类随时替换父类（LSP）、接口小而专（ISP）、高层依赖抽象不依赖细节（DIP）"。

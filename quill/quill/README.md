# Quill源码分析

`Quill`一开始就注册了内部定义的`blots`, `formats` ,`modules`和`ui`并存放在实例的`imports`内；其中`blots`和`formats`会调用`Parchment.register`再注册一遍。

## 自定义主题

自定义主题其实也是自定义工具栏，虽然`Quill`提供了`snow`和`bubble`这2种主题，但它们附带的工具栏样式并不能满足特殊业务的要求，需要自己重新定义一套，需要重新开发`toolbar`以及还未提供的`formats`样式

##### 1. 主题初始化

在`Quill`的构造函数中，通过`expandConfig` 函数，获取之前注册的主题类，并将主题类中定义的modules的默认配置项跟`userConfig`(为主)合并，最后实例化主题并调用其`init`函数。

```js
// quill 构造函数
constructor(container, options = {}) {
    this.options = expandConfig(container, options);
    // ...
    this.theme = new this.options.theme(this, this.options);
    // ...
    this.theme.init();
    // ...
}

function expandConfig(container, userConfig) {
    // ...
    if (!userConfig.theme || userConfig.theme === Quill.DEFAULTS.theme) {
      userConfig.theme = Theme;
    } else {
      userConfig.theme = Quill.import(`themes/${userConfig.theme}`);
      if (userConfig.theme == null) {
        throw new Error(`Invalid theme ${userConfig.theme}. Did you register it?`);
      }
    }
    let themeConfig = extend(true, {}, userConfig.theme.DEFAULTS);
    // ...
    let moduleNames = Object.keys(themeConfig.modules).concat(Object.keys(userConfig.modules));
    let moduleConfig = moduleNames.reduce(function(config, name) {
        let moduleClass = Quill.import(`modules/${name}`);
        if (moduleClass == null) {
          debug.error(`Cannot load ${name} module. Are you sure you registered it?`);
        } else {
          config[name] = moduleClass.DEFAULTS || {};
        }
        return config;
    }, {});
    // ...
}
```

##### 2. 自定义主题

主题的基类`Theme`其实很简单，`init`遍历上面配置项内的`modules`，`addModule`对其进行实例化并收集实例。

```js
class Theme {
  constructor(quill, options) {
    this.quill = quill;
    this.options = options;
    this.modules = {};
  }

  init() {
    Object.keys(this.options.modules).forEach((name) => {
      if (this.modules[name] == null) {
        this.addModule(name);
      }
    });
  }

  addModule(name) {
    let moduleClass = this.quill.constructor.import(`modules/${name}`);
    this.modules[name] = new moduleClass(this.quill, this.options.modules[name] || {});
    return this.modules[name];
  }
}
Theme.DEFAULTS = {
  modules: {}
};
Theme.themes = {
  'default': Theme
};


export default Theme;
```

`BubbleTheme`中对`toolbar`和`tooltip`分别进行了各个按钮的事件监听和`tooltip`和`picker`浮层的显隐切换键控制

```js
// 已/theme/base.js 和 /theme/bubble.js为例
class BaseTheme extends Theme {
  constructor(quill, options) {
    super(quill, options);
    let listener = (e) => {
      // ...
    };
    quill.emitter.listenDOM('click', document.body, listener);
  }
  addModule(name) {
    let module = super.addModule(name);
    if (name === 'toolbar') {
      this.extendToolbar(module);
    }
    return module;
  }
}

class BubbleTheme extends BaseTheme {
  constructor(quill, options) {
    if (options.modules.toolbar != null && options.modules.toolbar.container == null) {
      options.modules.toolbar.container = TOOLBAR_CONFIG;
    }
    super(quill, options);
    this.quill.container.classList.add('ql-bubble');
  }

  extendToolbar(toolbar) {
    this.tooltip = new BubbleTooltip(this.quill, this.options.bounds);
    this.tooltip.root.appendChild(toolbar.container);
    this.buildButtons([].slice.call(toolbar.container.querySelectorAll('button')), icons);
    this.buildPickers([].slice.call(toolbar.container.querySelectorAll('select')), icons);
  }
}
```

##### 自定义Toolbar

## Blot的生命周期

##### Creation

所有类型的`Blot`都有一个`static create`方法，接收初始值`value`，根据`Blot`的`tagName`和`className`创建对应的`DOM Node`。特殊的`Blot`可以重写`statis create`方法，最后要返回`node`。

```js
// formats/image.js
static create(value) {
  let node = super.create(value);
  if (typeof value === 'string') {
    node.setAttribute('src', this.sanitize(value));
  }
  return node;
}
```

`statis create`通常是在`insertAt` 、`insertBefore`、`replace`等插入和替换操作时，通过`Parchment.create`调用，同时返回实例化后的`Blot`对象。

```js
export function create(input: Node | string | Scope, value?: any): Blot {
  let match = query(input);
  if (match == null) {
    throw new ParchmentError(`Unable to create ${input} blot`);
  }
  let BlotClass = <BlotConstructor>match;
  let node =
    // @ts-ignore
    input instanceof Node || input['nodeType'] === Node.TEXT_NODE 
      ? input 
      : BlotClass.create(value); /* 调用静态方法 */ 
  return new BlotClass(<Node>node, value); /* 返回实例 */ 
}
```

##### Attach

`Parhcment.create`执行完后，需要将其挂载到Quill Tree 和 DOM Tree上。无论是`insertAt`还是`replace`,最后都会回到起父容器中执行`insertBefore`,  该方法会执行所有`child blot`的`insertInto`方法。在`insertInto`中会更新Quill Tree和DOM Tree

```js
// Parchment  /blot/abstract/shadow
insertInto(parentBlot: Parent, refBlot: Blot | null = null): void {
  if (this.parent != null) {
    this.parent.children.remove(this);
  }
  let refDomNode: Node | null = null;
  parentBlot.children.insertBefore(this, refBlot);
  if (refBlot != null) {
    refDomNode = refBlot.domNode;
  }
  if (this.domNode.parentNode != parentBlot.domNode ||
      this.domNode.nextSibling != refDomNode) {
    parentBlot.domNode.insertBefore(this.domNode, refDomNode);
  }
  this.parent = parentBlot;
  /* 执行对应blot的attach方法，如果当前blot为containerblot,
   * 会执行它所有子blot的attach方法
   */
  this.attach(); 
}
```

##### Update & Optimization

`ScrollBlot`是顶层的`ContainerBlot`，它是在`quill`初始化时实例化的，所有的`blot`都在这个容器内，内部通过`MutationObserver`监听了整个编辑器的`DOM`变化，并执行容器内所有`Blot`的attach方法。

```js
// Parchment  /blot/scroll
constructor(node: HTMLDivElement) {
  super(node);
  this.scroll = this;
  this.observer = new MutationObserver((mutations: MutationRecord[]) => {
    this.update(mutations);
  });
  this.observer.observe(this.domNode, OBSERVER_CONFIG);
  this.attach();
}
```

DOM树发生变化，触发ScrollBlot中的`update`方法，调用`mutaion`对应的`blot.update`，一是`FormatBlot.update`，会将样式重新创建；二是`ContainerBlot.update`，移除无父节点或父节点不是该容器的`Blot`，依次插入父节点为当前容器的节点。

```js
// Parchment  /blot/abstract/format
update(mutations: MutationRecord[], context: { [key: string]: any }): void {
  super.update(mutations, context);
  if (
    mutations.some(mutation => {
      return mutation.target === this.domNode && mutation.type === 'attributes';
    })
  ) {
    this.attributes.build();
  }
}
```

```js
// Parchment  /blot/abstract/container
update(mutations: MutationRecord[], context: { [key: string]: any }): void {
  // ...
  removedNodes.forEach((node: Node) => {
    // Check node has actually been removed
    // One exception is Chrome does not immediately remove IFRAMEs
    // from DOM but MutationRecord is correct in its reported removal
    // ...
    if (blot.domNode.parentNode == null || blot.domNode.parentNode === this.domNode) {
      blot.detach();
    }
  });
  addedNodes
    .filter(node => {
      return node.parentNode == this.domNode;
    })
    .sort(function(a, b) {
     // ...
    })
    .forEach(node => {
        //...
        this.insertBefore(blot, refBlot || undefined);
    });
}
```

最后执行ScrollBlot的`optimize`，会执行所有child的`optimize`，这个方法是在`update`后，对`Quill Tree` 和 `DOM Tree`进行的优化。以`ContainerBlot`和`InlineBlot`为例

```js
// Parchment  /blot/abstract/container
// 空值处理
// 如果children为空，设置了默认子节点就重新插入子节点，没有就移除整个ContainerBlot
optimize(context: { [key: string]: any }) {
  super.optimize(context);
  if (this.children.length === 0) {
    if (this.statics.defaultChild != null) {
      let child = Registry.create(this.statics.defaultChild);
      this.appendChild(child);
      child.optimize(context);
    } else {
      this.remove();
    }
  }
}
```

```js
// Parchment  /blot/inline
// InlineBlot 优化处理
// 如果没有样式，就去掉包裹，内容移到父层去
// 如果后面节点也是InlineBlot,而且样式一样，就合并为一个
optimize(context: { [key: string]: any }): void {
  super.optimize(context);
  let formats = this.formats();
  if (Object.keys(formats).length === 0) {
    return this.unwrap(); // unformatted span
  }
  let next = this.next;
  if (next instanceof InlineBlot && next.prev === this && isEqual(formats, next.formats())) {
    next.moveChildren(this);
    next.remove();
  }
}
```

##### Deletion and Detachment

`remove`最后都会调用这个方法，来删除`Quill Tree`和`DOM Tree`中的该节点

```js
remove(): void {
  if (this.domNode.parentNode != null) {
    this.domNode.parentNode.removeChild(this.domNode);
  }
  this.detach();
}
detach() {
  if (this.parent != null) this.parent.removeChild(this);
  // @ts-ignore
  delete this.domNode[Registry.DATA_KEY];
}
```

`removeChild` 移除`QuillTree`中的该节点

```js
// this.children 是 LinkList 类型
removeChild(child: Blot): void {
  this.children.remove(child);
}
```

`deleteAt`删除特定位置的节点

```js
deleteAt(index: number, length: number): void {
  let blot = this.isolate(index, length);
  blot.remove();
}
```

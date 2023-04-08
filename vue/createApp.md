# createApp
`createApp`是我们一开始就要调用的方法，通过该方法创建我们的`app`实例;
```js
// example
import App from './app.vue';
const app = createApp(App);

// app.use()

app.mount('#app');
```

```js
// 路径：packages/runtime-dom/src/index.ts createApp
export const createApp = ((...args) => {
  // 渲染方法
  const app = ensureRenderer().createApp(...args)

  if (__DEV__) {
    injectNativeTagCheck(app)
    injectCompilerOptionsCheck(app)
  }

  const { mount } = app
  app.mount = (containerOrSelector: Element | ShadowRoot | string): any => {
    // normalizeContainer(container: Element | ShadowRoot | string): Element | null
    // 返回实例的挂载节点
    const container = normalizeContainer(containerOrSelector)
    if (!container) return
    // 略
    container.innerHTML = ''
    const proxy = mount(container, false, container instanceof SVGElement)
    if (container instanceof Element) {
      container.removeAttribute('v-cloak')
      container.setAttribute('data-v-app', '')
    }
    return proxy
  }

  return app
}) as CreateAppFunction<Element>

// 路径：packages/runtime-dom/src/index.ts ensureRenderer

function ensureRenderer() {
  return (
    renderer ||
    (renderer = createRenderer<Node, Element | ShadowRoot>(rendererOptions))
  )
}
```

#### createRenderer
```js
// 路径：packages/runtime-dom/src/renderer.ts createRenderer
export function createRenderer<
  HostNode = RendererNode,
  HostElement = RendererElement
>(options: RendererOptions<HostNode, HostElement>) {
  return baseCreateRenderer<HostNode, HostElement>(options)
}
```

```js
// 路径：packages/runtime-core/src/renderer.ts baseCreateRenderer 
// 这个方法有2000行左右，后面在render板块中单独分析
function baseCreateRenderer(
  options: RendererOptions,
  createHydrationFns?: typeof createHydrationFunctions
): any {
  // ...

   const render: RootRenderFunction = (vnode, container, isSVG) => {
    if (vnode == null) {
      if (container._vnode) {
        unmount(container._vnode, null, null, true)
      }
    } else {
      // 更新dom节点
      patch(container._vnode || null, vnode, container, null, null, null, isSVG)
    }
    // 执行预处理回调函数
    flushPreFlushCbs()
    // 执行完DOM 更新完成之后执行一些副作用代码
    flushPostFlushCbs()
    container._vnode = vnode
  }

  return {
    render,
    hydrate,
    createApp: createAppAPI(render, hydrate)
  }
}
```

#### createAppAPI
```js
// packages/runtime-core/src/apiCreateApp.ts createAppAPI 
export function createAppAPI<HostElement>(
  render: RootRenderFunction<HostElement>,
  hydrate?: RootHydrateFunction
): CreateAppFunction<HostElement> {
  return function createApp(rootComponent, rootProps = null) {
    // ...
    const context = createAppContext()
    const installedPlugins = new Set()
    
    isMounted = false;

    const app: App = (context.app = {
      // ...
      use(plugin: Plugin, ...options: any[]) {}
      mixin(mixin: ComponentOptions) {}
      component(name: string, component?: Component): any {}
      directive(name: string, directive?: Directive) {}

      mount(
        rootContainer: HostElement,
        isHydrate?: boolean,
        isSVG?: boolean
      ): any {
        if (!isMounted) {
          // #5571
          if (__DEV__ && (rootContainer as any).__vue_app__) {
            // warn 略
          }
          const vnode = createVNode(
            rootComponent as ConcreteComponent,
            rootProps
          )
          // store app context on the root VNode.
          // this will be set on the root instance on initial mount.
          vnode.appContext = context

          // HMR root reload
          if (__DEV__) {
            context.reload = () => {
              render(cloneVNode(vnode), rootContainer, isSVG)
            }
          }

          // 是否是ssr渲染
          if (isHydrate && hydrate) {
            hydrate(vnode as VNode<Node, Element>, rootContainer as any)
          } else {
            render(vnode, rootContainer, isSVG)
          }
          isMounted = true
          app._container = rootContainer
          // for devtools and telemetry
          ;(rootContainer as any).__vue_app__ = app

          if (__DEV__ || __FEATURE_PROD_DEVTOOLS__) {
            app._instance = vnode.component
            devtoolsInitApp(app, version)
          }

          return getExposeProxy(vnode.component!) || vnode.component!.proxy
        } else if (__DEV__) {
          // warn 略
        }
      }

      unmount() {}

      provide(key, value) {}

      // ...
    })

    return app
  }
}
```

### 总结
vue项目初始化渲染到dom上经历了以下几点：
1. `createApp`内调用`ensureRenderer().createApp`方法，并扩展了它的`mount`方法
2. 用户调用`app.mount`, 内部会获取app挂载的dom节点；
`mount`函数干了什么：
1. 通过`createVNode`创建了`vnode`
2. 将`appContext`放入`vnode`
3. 调用`baseCreateRenderer`中定义的`render`函数将`vnode`渲染到我们设置的容器dom上,即`#app`; 
4. `render`函数内部调用了`path`方法更新节点

渲染相关的部分可以看`render/index.md`
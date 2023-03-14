# 初始化渲染

#### createApp
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

  // ...

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
// 路径：packages/runtime-dom/src/renderer.ts baseCreateRenderer 
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
      patch(container._vnode || null, vnode, container, null, null, null, isSVG)
    }
    flushPreFlushCbs()
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
我们可以看到很多熟悉的api，这里主要看下`mount`函数干了什么：
+ 通过`createVNode`创建了`vnode`
+ 将`appContext`放入`vnode`
+ 调用`baseCreateRenderer`中定义的`render`函数将`vnode`渲染到我们设置的容器dom上,即`#app`; 

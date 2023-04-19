# LifeCycle
`Vue3`的生命周期有：
1. `beforeCreate`: 在实例初始化之后，数据观测 (data observer) 和 event/watcher 事件配置之前被调用。
2. `created`: 实例已经创建完成之后被调用。在这一步，实例已经完成了数据观测 (data observer) ，属性和方法的运算，watch/event 事件回调等基本配置。
3. `beforeMount`: 在挂载开始之前被调用。相关的 render 函数首次被调用。
4. `mounted`: 实例挂载之后调用，这时 el 被新创建的 vm.$el 替换了。
5. `beforeUpdate`: 数据更新时调用，发生在虚拟 DOM 重新渲染和打补丁之前。可以在该钩子中进一步地更改状态，不会触发附加的重渲染过程。
6. `updated`: 由于数据更改导致的虚拟 DOM 重新渲染和打补丁完成之后调用。
7. `beforeUnmount`: 实例销毁之前调用。在这一步，实例仍然完全可用。
8. `unmounted`: Vue 实例销毁后调用。此时，所有指令已被解绑，所有事件监听器已被移除，所有子实例也已经被销毁。

以下是`Vue`官方的生命周期图示
![](https://cn.vuejs.org/assets/lifecycle.16e4c08e.png)

由图可知，setup方法是最开始执行的，我们平常写的生命周期钩子也会在这里被收集，在对应的阶段被调用。
```vue
<script setup>
	onBeforeMount(() => {})
	onMounted(() => {})
	onBeforeUpdate(() => {})
	onUpdated(() => {})
	onBeforeUnmount(() => {})
	onUnmounted(() => {})
</script>
```

## 源码分析
在`render`的分析中我们看到了 `beforeMount`这些生命周期的调用，现在我们已生命周期为主,重新分析下`setupStatefulComponent`方法

### setupStatefulComponent
1. `setup`方法在此时调用，优先于所有生命周期

```ts
// packages/runtime-core/src/component.ts 
function setupStatefulComponent(
  instance: ComponentInternalInstance,
  isSSR: boolean
) {
  // ...
  // 0. create render proxy property access cache
  instance.accessCache = Object.create(null)
  // 1. create public instance / render proxy
  // also mark it raw so it's never observed
  instance.proxy = markRaw(new Proxy(instance.ctx, PublicInstanceProxyHandlers))
  // 2. call setup()
  const { setup } = Component
  if (setup) {
    const setupContext = (instance.setupContext =
      setup.length > 1 ? createSetupContext(instance) : null)
	// ...

	// 内部调用setup方法，并捕获报错
    const setupResult = callWithErrorHandling(
      setup,
      instance,
      ErrorCodes.SETUP_FUNCTION,
      [__DEV__ ? shallowReadonly(instance.props) : instance.props, setupContext]
    )
	// ...

    if (isPromise(setupResult)) {
      setupResult.then(unsetCurrentInstance, unsetCurrentInstance)
      if (isSSR) {
        // return the promise so server-renderer can wait on it
		// ...
      } else if (__FEATURE_SUSPENSE__) {
        // async setup returned Promise.
        // bail here and wait for re-entry.
        // ...
      } else if (__DEV__) {
        warn(
          `setup() returned a Promise, but the version of Vue you are using ` +
            `does not support it yet.`
        )
      }
    } else {
      // 内部会调用finishComponentSetup
      handleSetupResult(instance, setupResult, isSSR)
    }
  } else {
    finishComponentSetup(instance, isSSR)
  }
}
```

### finishComponentSetup
1. `setup`执行完后调用`applyOptions`(`defineComponent`的写法)
2. `applyOptions`依次处理`beforeCreate`、`props`、`inject`、`methods`、`data`、`computed`、`watch`、`provide`、`created`
3. 注册对应的生命周期钩子
```ts
export function finishComponentSetup(
  instance: ComponentInternalInstance,
  isSSR: boolean,
  skipOptions?: boolean
) {
  const Component = instance.type as ComponentOptions
  // ...

  // support for 2.x options
  if (__FEATURE_OPTIONS_API__ && !(__COMPAT__ && skipOptions)) {
	// ...
    applyOptions(instance)
	// ...
  }
  // ...
}
```

```ts
// packages/runtime-core/src/componentOptions.ts
export function applyOptions(instance: ComponentInternalInstance) {
  //...
  // call beforeCreate first before accessing other options since
  // the hook may mutate resolved options (#2791)
  if (options.beforeCreate) {
    callHook(options.beforeCreate, instance, LifecycleHooks.BEFORE_CREATE)
  }
  // ...
  if (__DEV__) {
    const [propsOptions] = instance.propsOptions
    if (propsOptions) {
      for (const key in propsOptions) {
        checkDuplicateProperties!(OptionTypes.PROPS, key)
      }
    }
  }

  // options initialization order (to be consistent with Vue 2):
  // - props (already done outside of this function)
  // - inject
  // - methods
  // - data (deferred since it relies on `this` access)
  // - computed
  // - watch (deferred since it relies on `this` access)

  if (created) {
    callHook(created, instance, LifecycleHooks.CREATED)
  }

  // 就是我们平时直接使用onMounted(mounted)
  function registerLifecycleHook(
    register: Function,
    hook?: Function | Function[]
  ) {
    if (isArray(hook)) {
      hook.forEach(_hook => register(_hook.bind(publicThis)))
    } else if (hook) {
      register((hook as Function).bind(publicThis))
    }
  }

  registerLifecycleHook(onBeforeMount, beforeMount)
  registerLifecycleHook(onMounted, mounted)
  registerLifecycleHook(onBeforeUpdate, beforeUpdate)
  registerLifecycleHook(onUpdated, updated)
  registerLifecycleHook(onActivated, activated)
  registerLifecycleHook(onDeactivated, deactivated)
  registerLifecycleHook(onErrorCaptured, errorCaptured)
  registerLifecycleHook(onRenderTracked, renderTracked)
  registerLifecycleHook(onRenderTriggered, renderTriggered)
  registerLifecycleHook(onBeforeUnmount, beforeUnmount)
  registerLifecycleHook(onUnmounted, unmounted)
  registerLifecycleHook(onServerPrefetch, serverPrefetch)

  if (__COMPAT__) {
    if (
      beforeDestroy &&
      softAssertCompatEnabled(DeprecationTypes.OPTIONS_BEFORE_DESTROY, instance)
    ) {
      registerLifecycleHook(onBeforeUnmount, beforeDestroy)
    }
    if (
      destroyed &&
      softAssertCompatEnabled(DeprecationTypes.OPTIONS_DESTROYED, instance)
    ) {
      registerLifecycleHook(onUnmounted, destroyed)
    }
  }
  // ...
}
```
上面的方式与我们平常的使用的`setup`语法糖, 已`onMounted`为例，
```vue
<script setup>
  onMounted(() => {})
</script>
```
编译结果如下。我们使用的`onMounted`其实是一个注册的钩子，效果类似上面的`registerLifecycleHook(onMounted, mounted)`
```js
const setup = (props, { emit }) => {
  onMounted(() => {
    console.log('mounted')
  })
  return {}
}
```

### 生命周期钩子
`Vue`会对外暴露一些列的生命周期钩子函数共开发者使用，下面我们看看这些函数干了啥
1. 通过`createHook`创建对应的生命周期钩子
2. `injectHook`对传入的方法做了错误处理`wrappedHook`
3. 并将`wrappedHook`加到组件实例对应的生命周期属性上
```ts
// packages/runtime-core/src/apiLifecycle.ts
export const createHook =
  <T extends Function = () => any>(lifecycle: LifecycleHooks) =>
  (hook: T, target: ComponentInternalInstance | null = currentInstance) =>
    // post-create lifecycle registrations are noops during SSR (except for serverPrefetch)
    (!isInSSRComponentSetup || lifecycle === LifecycleHooks.SERVER_PREFETCH) &&
    injectHook(lifecycle, (...args: unknown[]) => hook(...args), target)

export const onBeforeMount = createHook(LifecycleHooks.BEFORE_MOUNT)
export const onMounted = createHook(LifecycleHooks.MOUNTED)
export const onBeforeUpdate = createHook(LifecycleHooks.BEFORE_UPDATE)
export const onUpdated = createHook(LifecycleHooks.UPDATED)
export const onBeforeUnmount = createHook(LifecycleHooks.BEFORE_UNMOUNT)
export const onUnmounted = createHook(LifecycleHooks.UNMOUNTED)
export const onServerPrefetch = createHook(LifecycleHooks.SERVER_PREFETCH)
```
```ts
// packages/runtime-core/src/apiLifecycle.ts
export function injectHook(
  type: LifecycleHooks,
  hook: Function & { __weh?: Function },
  target: ComponentInternalInstance | null = currentInstance,
  prepend: boolean = false
): Function | undefined {
  if (target) {
    const hooks = target[type] || (target[type] = [])
    // cache the error handling wrapper for injected hooks so the same hook
    // can be properly deduped by the scheduler. "__weh" stands for "with error
    // handling".
	
	// 加了一层错误处理
    const wrappedHook =
      hook.__weh ||
      (hook.__weh = (...args: unknown[]) => {
        if (target.isUnmounted) {
          return
        }
        // disable tracking inside all lifecycle hooks
        // since they can potentially be called inside effects.
        pauseTracking()
        // Set currentInstance during hook invocation.
        // This assumes the hook does not synchronously trigger other hooks, which
        // can only be false when the user does something really funky.
        setCurrentInstance(target)
        const res = callWithAsyncErrorHandling(hook, target, type, args)
        unsetCurrentInstance()
        resetTracking()
        return res
      })
    if (prepend) {
      hooks.unshift(wrappedHook)
    } else {
      hooks.push(wrappedHook)
    }
    return wrappedHook
  } else if (__DEV__) {
    const apiName = toHandlerKey(ErrorTypeStrings[type].replace(/ hook$/, ''))
    warn(
      `${apiName} is called when there is no active component instance to be ` +
        `associated with. ` +
        `Lifecycle injection APIs can only be used during execution of setup().` +
        (__FEATURE_SUSPENSE__
          ? ` If you are using async setup(), make sure to register lifecycle ` +
            `hooks before the first await statement.`
          : ``)
    )
  }
}
```

### 生命周期的调用时机
在上面的`finishComponentSetup` 部分已经介绍了`beforeCreate`和`created`的调用时机,下面看下`mount`、`update`、`unmounted`相关生命周期的调用：
#### mounted
1. 如果实例没有挂载, 执行收集的`beforeMount`方法，执行`props`内`onVnodeBeforeMount`的方法， 并向外发射`hook:beforeMount`
2. 调用`patch`挂载子组件
3. 执行收集的`mounted`方法，执行`props`内`onVnodeMounted`的方法， 并向外发射`hook:mounted`
4. 如果是`keep-alive`组件，会触发`activated`生命周期钩子
5. `instance.isMounted = true` 标记组件已经挂载

```ts
// packages/runtime-core/src/renderer.ts  setupRenderEffect方法内声明
const componentUpdateFn = () => {
   if (!instance.isMounted) {
     let vnodeHook: VNodeHook | null | undefined
     const { el, props } = initialVNode
     const { bm, m, parent } = instance
     const isAsyncWrapperVNode = isAsyncWrapper(initialVNode)

     // 关闭递归
     toggleRecurse(instance, false)
     // beforeMount hook
     if (bm) {
       invokeArrayFns(bm)
     }
     // onVnodeBeforeMount
     if (
       !isAsyncWrapperVNode &&
       (vnodeHook = props && props.onVnodeBeforeMount)
     ) {
       invokeVNodeHook(vnodeHook, parent, initialVNode)
     }
     if (
       __COMPAT__ &&
       isCompatEnabled(DeprecationTypes.INSTANCE_EVENT_HOOKS, instance)
     ) {
       instance.emit('hook:beforeMount')
     }

      // 开启递归
     toggleRecurse(instance, true)

     if (el && hydrateNode) {
       // vnode has adopted host node - perform hydration instead of mount.
     } else {
       const subTree = (instance.subTree = renderComponentRoot(instance))
       patch(
         null,
         subTree,
         container,
         anchor,
         instance,
         parentSuspense,
         isSVG
       )
       initialVNode.el = subTree.el
     }
     // mounted hook
     if (m) {
       queuePostRenderEffect(m, parentSuspense)
     }
     // onVnodeMounted
     if (
       !isAsyncWrapperVNode &&
       (vnodeHook = props && props.onVnodeMounted)
     ) {
       const scopedInitialVNode = initialVNode
       queuePostRenderEffect(
         () => invokeVNodeHook(vnodeHook!, parent, scopedInitialVNode),
         parentSuspense
       )
     }
     if (
       __COMPAT__ &&
       isCompatEnabled(DeprecationTypes.INSTANCE_EVENT_HOOKS, instance)
     ) {
       queuePostRenderEffect(
         () => instance.emit('hook:mounted'),
         parentSuspense
       )
     }

     // activated hook for keep-alive roots.
     // #1742 activated hook must be accessed after first render
     // since the hook may be injected by a child keep-alive
     if (
       initialVNode.shapeFlag & ShapeFlags.COMPONENT_SHOULD_KEEP_ALIVE ||
       (parent &&
         isAsyncWrapper(parent.vnode) &&
         parent.vnode.shapeFlag & ShapeFlags.COMPONENT_SHOULD_KEEP_ALIVE)
     ) {
       instance.a && queuePostRenderEffect(instance.a, parentSuspense)
       if (
         __COMPAT__ &&
         isCompatEnabled(DeprecationTypes.INSTANCE_EVENT_HOOKS, instance)
       ) {
         queuePostRenderEffect(
           () => instance.emit('hook:activated'),
           parentSuspense
         )
       }
     }
     instance.isMounted = true

     // #2458: deference mount-only object parameters to prevent memleaks
     initialVNode = container = anchor = null as any
   } else {
     // updateComponent
   }
 }
```

#### updated
1. [组件修改触发更新]('../reactivity/index.md')，执行收集的`beforeUpdate`方法，执行`props`内`onVnodeBeforeUpdate`的方法， 并向外发射`hook:beforeUpdate`
2. 调用`patch`更新子组件
3. 执行收集的`updated`方法，执行`props`内`onVnodeUpdated`的方法， 并向外发射`hook:updated`

```ts
// packages/runtime-core/src/renderer.ts  setupRenderEffect方法内声明
const componentUpdateFn = () => {
   if (!instance.isMounted) {
    // mount component
   } else {
     // updateComponent
     // This is triggered by mutation of component's own state (next: null)
     // OR parent calling processComponent (next: VNode)
     let { next, bu, u, parent, vnode } = instance
     let originNext = next
     let vnodeHook: VNodeHook | null | undefined

     // Disallow component effect recursion during pre-lifecycle hooks.
     toggleRecurse(instance, false)

    //  next 就是表示下一个更新周期中的组件 vnode。而 updateComponentPreRender 方法主要是对下一个版本的组件 vnode 进行预渲染更新，即对组件的状态和属性进行更新，以便在下一个周期的渲染中能够尽可能地优化性能。
     if (next) {
       next.el = vnode.el
       updateComponentPreRender(instance, next, optimized)
     } else {
       next = vnode
     }

     // beforeUpdate hook
     if (bu) {
       invokeArrayFns(bu)
     }
     // onVnodeBeforeUpdate
     if ((vnodeHook = next.props && next.props.onVnodeBeforeUpdate)) {
       invokeVNodeHook(vnodeHook, parent, next, vnode)
     }
     if (
       __COMPAT__ &&
       isCompatEnabled(DeprecationTypes.INSTANCE_EVENT_HOOKS, instance)
     ) {
       instance.emit('hook:beforeUpdate')
     }

     // render
     const nextTree = renderComponentRoot(instance)
   
     const prevTree = instance.subTree
     instance.subTree = nextTree

     patch(
       prevTree,
       nextTree,
       // parent may have changed if it's in a teleport
       hostParentNode(prevTree.el!)!,
       // anchor may have changed if it's in a fragment
       getNextHostNode(prevTree),
       instance,
       parentSuspense,
       isSVG
     )
     next.el = nextTree.el
     if (originNext === null) {
       // self-triggered update. In case of HOC, update parent component
       // vnode el. HOC is indicated by parent instance's subTree pointing
       // to child component's vnode
       updateHOCHostEl(instance, nextTree.el)
     }
     // updated hook
     if (u) {
       queuePostRenderEffect(u, parentSuspense)
     }
     // onVnodeUpdated
     if ((vnodeHook = next.props && next.props.onVnodeUpdated)) {
       queuePostRenderEffect(
         () => invokeVNodeHook(vnodeHook!, parent, next!, vnode),
         parentSuspense
       )
     }
     if (
       __COMPAT__ &&
       isCompatEnabled(DeprecationTypes.INSTANCE_EVENT_HOOKS, instance)
     ) {
       queuePostRenderEffect(
         () => instance.emit('hook:updated'),
         parentSuspense
       )
     }
   }
```

#### unmounted
1. 在`patch`方法中，如果新旧节点不相同则将触发组件的卸载
2. 执行收集的`beforeUnmount`方法，并向外发射`hook:beforeDestroy`
3. 卸载子组件
4. 执行收集的`unmount`方法，并向外发射`hook:destroyed`
5. 组件标记为已卸载`instance.isUnmounted = true`
```ts
// 路径：packages/runtime-core/src/renderer.ts baseCreateRenderer 
// patching & not same type, unmount old tree
if (n1 && !isSameVNodeType(n1, n2)) {
  anchor = getNextHostNode(n1)
  unmount(n1, parentComponent, parentSuspense, true)
  n1 = null
}
```
```ts
// 路径：packages/runtime-core/src/renderer.ts
const unmount: UnmountFn = (
 vnode,
 parentComponent,
 parentSuspense,
 doRemove = false,
 optimized = false
) => {
 const {
   type,
   props,
   ref,
   children,
   dynamicChildren,
   shapeFlag,
   patchFlag,
   dirs
 } = vnode
 // ...
 if (shapeFlag & ShapeFlags.COMPONENT) {
   unmountComponent(vnode.component!, parentSuspense, doRemove)
 } else {
   // ...
   //  unmountChildren
   // ...
   if (doRemove) {
     remove(vnode)
   }
 }
}
```

```ts
// 路径：packages/runtime-core/src/renderer.ts
const unmountComponent = (
 instance: ComponentInternalInstance,
 parentSuspense: SuspenseBoundary | null,
 doRemove?: boolean
) => {
 const { bum, scope, update, subTree, um } = instance

 // beforeUnmount hook
 if (bum) {
   invokeArrayFns(bum)
 }

 if (
   __COMPAT__ &&
   isCompatEnabled(DeprecationTypes.INSTANCE_EVENT_HOOKS, instance)
 ) {
   instance.emit('hook:beforeDestroy')
 }

 // stop effects in component scope
 scope.stop()

 // update may be null if a component is unmounted before its async
 // setup has resolved.
 if (update) {
   // so that scheduler will no longer invoke it
   update.active = false
   unmount(subTree, instance, parentSuspense, doRemove)
 }
 // unmounted hook
 if (um) {
   queuePostRenderEffect(um, parentSuspense)
 }
 if (
   __COMPAT__ &&
   isCompatEnabled(DeprecationTypes.INSTANCE_EVENT_HOOKS, instance)
 ) {
   queuePostRenderEffect(
     () => instance.emit('hook:destroyed'),
     parentSuspense
   )
 }
 //...
}
```
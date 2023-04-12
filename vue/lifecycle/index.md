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



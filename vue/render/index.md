# Vue的渲染

### baseCreateRenderer
在`createApp`的mount方法中，调用`baseCreateRenderer`返回的`render`函数来进行渲染；内部会调用`patch`方法，将虚拟 DOM 转换为真实 DOM
```ts
// 路径：packages/runtime-core/src/renderer.ts baseCreateRenderer 
// 这个方法有2000行左右，后面在render板块中单独分析
function baseCreateRenderer(
  options: RendererOptions,
  createHydrationFns?: typeof createHydrationFunctions
): any {
  const patch: PatchFn = (
    n1,         // 旧 VNode
    n2,                // 新 VNode
    container,  // 真实 DOM 容器
    anchor = null,  // 插入位置
    parentComponent = null,  // 父级组件实例
    parentSuspense = null,  // 父级 suspense 上下文
    isSVG = false,  // 是否为 SVG 元素
    slotScopeIds = null,
    optimized = __DEV__ && isHmrUpdating ? false : !!n2.dynamicChildren
  ) => {
	if (n1 === n2) {
      return
    }

    // patching & not same type, unmount old tree
    if (n1 && !isSameVNodeType(n1, n2)) {
      anchor = getNextHostNode(n1)
      unmount(n1, parentComponent, parentSuspense, true)
      n1 = null
    }

    if (n2.patchFlag === PatchFlags.BAIL) {
      optimized = false
      n2.dynamicChildren = null
    }

    const { type, ref, shapeFlag } = n2
    switch (type) {
      case Text:
		// ...
        break
      case Comment:
       // ...
        break
      case Static:
        // ...
        break
      case Fragment:
        // ...
        break
      default:
        if (shapeFlag & ShapeFlags.ELEMENT) {
			// ...
        } else if (shapeFlag & ShapeFlags.COMPONENT) {
          processComponent(
            n1,
            n2,
            container,
            anchor,
            parentComponent,
            parentSuspense,
            isSVG,
            slotScopeIds,
            optimized
          )
        } else if (shapeFlag & ShapeFlags.TELEPORT) {
			//   ...
        } else if (__FEATURE_SUSPENSE__ && shapeFlag & ShapeFlags.SUSPENSE) {
        	//  ...
        } else if (__DEV__) {
          warn('Invalid VNode type:', type, `(${typeof type})`)
        }
    }

    // set ref
    if (ref != null && parentComponent) {
      setRef(ref, n1 && n1.ref, parentSuspense, n2 || n1, !n2)
    }
  }

  // 1. 判断vnode是不是keep_alive组件，2. 是挂载组件还是更新组件
  const processComponent = (
    n1: VNode | null,
    n2: VNode,
    container: RendererElement,
    anchor: RendererNode | null,
    parentComponent: ComponentInternalInstance | null,
    parentSuspense: SuspenseBoundary | null,
    isSVG: boolean,
    slotScopeIds: string[] | null,
    optimized: boolean
  ) => {
    n2.slotScopeIds = slotScopeIds
    if (n1 == null) {
      if (n2.shapeFlag & ShapeFlags.COMPONENT_KEPT_ALIVE) {
        ;(parentComponent!.ctx as KeepAliveContext).activate(
          n2,
          container,
          anchor,
          isSVG,
          optimized
        )
      } else {
        mountComponent(
          n2,
          container,
          anchor,
          parentComponent,
          parentSuspense,
          isSVG,
          optimized
        )
      }
    } else {
      updateComponent(n1, n2, optimized)
    }
  }
}
```


#### mountComponent
1. `createComponentInstance`创建组件实例
2. `setupComponent`安装组件
3. `setupRenderEffect`渲染组件
```js
const mountComponent: MountComponentFn = (
    initialVNode,
    container,
    anchor,
    parentComponent,
    parentSuspense,
    isSVG,
    optimized
  ) => {
    // 2.x compat may pre-create the component instance before actually
    // mounting
    const compatMountInstance =
      __COMPAT__ && initialVNode.isCompatRoot && initialVNode.component

	// createComponentInstance 里面就是声明里一个对象，并没有其他处理
	// 源码路径packages/runtime-core/src/component.ts
    const instance: ComponentInternalInstance =
      compatMountInstance ||
      (initialVNode.component = createComponentInstance(
        initialVNode,
        parentComponent,
        parentSuspense
      ))

   // 注册热更新
    if (__DEV__ && instance.type.__hmrId) {
      registerHMR(instance)
    }

	// ...

    // resolve props and slots for setup context
    if (!(__COMPAT__ && compatMountInstance)) {
      //...
      setupComponent(instance)
      // ...
    }

	// ...	

    setupRenderEffect(
      instance,
      initialVNode,
      container,
      anchor,
      parentSuspense,
      isSVG,
      optimized
    )
	// ...
}
```
#### setupComponent
1. 初始化组件的props、slots，
2. 如果组件是有状态组件，需要执行setupStatefulComponent, 这里面会执行组件内的setup方法
```ts
// packages/runtime-core/src/component.ts 
export function setupComponent(
  instance: ComponentInternalInstance,
  isSSR = false
) {
  isInSSRComponentSetup = isSSR

  const { props, children } = instance.vnode
  const isStateful = isStatefulComponent(instance)
  initProps(instance, props, isStateful, isSSR)
  initSlots(instance, children)

  const setupResult = isStateful
    ? setupStatefulComponent(instance, isSSR)
    : undefined
  isInSSRComponentSetup = false
  return setupResult
}


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
      handleSetupResult(instance, setupResult, isSSR)
    }
  } else {
    finishComponentSetup(instance, isSSR)
  }
}

```

#### setupRenderEffect

1. 创建`ReactiveEffect`实例对象`effect`
2. 将`effect.run`给`instance.update`, 并调用该方法，后面组件如果更新将会在`updateComponent`调用该方法

```ts
// packages/runtime-core/src/renderer.ts
const setupRenderEffect: SetupRenderEffectFn = (
 instance,
 initialVNode,
 container,
 anchor,
 parentSuspense,
 isSVG,
 optimized
) => {
	const componentUpdateFn = () => {
		// ...
	}
	// create reactive effect for rendering
    const effect = (instance.effect = new ReactiveEffect(
      componentUpdateFn,
      () => queueJob(update),
      instance.scope // track it in component's effect scope
    ))

    const update: SchedulerJob = (instance.update = () => effect.run())
    update.id = instance.uid
    // allowRecurse
    // #1801, #2043 component render effects should allow recursive updates
    toggleRecurse(instance, true)

    if (__DEV__) {
      effect.onTrack = instance.rtc
        ? e => invokeArrayFns(instance.rtc!, e)
        : void 0
      effect.onTrigger = instance.rtg
        ? e => invokeArrayFns(instance.rtg!, e)
        : void 0
      update.ownerInstance = instance
    }

    update()
}
```

##### componentUpdateFn
1. 调用`beforeMount hook`
2. 调用`renderComponentRoot`生成`subtree`
3. 调用`patch`, 对`subtree`进行渲染
4. 调用`mounted hook`
5. 标记组件已经渲染完成`instance.isMounted = true`
```ts
const componentUpdateFn = () => {
	// 如果组件没有渲染
   if (!instance.isMounted) {
     let vnodeHook: VNodeHook | null | undefined
     const { el, props } = initialVNode
	  // 获取组件上挂载的beforeMount onMounted parent
     const { bm, m, parent } = instance
     const isAsyncWrapperVNode = isAsyncWrapper(initialVNode)

     toggleRecurse(instance, false)
     // beforeMount hook
     if (bm) {
       invokeArrayFns(bm) // 调用beforeMount
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
     toggleRecurse(instance, true)

     if (el && hydrateNode) {
       // vnode has adopted host node - perform hydration instead of mount.
	   // ...
     } else {
       const subTree = (instance.subTree = renderComponentRoot(instance))
      
	   // 在挂载组件前，先挂载子组件
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
	 
	 // ...

     instance.isMounted = true

     // #2458: deference mount-only object parameters to prevent memleaks
     initialVNode = container = anchor = null as any
   } else {
     // updateComponent
	 // ...
   }
 }
```

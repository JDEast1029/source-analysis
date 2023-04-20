# Watch

## 问题
1. 响应对象的值发生变化怎么通知`watch`的？

## doWatch
`doWatch`是`watch`相关API的核心方法；
1. 将`source`转变为`getter`方法
2. 创建一个`SchedulerJob`方法`job`, 内部定义了`watch`和`watchEffect`的执行逻辑
3. 定义`scheduler`，根据`flush`定义其执行方式是异步or同步，是渲染前还是渲染后执行
4. 定义`effect`, 将`getter`作为fn
5. 执行`effect.run()`
6. 返回`unwatch`方法
```ts
function doWatch(
  source: WatchSource | WatchSource[] | WatchEffect | object,
  cb: WatchCallback | null,
  { immediate, deep, flush, onTrack, onTrigger }: WatchOptions = EMPTY_OBJ
): WatchStopHandle {
  // ...

  const instance =
    getCurrentScope() === currentInstance?.scope ? currentInstance : null
  // const instance = currentInstance
  let getter: () => any
  let forceTrigger = false
  let isMultiSource = false

  /** 赋值getter **/
  if (isRef(source)) {
  } else if (isReactive(source)) {
  } else if (isArray(source)) {
  } else if (isFunction(source)) {
  } else {
  }
  if (__COMPAT__ && cb && !deep) {}
  if (cb && deep) {}
  /** 赋值getter **/

  let cleanup: () => void
  let onCleanup: OnCleanup = (fn: () => void) => {
    cleanup = effect.onStop = () => {
      callWithErrorHandling(fn, instance, ErrorCodes.WATCH_CLEANUP)
    }
  }

  // in SSR there is no need to setup an actual effect, and it should be noop
  // unless it's eager or sync flush
  // ... ssr

  let oldValue: any = isMultiSource
    ? new Array((source as []).length).fill(INITIAL_WATCHER_VALUE)
    : INITIAL_WATCHER_VALUE
  const job: SchedulerJob = () => {
    if (!effect.active) {
      return
    }
    if (cb) {
      // watch(source, cb)
      const newValue = effect.run()
      if (
        deep ||
        forceTrigger ||
        (isMultiSource
          ? (newValue as any[]).some((v, i) =>
              hasChanged(v, (oldValue as any[])[i])
            )
          : hasChanged(newValue, oldValue)) ||
        (__COMPAT__ &&
          isArray(newValue) &&
          isCompatEnabled(DeprecationTypes.WATCH_ARRAY, instance))
      ) {
        // cleanup before running cb again
        if (cleanup) {
          cleanup()
        }
        callWithAsyncErrorHandling(cb, instance, ErrorCodes.WATCH_CALLBACK, [
          newValue,
          // pass undefined as the old value when it's changed for the first time
          oldValue === INITIAL_WATCHER_VALUE
            ? undefined
            : isMultiSource && oldValue[0] === INITIAL_WATCHER_VALUE
            ? []
            : oldValue,
          onCleanup
        ])
        oldValue = newValue
      }
    } else {
      // watchEffect
      effect.run()
    }
  }

  // important: mark the job as a watcher callback so that scheduler knows
  // it is allowed to self-trigger (#1727)
  job.allowRecurse = !!cb

  let scheduler: EffectScheduler
  if (flush === 'sync') { 
	// 直接调用，下面的queuePostRenderEffect和queueJob都是异步的，详见../render/scheduler.md
    scheduler = job as any // the scheduler function gets called directly
  } else if (flush === 'post') {
	// 渲染后执行
    scheduler = () => queuePostRenderEffect(job, instance && instance.suspense)
  } else {
    // default: 'pre' 渲染前执行
    job.pre = true
    if (instance) job.id = instance.uid
    scheduler = () => queueJob(job)
  }

  const effect = new ReactiveEffect(getter, scheduler)

  // initial run
  if (cb) {
    if (immediate) {
      job()
    } else {
      oldValue = effect.run()
    }
  } else if (flush === 'post') {
    queuePostRenderEffect(
      effect.run.bind(effect),
      instance && instance.suspense
    )
  } else {
    effect.run()
  }

  const unwatch = () => {
    effect.stop()
    if (instance && instance.scope) {
      remove(instance.scope.effects!, effect)
    }
  }

  return unwatch
}
```

## watchEffect
1. 在`doWatch`的`initial run`阶段，执行的是`effect.run()`
2. 将当前的`activeEffect`设置为`doWatch`内申明的`effect`
3. `run`内执行了`getter`即`() => console.log(count.value)`, 调用了`count.value`的`get value()`,收集`activeEffect`, 因为这是立马执行的，所以打印0
4. `count.value++`触发`set value()`, 调用`effect.scheduler()`,即`() => queueJob(job)`, job内会执行`effect.run()`,重新触发`getter`，打印1
```ts
const count = ref(0)

watchEffect(() => console.log(count.value))
// -> 输出 0

count.value++
// -> 输出 1
```

```ts
export function watchEffect(
  effect: WatchEffect,
  options?: WatchOptionsBase
): WatchStopHandle {
  return doWatch(effect, null, options)
}
```

## watch
1. 在`doWatch`的`initial run`阶段，执行的是`oldValue = effect.run()` 获取旧值，此时调用了`state.count`的`get value()`,收集`activeEffect`即`doWatch`内申明的`effect`
2. `state.count++`触发`set value()`, 调用`effect.scheduler()`,即`() => queueJob(job)`，
3. 再次执行`effect.run()`获取新值, 执行`cb`,将新值和旧值传入
4. 所以watch不会一开始就触发回调，除非`{ immediate: true }`,这是它会执行`job`即3的步骤
```ts
const state = reactive({ count: 0 })
watch(
  () => state.count,
  (count, prevCount) => {
    /* ... */
  }
)
state.count++
```
```ts
export function watch<T = any, Immediate extends Readonly<boolean> = false>(
  source: T | WatchSource<T>,
  cb: any,
  options?: WatchOptions<Immediate>
): WatchStopHandle {
  // ...
  return doWatch(source as any, cb, options)
}
```
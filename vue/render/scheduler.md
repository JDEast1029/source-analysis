# Scheduler(调度器)

## nextTick
我们知道`nextTick`是等待下一次 DOM 更新刷新的工具方法，从代码可已看出等待的就是`currentFlushPromise`，如果当前没有存在更新的promise，则使用一个resolved状态的promise替代
```ts
export function nextTick<T = void>(
  this: T,
  fn?: (this: T) => void
): Promise<void> {
  const p = currentFlushPromise || resolvedPromise
  return fn ? p.then(this ? fn.bind(this) : fn) : p
}
```

## queueJob
在`render`的`setupRenderEffect`中，`queueJob`会被传入到`ReactiveEffect`，当[触发响应](../reactivity/index.md)时，就会执行将`update`任务传入调度队列
```ts
// packages/runtime-core/src/renderer.ts   setupRenderEffect
const effect = (instance.effect = new ReactiveEffect(
	componentUpdateFn,
	() => queueJob(update), // 作为scheduler
	instance.scope // track it in component's effect scope
))

// effect.run 最终就会执行componentUpdateFn，会执行mounted和updated
const update: SchedulerJob = (instance.update = () => effect.run())
```

```ts
// packages/runtime-core/src/scheduler.ts
export function queueJob(job: SchedulerJob) {
  // the dedupe search uses the startIndex argument of Array.includes()
  // by default the search index includes the current job that is being run
  // so it cannot recursively trigger itself again.
  // if the job is a watch() callback, the search will start with a +1 index to
  // allow it recursively trigger itself - it is the user's responsibility to
  // ensure it doesn't end up in an infinite loop.
  if (
    !queue.length ||
    !queue.includes(
      job,
      isFlushing && job.allowRecurse ? flushIndex + 1 : flushIndex
    )
  ) {
    if (job.id == null) {
      queue.push(job)
    } else {
	  // 将邻近的id的任务放到一起
      queue.splice(findInsertionIndex(job.id), 0, job)
    }
    queueFlush()
  }
}
```

```ts
function queueFlush() {
  if (!isFlushing && !isFlushPending) {
    isFlushPending = true
    currentFlushPromise = resolvedPromise.then(flushJobs)
  }
}
```

## queuePostFlushCb
在`render`中，我们看到有很多地方都调用了`queuePostRenderEffect`方法来处理渲染后的副作用，而这里面就调用了`queuePostFlushCb`。
```ts
// packages/runtime-core/src/renderer.ts
// 如果不是异步组件，则使用queuePostFlushCb
export const queuePostRenderEffect = __FEATURE_SUSPENSE__
  ? __TEST__
    ? // vitest can't seem to handle eager circular dependency
      (fn: Function | Function[], suspense: SuspenseBoundary | null) =>
        queueEffectWithSuspense(fn, suspense)
    : queueEffectWithSuspense
  : queuePostFlushCb
```
将任务推送到 等待队列`pendingPostFlushCbs`内，在调用`queueFlush`执行任务，在当前队列清理完毕后会调用`flushPostFlushCbs`，执行等待队列中的回调
```ts
// packages/runtime-core/src/scheduler.ts
export function queuePostFlushCb(cb: SchedulerJobs) {
  if (!isArray(cb)) {
    if (
      !activePostFlushCbs ||
      !activePostFlushCbs.includes(
        cb,
        cb.allowRecurse ? postFlushIndex + 1 : postFlushIndex
      )
    ) {
      pendingPostFlushCbs.push(cb)
    }
  } else {
    // if cb is an array, it is a component lifecycle hook which can only be
    // triggered by a job, which is already deduped in the main queue, so
    // we can skip duplicate check here to improve perf
    pendingPostFlushCbs.push(...cb)
  }
  queueFlush()
}
```

```ts
function flushJobs(seen?: CountMap) {
  isFlushPending = false
  isFlushing = true
  if (__DEV__) {
    seen = seen || new Map()
  }

  // Sort queue before flush.
  // This ensures that:
  // 1. Components are updated from parent to child. (because parent is always
  //    created before the child so its render effect will have smaller
  //    priority number)
  // 2. If a component is unmounted during a parent component's update,
  //    its update can be skipped.
  queue.sort(comparator) // 父子组件的副作用执行顺序跟其创建顺序保持一致

  // conditional usage of checkRecursiveUpdate must be determined out of
  // try ... catch block since Rollup by default de-optimizes treeshaking
  // inside try-catch. This can leave all warning code unshaked. Although
  // they would get eventually shaken by a minifier like terser, some minifiers
  // would fail to do that (e.g. https://github.com/evanw/esbuild/issues/1610)
  const check = __DEV__
    ? (job: SchedulerJob) => checkRecursiveUpdates(seen!, job)
    : NOOP

  try {
    for (flushIndex = 0; flushIndex < queue.length; flushIndex++) {
      const job = queue[flushIndex]
      if (job && job.active !== false) {
        if (__DEV__ && check(job)) {
          continue
        }
        // console.log(`running:`, job.id)
        callWithErrorHandling(job, null, ErrorCodes.SCHEDULER)
      }
    }
  } finally {
    flushIndex = 0
    queue.length = 0

    flushPostFlushCbs(seen)

    isFlushing = false
    currentFlushPromise = null
    // some postFlushCb queued jobs!(在flushPostFlushCbs的时候有些任务会往queue中添加任务，这里需要在执行一遍flushJobs，直到没有任务为止)
    // keep flushing until it drains.
    if (queue.length || pendingPostFlushCbs.length) {
      flushJobs(seen)
    }
  }
}
```

```ts
export function flushPostFlushCbs(seen?: CountMap) {
  if (pendingPostFlushCbs.length) {
    const deduped = [...new Set(pendingPostFlushCbs)]
    pendingPostFlushCbs.length = 0

    // #1947 already has active queue, nested flushPostFlushCbs call
    if (activePostFlushCbs) {
      activePostFlushCbs.push(...deduped)
      return
    }

    activePostFlushCbs = deduped
    if (__DEV__) {
      seen = seen || new Map()
    }

    activePostFlushCbs.sort((a, b) => getId(a) - getId(b))

    for (
      postFlushIndex = 0;
      postFlushIndex < activePostFlushCbs.length;
      postFlushIndex++
    ) {
      if (
        __DEV__ &&
        checkRecursiveUpdates(seen!, activePostFlushCbs[postFlushIndex])
      ) {
        continue
      }
      activePostFlushCbs[postFlushIndex]()
    }
    activePostFlushCbs = null
    postFlushIndex = 0
  }
}
```

## 总结
为什么说`nextTick`是在`dom`更新以后执行呢？上面说到`nextTick`其实是在等`currentFlushPromise`结束后才执行，而`currentFlushPromise`执行的是`flushJobs`方法，内部将之前通过`queueJob`加入的`update`方法执行完毕后才结束，而`update`就是将组件挂载或更新的方法
# Vue的响应式
Vue 最标志性的功能就是其低侵入性的响应式系统。组件状态都是由响应式的 JavaScript 对象组成的。当更改它们时，视图会随即自动更新。其核心API有下面几个：
1. ref()
2. reactive()
3. computed()
4. readonly()
5. watchEffect()
6. watchPostEffect()
7. watchSyncEffect()
8. watch()

## 流程介绍
我还是想先把整个响应流程写在最前面，这样下面的API分析才更好体会,下面将基于这个使用场景进行分析
```vue
<template>
  <div>{{ count }}</div>
</template>

<script>
import { ref } from 'vue'

export default {
  setup() {
    const count = ref(0)
    // ...

    count.value = 1
  }
}
</script>
```

1. 在[生命周期](../render/lifecycle.md)中可以知道，`setup`函数是在最开始执行的，所以上面的`count`在`template`内部使用之前已经申明了
2. 在[渲染](../render/index.md)一文中介绍了内部在挂载组件时会申明一个`update`函数并执行，即`() => effect.run()`, 这个`effect`我们就叫`effectA`，构造`effectA`时传递了`scheduler`和`effect.run()`内部执行的`fn` 在`run`执行时，内部将`activeEffect`赋值为`effectA`
3. `template`渲染函数调用了`count`,会触发`ref`内部定义的`get value()`，该方法内会调用`trackRefValue`方法收集`activeEffect`（`effectA`）存入`dep`内
4. 现在我们通过`count.value = 1`改变了count的值, 触发`set value()`, 该方法内会调用`triggerRefValue`方法，调用之前在`dep`内收集的`effect`（`effectA`）的`scheduler`或`run`方法，重新更新组件

以上就是基于`template`的响应式更新，还有`computed`,`watch`这些

## ref()
1. `ref()`会创建一个`RefImpl`对象
2. `_rawValue`是`ref()`的参数，即原始值
3. `_value`是被被`reactive()`进行代理过的值
```ts
export function ref(value?: unknown) {
  return createRef(value, false)
}
function createRef(rawValue: unknown, shallow: boolean) {
  if (isRef(rawValue)) {
    return rawValue
  }
  return new RefImpl(rawValue, shallow)
}
class RefImpl<T> {
  private _value: T
  private _rawValue: T

  public dep?: Dep = undefined
  public readonly __v_isRef = true

  constructor(value: T, public readonly __v_isShallow: boolean) {
	// toRaw: 根据一个 Vue 创建的代理返回其原始对象, _rawValue 则是原对象
    this._rawValue = __v_isShallow ? value : toRaw(value)
	// _value 被代理的值
    this._value = __v_isShallow ? value : toReactive(value)
  }

  get value() {
    trackRefValue(this)
    return this._value
  }

  set value(newVal) {
    const useDirectValue =
      this.__v_isShallow || isShallow(newVal) || isReadonly(newVal)
    newVal = useDirectValue ? newVal : toRaw(newVal)
    if (hasChanged(newVal, this._rawValue)) {
      this._rawValue = newVal
      this._value = useDirectValue ? newVal : toReactive(newVal)
      triggerRefValue(this, newVal)
    }
  }
}
```
4. 获取`value`时(例：我们在`template`中使用), 会创建一个`effect`的Set集合(`dep`)，调用[trackEffects](./effect.md)收集当前的`activeEffect`
```ts
// trackEffects 参考 './effect.md'
export function trackRefValue(ref: RefBase<any>) {
  if (shouldTrack && activeEffect) {
    ref = toRaw(ref)
    if (__DEV__) {
      trackEffects(ref.dep || (ref.dep = createDep()), {
        target: ref,
        type: TrackOpTypes.GET,
        key: 'value'
      })
    } else {
      trackEffects(ref.dep || (ref.dep = createDep()))
    }
  }
}
```
```ts
// dep是一个effect的集合
// wasTracked和newTracked维护了多级effect跟踪递归的状态。每个级别使用一个比特位来定义该依赖项是否被跟踪。
export const createDep = (effects?: ReactiveEffect[]): Dep => {
  const dep = new Set<ReactiveEffect>(effects) as Dep
  dep.w = 0 // wasTracked 
  dep.n = 0 // newTracked
  return dep
}
```
5. 设置`value`时，如果前后的值不一致，重新进行响应处理，调用[triggerEffects](./effect.md)
```ts
export function triggerRefValue(ref: RefBase<any>, newVal?: any) {
  ref = toRaw(ref)
  const dep = ref.dep
  if (dep) {
    if (__DEV__) {
      triggerEffects(dep, {
        target: ref,
        type: TriggerOpTypes.SET,
        key: 'value',
        newValue: newVal
      })
    } else {
      triggerEffects(dep)
    }
  }
}
```

## reactive()
1. 通过`createReactiveObject`创建代理对象并返回
2. 已当前的实例为key，缓存proxy，如果存在则用之前的
3. 默认已`mutableHandlers`为`handler`, 如果对象是集合类型, 则通过`mutableCollectionHandlers`作为`handler`创建proxy
4. 获取某个属性值时(例：我们在`template`中使用), 触发`proxy`的`get`，内部调用[track](./effect.md)
5. 设置某个属性值时，触发`proxy`的`set`,内部调用[trigger](./effect.md)
```ts
// packages/reactivity/src/reactive.ts
export function reactive(target: object) {
  // if trying to observe a readonly proxy, return the readonly version.
  if (isReadonly(target)) {
    return target
  }
  return createReactiveObject(
    target,
    false,
    mutableHandlers,
    mutableCollectionHandlers,
    reactiveMap
  )
}
```

```ts
function createReactiveObject(
  target: Target,
  isReadonly: boolean,
  baseHandlers: ProxyHandler<any>,
  collectionHandlers: ProxyHandler<any>,
  proxyMap: WeakMap<Target, any>
) {
  if (!isObject(target)) {
    if (__DEV__) {
      console.warn(`value cannot be made reactive: ${String(target)}`)
    }
    return target
  }
  // target is already a Proxy, return it.
  // exception: calling readonly() on a reactive object
  if (
    target[ReactiveFlags.RAW] &&
    !(isReadonly && target[ReactiveFlags.IS_REACTIVE])
  ) {
    return target
  }
  // target already has corresponding Proxy
  const existingProxy = proxyMap.get(target)
  if (existingProxy) {
    return existingProxy
  }
  // only specific value types can be observed.
  const targetType = getTargetType(target)
  if (targetType === TargetType.INVALID) {
    return target
  }
  const proxy = new Proxy(
    target,
    targetType === TargetType.COLLECTION ? collectionHandlers : baseHandlers
  )
  proxyMap.set(target, proxy)
  return proxy
}
```

#### baseHandlers
会对以下操作进行拦截
1. get: 拦截对象属性的读取，比如`proxy.foo`和`proxy['foo']`。
2. set: 拦截对象属性的设置，比如`proxy.foo = v`或`proxy['foo']` = v，返回一个布尔值。
3. deleteProperty: 拦截`delete proxy[propKey]`的操作，返回一个布尔值。
4. has: 拦截`propKey in proxy`的操作，返回一个布尔值。
5. ownKeys: 拦截`Object.getOwnPropertyNames(proxy)`、`Object.getOwnPropertySymbols(proxy)`、`Object.keys(proxy)`、`for...in`循环，返回一个数组。该方法返回目标对象所有自身的属性的属性名，而`Object.keys()`的返回结果仅包括目标对象自身的可遍历属性。
```ts
// packages/reactivity/src/baseHandlers.ts
export const mutableHandlers: ProxyHandler<object> = {
  get,
  set,
  deleteProperty,
  has,
  ownKeys,
}
```

```ts
// packages/reactivity/src/baseHandlers.ts
const get = /*#__PURE__*/ createGetter()

const set = /*#__PURE__*/ createSetter()

function deleteProperty(target: object, key: string | symbol): boolean {
  const hadKey = hasOwn(target, key)
  const oldValue = (target as any)[key]
  const result = Reflect.deleteProperty(target, key)
  if (result && hadKey) {
    // 触发更新 在./effect.md内说明
    trigger(target, TriggerOpTypes.DELETE, key, undefined, oldValue)
  }
  return result
}

function has(target: object, key: string | symbol): boolean {
  const result = Reflect.has(target, key)
  if (!isSymbol(key) || !builtInSymbols.has(key)) {
    // 收集effect 在./effect.md内说明
    track(target, TrackOpTypes.HAS, key)
  }
  return result
}

function ownKeys(target: object): (string | symbol)[] {
  // 收集effect 在./effect.md内说明
  track(target, TrackOpTypes.ITERATE, isArray(target) ? 'length' : ITERATE_KEY)
  return Reflect.ownKeys(target)
}

```

#### createGetter
```ts
function createGetter(isReadonly = false, shallow = false) {
  return function get(target: Target, key: string | symbol, receiver: object) {
    if (key === ReactiveFlags.IS_REACTIVE) {
      return !isReadonly
    } else if (key === ReactiveFlags.IS_READONLY) {
      return isReadonly
    } else if (key === ReactiveFlags.IS_SHALLOW) {
      return shallow
    } else if (
      key === ReactiveFlags.RAW &&
      receiver ===
        (isReadonly
          ? shallow
            ? shallowReadonlyMap
            : readonlyMap
          : shallow
          ? shallowReactiveMap
          : reactiveMap
        ).get(target)
    ) {
      return target
    }

    const targetIsArray = isArray(target)

    if (!isReadonly) {
      if (targetIsArray && hasOwn(arrayInstrumentations, key)) {
        return Reflect.get(arrayInstrumentations, key, receiver)
      }
      if (key === 'hasOwnProperty') {
        return hasOwnProperty
      }
    }

    const res = Reflect.get(target, key, receiver)

    if (isSymbol(key) ? builtInSymbols.has(key) : isNonTrackableKeys(key)) {
      return res
    }

    if (!isReadonly) {
      track(target, TrackOpTypes.GET, key)
    }

    if (shallow) {
      return res
    }

    if (isRef(res)) {
      // ref unwrapping - skip unwrap for Array + integer key.
      return targetIsArray && isIntegerKey(key) ? res : res.value
    }

    if (isObject(res)) {
      // Convert returned value into a proxy as well. we do the isObject check
      // here to avoid invalid value warning. Also need to lazy access readonly
      // and reactive here to avoid circular dependency.
      return isReadonly ? readonly(res) : reactive(res)
    }

    return res
  }
}
```

#### createSetter
```ts
function createSetter(shallow = false) {
  return function set(
    target: object,
    key: string | symbol,
    value: unknown,
    receiver: object
  ): boolean {
    let oldValue = (target as any)[key]
    if (isReadonly(oldValue) && isRef(oldValue) && !isRef(value)) {
      return false
    }
    if (!shallow) {
      if (!isShallow(value) && !isReadonly(value)) {
        oldValue = toRaw(oldValue)
        value = toRaw(value)
      }
      if (!isArray(target) && isRef(oldValue) && !isRef(value)) {
        oldValue.value = value
        return true
      }
    } else {
      // in shallow mode, objects are set as-is regardless of reactive or not
    }

    const hadKey =
      isArray(target) && isIntegerKey(key)
        ? Number(key) < target.length
        : hasOwn(target, key)
    const result = Reflect.set(target, key, value, receiver)
    // don't trigger if target is something up in the prototype chain of original
    if (target === toRaw(receiver)) {
      // 原来的对象没有这个key
      if (!hadKey) {
        trigger(target, TriggerOpTypes.ADD, key, value)
      } else if (hasChanged(value, oldValue)) { // 值发生改变
        trigger(target, TriggerOpTypes.SET, key, value, oldValue)
      }
    }
    return result
  }
}
```
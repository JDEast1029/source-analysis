## 各类Hook生成的方法

## 同步的Hook

同步的`Hook`只能使用`tap`和`call`

### SyncHook(同步)

#### `call`代码示例

```js
const syncHook = new SyncHook(['value'])
syncHook.tap('syncHookPlugin', (value) => {
    console.log('syncHook', value);
})
syncHook.tap('syncHookPlugin', (value) => {
    console.log('syncHook', value);
})
syncHook.call(0)
```

#### `call`代码逻辑

```js
function anonymous(value) {
    "use strict";
    var _context;
    var _x = this._x;
    var _fn0 = _x[0];
    _fn0(value);
    var _fn1 = _x[1];
    _fn1(value);
}
```

### SyncBailHook(同步熔断)

#### `call`代码示例

```js
const syncBailHook = new SyncBailHook(['value1', 'value2'])

syncBailHook.tap('syncBailHookPlugin', (value1, value2) => {
    console.log('syncBailHookPlugin', value1, value2);
})
syncBailHook.tap('syncBailHookPlugin1', (value1, value2) => {
    console.log('syncBailHookPlugin1', value1, value2);
})

syncBailHook.call('传值1', '传值2')
```

#### `call`代码逻辑

```js
// 多个tap，多个嵌套，有一个tap的回有返回值，后面的tap回调都不会执行
function anonymous(SyncBailHookPlugin) {
    "use strict";
    var _context;
    var _x = this._x;
    var _fn0 = _x[0];
    var _result0 = _fn0(SyncBailHookPlugin);
    if (_result0 !== undefined) {
        return _result0;
        ;
    } else {
        var _fn1 = _x[1];
        var _result1 = _fn1(SyncBailHookPlugin);
        if (_result1 !== undefined) {
            return _result1;
            ;
        } else {
        }
    }

}
```

### SyncWaterfallHook(同步瀑布流)

#### `call`代码示例

```js
const syncWaterfallHook = new SyncWaterfallHook(['value'])
syncWaterfallHook.tap('syncWaterfallHookPlugin', (value) => {
    console.log(`我是第${value}个`);
    return value + 1;
})
syncWaterfallHook.tap('syncWaterfallHookPlugin', (value) => {
    console.log(`我是第${value}个`);
    return value + 1;
})
const res = syncWaterfallHook.call(0)
console.log(`当前是第${res}个`)
// 我是第0个
// 我是第1个
// 当前是第2个
```

#### `call`代码逻辑

```js
// 瀑布流Hook,当前钩子的参数为上一个钩子返回的结果，
// 若果上一个的返回结果为undefined,则沿用上上一个的
function anonymous(value) {
    "use strict";
    var _context;
    var _x = this._x;
    var _fn0 = _x[0];
    var _result0 = _fn0(value);
    if (_result0 !== undefined) {
        value = _result0;
    }
    var _fn1 = _x[1];
    var _result1 = _fn1(value);
    if (_result1 !== undefined) {
        value = _result1;
    }
    return value;

}
```

### SyncLoopHook(同步循环)

#### `call`代码示例

```js
const syncLoopHook = new SyncLoopHook(['value'])
syncLoopHook.tap('SyncLoopHookPlugin', (value) => {
    console.log(`我是第${value}个`);
})
syncLoopHook.tap('SyncLoopHookPlugin', (value) => {
    console.log(`我是第${value}个`);
})
syncLoopHook.call(0)
```

#### `call`代码逻辑

```js
// loop循环，当钩子返回结果不为undefined时，轮询结束,
// 不然会一直循环调用钩子，陷入死循环
function anonymous(value) {
    "use strict";
    var _context;
    var _x = this._x;
    var _loop;
    do {
        _loop = false;
        var _fn0 = _x[0];
        var _result0 = _fn0(value);
        if (_result0 !== undefined) {
            _loop = true;
        } else {
            var _fn1 = _x[1];
            var _result1 = _fn1(value);
            if (_result1 !== undefined) {
                _loop = true;
            } else {
                if (!_loop) {
                }
            }
        }
    } while (_loop);

}
```

## 异步的Hook

异步的`Hook`可以使用`tapAsync`和`callAsync`；`tapPromise`和`promise`

### AsyncParallelHook(异步并行)

#### `callAsync`代码示例

```js
const asyncParallelHook = new AsyncParallelHook(['value'])
asyncParallelHook.tapAsync('AsyncParallelHookPlugin', (value, callback) => {
    new Promise((r, j) => {
        setTimeout(() => {
            if (~~(Math.random() * 10) > 5) {
                console.log(`我是1号Tap第${value}个`);
                r('成功了')
            } else {
                j('小于5失败了')
            }
        }, 1000);
    }).catch((err) => callback(err))

})
asyncParallelHook.tapAsync('AsyncParallelHookPlugin', (value, callback) => {
    console.log(`我是2号Tap第${value}个`);
})
asyncParallelHook.callAsync(0, (err) => {
    console.log('结束');
    err && console.log('err', err);
})


/** 结果 **/
// 我是2号Tap第0个
// 我是1号Tap第0个
// or
// 结束
// err 小于5失败了
```

#### `callAsync`代码逻辑

```js
// 同时执行钩子，钩子会额外传入一个`callback`参数，
// 如果某个钩子内部调用了callback，内部计数_counter = 0，后面还未执行的fn将不会执行
// 并且callAsync传入`_callback`会被执行；由于内部没有校验_callback的空值，这个函数必传
function anonymous(value, _callback) {
    "use strict";
    var _context;
    var _x = this._x;
    do {
        var _counter = 2;
        var _done = (function () {
            _callback();
        });
        if (_counter <= 0) break;
        var _fn0 = _x[0];
        _fn0(value, (function (_err0) {
            if (_err0) {
                if (_counter > 0) {
                    _callback(_err0);
                    _counter = 0;
                }
            } else {
                if (--_counter === 0) _done();
            }
        }));
        if (_counter <= 0) break;
        var _fn1 = _x[1];
        _fn1(value, (function (_err1) {
            if (_err1) {
                if (_counter > 0) {
                    _callback(_err1);
                    _counter = 0;
                }
            } else {
                if (--_counter === 0) _done();
            }
        }));
    } while (false);

}
```

#### `promise`代码示例

```js
const asyncParallelHook = new AsyncParallelHook(['value'])
// 必须return promise
asyncParallelHook.tapPromise('AsyncParallelHookPlugin', (value) => {
    return new Promise((r, j) => {
        setTimeout(() => {
            if (~~(Math.random() * 10) > 5) {
                console.log(`我是1号Tap第${value}个`);
                r('成功了')
            } else {
                j('小于5失败了')
            }
        }, 1000);
    });
})
asyncParallelHook.tapPromise('AsyncParallelHookPlugin', (value) => {
    return new Promise((r, j) => {
        setTimeout(() => {
            if (~~(Math.random() * 10) <= 5) {
                console.log(`我是2号Tap第${value}个`);
                r('成功了')
            } else {
                j('大于5失败了')
            }
        }, 500);
    });
})
asyncParallelHook.promise(0).then((res) => {

}).catch((err) => {
    console.log('err', err);
})
```

#### `promise`代码逻辑

```js
// 同时执行钩子，注册的钩子必须返回promise
// 如果钩子中的promise reject了，asyncParallelHook.promise.catch会被执行;所有钩子执行完毕后，会执行asyncParallelHook.promise.then
function anonymous(value) {
    "use strict";
    var _context;
    var _x = this._x;
    return new Promise((function (_resolve, _reject) {
        var _sync = true;
        function _error(_err) {
            if (_sync)
                _resolve(Promise.resolve().then((function () { throw _err; })));
            else
                _reject(_err);
        };
        do {
            var _counter = 2;
            var _done = (function () {
                _resolve();
            });
            if (_counter <= 0) break;
            var _fn0 = _x[0];
            var _hasResult0 = false;
            var _promise0 = _fn0(value);
            if (!_promise0 || !_promise0.then)
                throw new Error('Tap function (tapPromise) did not return promise (returned ' + _promise0 + ')');
            _promise0.then((function (_result0) {
                _hasResult0 = true;
                if (--_counter === 0) _done();
            }), function (_err0) {
                if (_hasResult0) throw _err0;
                if (_counter > 0) {
                    _error(_err0);
                    _counter = 0;
                }
            });
            if (_counter <= 0) break;
            var _fn1 = _x[1];
            var _hasResult1 = false;
            var _promise1 = _fn1(value);
            if (!_promise1 || !_promise1.then)
                throw new Error('Tap function (tapPromise) did not return promise (returned ' + _promise1 + ')');
            _promise1.then((function (_result1) {
                _hasResult1 = true;
                if (--_counter === 0) _done();
            }), function (_err1) {
                if (_hasResult1) throw _err1;
                if (_counter > 0) {
                    _error(_err1);
                    _counter = 0;
                }
            });
        } while (false);
        _sync = false;
    }));
}
```

### AsyncParallelBailHook(异步并行熔断)

#### `callAsync`代码示例

```js
const asyncParallelBailHook = new AsyncParallelBailHook(['value'])
// 必须return promise
asyncParallelBailHook.tapAsync('AsyncParallelBailHookPlugin', (value, callback) => {
    new Promise((r, j) => {
        setTimeout(() => {
            if (~~(Math.random() * 10) > 5) {
                console.log(`我是1号Tap第${value}个`);
                r('成功了')
            } else {
                j('小于5失败了')
            }
        }, 1000);
    }).then(
        (res) => callback(null, res),
        (err) => callback(err)
    );
})
asyncParallelBailHook.tapAsync('AsyncParallelBailHookPlugin', (value, callback) => {
    console.log(`我是2号Tap第${value}个`);
})
asyncParallelBailHook.callAsync(0, (err, result) => {
    console.log('结束');
    err && console.log('err', err);
})
```

#### `callAsync`代码逻辑

```js
/**
 * 同时执行钩子_fn[i]，钩子会额外传入一个callback: (err, result) => void
 * 如果callback传递的err有值，会将_counter=0；后面的callback内的逻辑将不会执行，_callback中的回调拿到_err
 * 如果callback传递的err为null，res有值，会将_counter=0；后面的callback内的逻辑将不会执行，_callback中的回调拿到result
 * 如果callback传递的err为null，res没有值，后面的callback逻辑继续执行，最后执行_callback,没有参数
 */
function anonymous(value, _callback) {
    "use strict";
    var _context;
    var _x = this._x;
    var _results = new Array(2);
    var _checkDone = function () {
        for (var i = 0; i < _results.length; i++) {
            var item = _results[i];
            if (item === undefined) return false;
            if (item.result !== undefined) {
                _callback(null, item.result);
                return true;
            }
            if (item.error) {
                _callback(item.error);
                return true;
            }
        }
        return false;
    }
    do {
        var _counter = 2;
        var _done = (function () {
            _callback();
        });
        if (_counter <= 0) break;
        var _fn0 = _x[0];
        _fn0(value, (function (_err0, _result0) {
            if (_err0) {
                if (_counter > 0) {
                    if (0 < _results.length && ((_results.length = 1), (_results[0] = { error: _err0 }), _checkDone())) {
                        _counter = 0;
                    } else {
                        if (--_counter === 0) _done();
                    }
                }
            } else {
                if (_counter > 0) {
                    if (0 < _results.length && (_result0 !== undefined && (_results.length = 1), (_results[0] = { result: _result0 }), _checkDone())) {
                        _counter = 0;
                    } else {
                        if (--_counter === 0) _done();
                    }
                }
            }
        }));
        if (_counter <= 0) break;
        if (1 >= _results.length) {
            if (--_counter === 0) _done();
        } else {
            var _fn1 = _x[1];
            _fn1(value, (function (_err1, _result1) {
                if (_err1) {
                    if (_counter > 0) {
                        if (1 < _results.length && ((_results.length = 2), (_results[1] = { error: _err1 }), _checkDone())) {
                            _counter = 0;
                        } else {
                            if (--_counter === 0) _done();
                        }
                    }
                } else {
                    if (_counter > 0) {
                        if (1 < _results.length && (_result1 !== undefined && (_results.length = 2), (_results[1] = { result: _result1 }), _checkDone())) {
                            _counter = 0;
                        } else {
                            if (--_counter === 0) _done();
                        }
                    }
                }
            }));
        }
    } while (false);
}
```

#### `promise`代码示例

```js
const asyncParallelBailHook = new AsyncParallelBailHook(['value'])
// 必须return promise
asyncParallelBailHook.tapPromise('AsyncParallelBailHookPlugin', (value) => {
    return new Promise((r, j) => {
        setTimeout(() => {
            if (~~(Math.random() * 10) > 5) {
                console.log(`我是1号Tap第${value}个`);
                r()
            } else {
                j('小于5失败了')
            }
        }, 1000);

    });
})
asyncParallelBailHook.tapPromise('AsyncParallelBailHookPlugin', (value) => {
    return new Promise((r, j) => {
        setTimeout(() => {
            if (~~(Math.random() * 10) <= 5) {
                console.log(`我是2号Tap第${value}个`);
                r('来自2号Tap熔断')
            } else {
                j('大于5失败了')
            }
        }, 1000);
    });
})
asyncParallelBailHook.promise(0).then((res) => {
    console.log('res', res);
}).catch((err) => {
    console.log('err', err);
})
```

#### `promise`代码逻辑

```js
/**
 * 同时执行钩子，注册的钩子必须返回promise
 * 钩子中的promise.resolve，有值时，会将_counter = 0; 后面钩子的promise.then不会执行，并在asyncParallelBailHook.promise.then中获得参数
 * 钩子中的promise.reject, 会将_counter = 0; 后面钩子的promise.then不会执行，并在asyncParallelBailHook.promise.catch中获得参数
 */
function anonymous(value) {
    "use strict";
    var _context;
    var _x = this._x;
    return new Promise((function (_resolve, _reject) {
        var _sync = true;
        function _error(_err) {
            if (_sync)
                _resolve(Promise.resolve().then((function () { throw _err; })));
            else
                _reject(_err);
        };
        var _results = new Array(2);
        var _checkDone = function () {
            for (var i = 0; i < _results.length; i++) {
                var item = _results[i];
                if (item === undefined) return false;
                if (item.result !== undefined) {
                    _resolve(item.result);
                    return true;
                }
                if (item.error) {
                    _error(item.error);
                    return true;
                }
            }
            return false;
        }
        do {
            var _counter = 2;
            var _done = (function () {
                _resolve();
            });
            if (_counter <= 0) break;
            var _fn0 = _x[0];
            var _hasResult0 = false;
            var _promise0 = _fn0(value);
            if (!_promise0 || !_promise0.then)
                throw new Error('Tap function (tapPromise) did not return promise (returned ' + _promise0 + ')');
            _promise0.then((function (_result0) {
                _hasResult0 = true;
                if (_counter > 0) {
                    if (0 < _results.length && (_result0 !== undefined && (_results.length = 1), (_results[0] = { result: _result0 }), _checkDone())) {
                        _counter = 0;
                    } else {
                        if (--_counter === 0) _done();
                    }
                }
            }), function (_err0) {
                if (_hasResult0) throw _err0;
                if (_counter > 0) {
                    if (0 < _results.length && ((_results.length = 1), (_results[0] = { error: _err0 }), _checkDone())) {
                        _counter = 0;
                    } else {
                        if (--_counter === 0) _done();
                    }
                }
            });
            if (_counter <= 0) break;
            if (1 >= _results.length) {
                if (--_counter === 0) _done();
            } else {
                var _fn1 = _x[1];
                var _hasResult1 = false;
                var _promise1 = _fn1(value);
                if (!_promise1 || !_promise1.then)
                    throw new Error('Tap function (tapPromise) did not return promise (returned ' + _promise1 + ')');
                _promise1.then((function (_result1) {
                    _hasResult1 = true;
                    if (_counter > 0) {
                        if (1 < _results.length && (_result1 !== undefined && (_results.length = 2), (_results[1] = { result: _result1 }), _checkDone())) {
                            _counter = 0;
                        } else {
                            if (--_counter === 0) _done();
                        }
                    }
                }), function (_err1) {
                    if (_hasResult1) throw _err1;
                    if (_counter > 0) {
                        if (1 < _results.length && ((_results.length = 2), (_results[1] = { error: _err1 }), _checkDone())) {
                            _counter = 0;
                        } else {
                            if (--_counter === 0) _done();
                        }
                    }
                });
            }
        } while (false);
        _sync = false;
    }));
}
```

### AsyncSeriesHook(异步串行)

#### `callAsync`代码示例

```js
const asyncSeriesHook = new AsyncSeriesHook(['value']);
asyncSeriesHook.tapAsync('AsyncSeriesHookPlugin', (value, callback) => {
    console.log(value);
    callback('我是第1个error')
});
asyncSeriesHook.tapAsync('AsyncSeriesHookPlugin', (value, callback) => {
    console.log(value);
    callback()
});
asyncSeriesHook.callAsync(0, (error) => {
    console.log('error', error);
})
```

#### `callAsync`代码逻辑

```js
/**
 * 异步串行, 钩子的callback需要执行，才会触发下一个钩子，
 * 如果callback传递参数，则下一个钩子不会执行，参数会被_callback接收
 */
function anonymous(value, _callback) {
    "use strict";
    var _context;
    var _x = this._x;
    function _next0() {
        var _fn1 = _x[1];
        _fn1(value, (function (_err1) {
            if (_err1) {
                _callback(_err1);
            } else {
                _callback();
            }
        }));
    }
    var _fn0 = _x[0];
    _fn0(value, (function (_err0) {
        if (_err0) {
            _callback(_err0);
        } else {
            _next0();
        }
    }));
}
```

#### `promise`代码示例

```js
const asyncSeriesHook = new AsyncSeriesHook(['value']);
asyncSeriesHook.tapPromise('AsyncSeriesHookPlugin', (value) => {
    return new Promise((r, j) => {
        setTimeout(() => {
            if (~~(Math.random() * 10) > 5) {
                console.log(`我是1号Tap`);
                r()
            } else {
                j('小于5失败了')
            }
        }, 1000);

    });
});
asyncSeriesHook.tapPromise('AsyncSeriesHookPlugin', (value) => {
    return new Promise((r, j) => {
        setTimeout(() => {
            if (~~(Math.random() * 10) <= 5) {
                console.log(`我是2号Tap`);
                r('来自2号Tap熔断')
            } else {
                j('大于5失败了')
            }
        }, 1000);
    });
});
asyncSeriesHook.promise(0).then(() => {
    // 无参数
    console.log('结束');
}).catch((err) => {
    console.log('错误', err);
})
```

#### `promise`代码逻辑

```js
/**
 * 异步串行， 钩子需要返回promise
 * promise.resolve, 下一个钩子才会执行
 * promise.reject, 后面钩子会被中断，错误在asyncSeriesHook.promise().catch中捕获
 */
function anonymous(value) {
    "use strict";
    var _context;
    var _x = this._x;
    return new Promise((function (_resolve, _reject) {
        var _sync = true;
        function _error(_err) {
            if (_sync)
                _resolve(Promise.resolve().then((function () { throw _err; })));
            else
                _reject(_err);
        };
        function _next0() {
            var _fn1 = _x[1];
            var _hasResult1 = false;
            var _promise1 = _fn1(value);
            if (!_promise1 || !_promise1.then)
                throw new Error('Tap function (tapPromise) did not return promise (returned ' + _promise1 + ')');
            _promise1.then((function (_result1) {
                _hasResult1 = true;
                _resolve();
            }), function (_err1) {
                if (_hasResult1) throw _err1;
                _error(_err1);
            });
        }
        var _fn0 = _x[0];
        var _hasResult0 = false;
        var _promise0 = _fn0(value);
        if (!_promise0 || !_promise0.then)
            throw new Error('Tap function (tapPromise) did not return promise (returned ' + _promise0 + ')');
        _promise0.then((function (_result0) {
            _hasResult0 = true;
            _next0();
        }), function (_err0) {
            if (_hasResult0) throw _err0;
            _error(_err0);
        });
        _sync = false;
    }));
}
```

### AsyncSeriesBailHook(异步串行熔断)

#### `callAsync`代码示例

```js
const asyncSeriesBailHook = new AsyncSeriesBailHook(['value']);
asyncSeriesBailHook.tapAsync('AsyncSeriesBailHookPlugin', (value, callback) => {
    console.log(value);
    callback('我是第1个error')
});
asyncSeriesBailHook.tapAsync('AsyncSeriesBailHookPlugin', (value, callback) => {
    console.log(value);
    callback(null, '我是第二个')
});
asyncSeriesBailHook.callAsync(0, (error, result) => {
    console.log('error', error);
    console.log('result', result);
})
```

#### `callAsync`代码逻辑

```js
/**
 * 异步串行熔断, 钩子的callback需要执行，才会触发下一个钩子，
 * 如果callback传递error，则下一个钩子不会执行，error会被_callback接收
 * 如果callback传递error=null, result有值，则下一个钩子不会执行，result会被_callback接收
 * 如果callback没有传递参数，则继续执行下一个钩子
 */
function anonymous(value, _callback) {
    "use strict";
    var _context;
    var _x = this._x;
    function _next0() {
        var _fn1 = _x[1];
        _fn1(value, (function (_err1, _result1) {
            if (_err1) {
                _callback(_err1);
            } else {
                if (_result1 !== undefined) {
                    _callback(null, _result1);

                } else {
                    _callback();
                }
            }
        }));
    }
    var _fn0 = _x[0];
    _fn0(value, (function (_err0, _result0) {
        if (_err0) {
            _callback(_err0);
        } else {
            if (_result0 !== undefined) {
                _callback(null, _result0);

            } else {
                _next0();
            }
        }
    }));

}
```

#### `promise`代码示例

```js
const asyncSeriesBailHook = new AsyncSeriesBailHook(['value']);
asyncSeriesBailHook.tapPromise('AsyncSeriesBailHookPlugin', (value) => {
    return new Promise((r, j) => {
        setTimeout(() => {
            if (~~(Math.random() * 10) > 5) {
                console.log(`我是1号Tap`);
                r()
            } else {
                j('小于5失败了')
            }
        }, 1000);

    });
});
asyncSeriesBailHook.tapPromise('AsyncSeriesBailHookPlugin', (value) => {
    return new Promise((r, j) => {
        setTimeout(() => {
            if (~~(Math.random() * 10) <= 5) {
                console.log(`我是2号Tap`);
                r('来自2号Tap熔断')
            } else {
                j('大于5失败了')
            }
        }, 1000);
    });
});
asyncSeriesBailHook.promise(0).then(() => {
    // 无参数
    console.log('结束');
}).catch((err) => {
    console.log('错误', err);
})
```

#### `promise`代码逻辑

```js
/**
 * 异步串行熔断，钩子需要返回promise
 * promise.resolve, 有值，下一个钩子不会执行，asyncSeriesBailHook.promise.then 会接收这个参数；无值，下一个钩子继续执行
 * promise.reject, 下一个钩子不会 执行，asyncSeriesBailHook.promise.catch接收报错
 */
function anonymous(value) {
    "use strict";
    var _context;
    var _x = this._x;
    return new Promise((function (_resolve, _reject) {
        var _sync = true;
        function _error(_err) {
            if (_sync)
                _resolve(Promise.resolve().then((function () { throw _err; })));
            else
                _reject(_err);
        };
        function _next0() {
            var _fn1 = _x[1];
            var _hasResult1 = false;
            var _promise1 = _fn1(value);
            if (!_promise1 || !_promise1.then)
                throw new Error('Tap function (tapPromise) did not return promise (returned ' + _promise1 + ')');
            _promise1.then((function (_result1) {
                _hasResult1 = true;
                if (_result1 !== undefined) {
                    _resolve(_result1);

                } else {
                    _resolve();
                }
            }), function (_err1) {
                if (_hasResult1) throw _err1;
                _error(_err1);
            });
        }
        var _fn0 = _x[0];
        var _hasResult0 = false;
        var _promise0 = _fn0(value);
        if (!_promise0 || !_promise0.then)
            throw new Error('Tap function (tapPromise) did not return promise (returned ' + _promise0 + ')');
        _promise0.then((function (_result0) {
            _hasResult0 = true;
            if (_result0 !== undefined) {
                _resolve(_result0);

            } else {
                _next0();
            }
        }), function (_err0) {
            if (_hasResult0) throw _err0;
            _error(_err0);
        });
        _sync = false;
    }));
}
```

### AsyncSeriesLoopHook(异步串行轮询)

#### `callAsync`代码示例

```js
const asyncSeriesLoopHook = new AsyncSeriesLoopHook(['value']);
asyncSeriesLoopHook.tapAsync('AsyncSeriesLoopHookPlugin', (value, callback) => {
    console.log(value);
    callback('我是第1个error')
});
asyncSeriesLoopHook.tapAsync('AsyncSeriesLoopHookPlugin', (value, callback) => {
    console.log(value);
    callback(null, '我是第二个')
});
asyncSeriesLoopHook.callAsync(0, (error, result) => {
    console.log('error', error);
    console.log('result', result);
})
```

#### `callAsync`代码逻辑

```js
/**
 * 异步串行轮询, 钩子的callback需要执行，才会触发下一个钩子，
 * 如果callback传递error，则下一个钩子不会执行，error会被_callback接收
 * 如果callback传递error=null, result有值，则下一个钩子不会执行，会重新执行_looper，直到当前callback返回错误或者result为undefined
 * 如果callback没有error=null, result=undefined，则继续执行下一个钩子
 */
function anonymous(value, _callback) {
    "use strict";
    var _context;
    var _x = this._x;
    var _looper = (function () {
        var _loopAsync = false;
        var _loop;
        do {
            _loop = false;
            function _next0() {
                var _fn1 = _x[1];
                _fn1(value, (function (_err1, _result1) {
                    if (_err1) {
                        _callback(_err1);
                    } else {
                        if (_result1 !== undefined) {
                            _loop = true;
                            if (_loopAsync) _looper();
                        } else {
                            if (!_loop) {
                                _callback();
                            }
                        }
                    }
                }));
            }
            var _fn0 = _x[0];
            _fn0(value, (function (_err0, _result0) {
                if (_err0) {
                    _callback(_err0);
                } else {
                    if (_result0 !== undefined) {
                        _loop = true;
                        if (_loopAsync) _looper();
                    } else {
                        _next0();
                    }
                }
            }));
        } while (_loop);
        _loopAsync = true;
    });
    _looper();
}
```

#### `promise`代码示例

```js
const asyncSeriesLoopHook = new AsyncSeriesLoopHook(['value']);
asyncSeriesLoopHook.tapPromise('asyncSeriesLoopHookPlugin', (value) => {
    return new Promise((r, j) => {
        setTimeout(() => {
            if (~~(Math.random() * 10) > 5) {
                console.log(`我是1号Tap`);
                r()
            } else {
                j('小于5失败了')
            }
        }, 1000);

    });
});
asyncSeriesLoopHook.tapPromise('asyncSeriesLoopHookPlugin', (value) => {
    return new Promise((r, j) => {
        setTimeout(() => {
            if (~~(Math.random() * 10) <= 5) {
                console.log(`我是2号Tap`);
                r('来自2号Tap熔断')
            } else {
                j('大于5失败了')
            }
        }, 1000);
    });
});
asyncSeriesLoopHook.promise(0).then(() => {
    // 无参数
    console.log('结束');
}).catch((err) => {
    console.log('错误', err);
})
```

#### `promise`代码逻辑

```js
/**
 * 异步串行轮询， 钩子需要返回promise
 * promise.resolve, 有值，下一个钩子不会执行，会重新执行_looper，直到promise.resolve无值，或者promise.catch；无值，下一个钩子继续执行，最后asyncSeriesLoopHook.promise.then执行
 * promise.reject, 下一个钩子不会执行，asyncSeriesLoopHook.promise.catch接收报错
 */
function anonymous(value) {
    "use strict";
    var _context;
    var _x = this._x;
    return new Promise((function (_resolve, _reject) {
        var _sync = true;
        function _error(_err) {
            if (_sync)
                _resolve(Promise.resolve().then((function () { throw _err; })));
            else
                _reject(_err);
        };
        var _looper = (function () {
            var _loopAsync = false;
            var _loop;
            do {
                _loop = false;
                function _next0() {
                    var _fn1 = _x[1];
                    var _hasResult1 = false;
                    var _promise1 = _fn1(value);
                    if (!_promise1 || !_promise1.then)
                        throw new Error('Tap function (tapPromise) did not return promise (returned ' + _promise1 + ')');
                    _promise1.then((function (_result1) {
                        _hasResult1 = true;
                        if (_result1 !== undefined) {
                            _loop = true;
                            if (_loopAsync) _looper();
                        } else {
                            if (!_loop) {
                                _resolve();
                            }
                        }
                    }), function (_err1) {
                        if (_hasResult1) throw _err1;
                        _error(_err1);
                    });
                }
                var _fn0 = _x[0];
                var _hasResult0 = false;
                var _promise0 = _fn0(value);
                if (!_promise0 || !_promise0.then)
                    throw new Error('Tap function (tapPromise) did not return promise (returned ' + _promise0 + ')');
                _promise0.then((function (_result0) {
                    _hasResult0 = true;
                    if (_result0 !== undefined) {
                        _loop = true;
                        if (_loopAsync) _looper();
                    } else {
                        _next0();
                    }
                }), function (_err0) {
                    if (_hasResult0) throw _err0;
                    _error(_err0);
                });
            } while (_loop);
            _loopAsync = true;
        });
        _looper();
        _sync = false;
    }));

}
```

### AsyncSeriesWaterfallHook(异步串行瀑布流)

#### `callAsync`代码示例

```js
const asyncSeriesWaterfallHook = new AsyncSeriesWaterfallHook(['value']);
asyncSeriesWaterfallHook.tapAsync('AsyncSeriesWaterfallHookPlugin', (value, callback) => {
    console.log(value);
    callback('我是第1个error')
});
asyncSeriesWaterfallHook.tapAsync('AsyncSeriesWaterfallHookPlugin', (value, callback) => {
    console.log(value);
    callback(null, '我是第二个')
});
asyncSeriesWaterfallHook.callAsync(0, (error, result) => {
    console.log('error', error);
    console.log('result', result);
})
```

#### `callAsync`代码逻辑

```js
/**
 * 异步串行瀑布流，钩子的callback需要执行，才会触发下一个钩子，
 * 当前钩子的参数为上一个钩子返回的结果，
 * 若果上一个的返回结果为undefined,则沿用上上一个的，
 * callback(有值)，_callback接收错误
 * callback(null, undefined)，_callback接收最后的value
 */
function anonymous(value, _callback) {
    "use strict";
    var _context;
    var _x = this._x;
    function _next0() {
        var _fn1 = _x[1];
        _fn1(value, (function (_err1, _result1) {
            if (_err1) {
                _callback(_err1);
            } else {
                if (_result1 !== undefined) {
                    value = _result1;
                }
                _callback(null, value);
            }
        }));
    }
    var _fn0 = _x[0];
    _fn0(value, (function (_err0, _result0) {
        if (_err0) {
            _callback(_err0);
        } else {
            if (_result0 !== undefined) {
                value = _result0;
            }
            _next0();
        }
    }));
}
```

#### `promise`代码示例

```js
const asyncSeriesWaterfallHook = new AsyncSeriesWaterfallHook(['value']);
asyncSeriesWaterfallHook.tapPromise('AsyncSeriesWaterfallHookPlugin', (value) => {
    return new Promise((r, j) => {
        setTimeout(() => {
            if (~~(Math.random() * 10) > 5) {
                console.log(`我是1号Tap`);
                r()
            } else {
                j('小于5失败了')
            }
        }, 1000);

    });
});
asyncSeriesWaterfallHook.tapPromise('AsyncSeriesWaterfallHookPlugin', (value) => {
    return new Promise((r, j) => {
        setTimeout(() => {
            if (~~(Math.random() * 10) <= 5) {
                console.log(`我是2号Tap`);
                r('来自2号Tap熔断')
            } else {
                j('大于5失败了')
            }
        }, 1000);
    });
});
asyncSeriesWaterfallHook.promise(0).then(() => {
    // 无参数
    console.log('结束');
}).catch((err) => {
    console.log('错误', err);
})
```

#### `promise`代码逻辑

```js
/**
 * 异步串行瀑布流，钩子需要返回promise
 * 当前钩子的参数为上一个钩子promise.resolve的结果，
 * promise.resolve, 无值继续执行下个钩子，最后asyncSeriesWaterfallHook.promise.then接收最后的value
 * promise.reject, 下一个钩子不会执行，asyncSeriesWaterfallHook.promise.catch接收报错
 */
function anonymous(value) {
    "use strict";
    var _context;
    var _x = this._x;
    return new Promise((function (_resolve, _reject) {
        var _sync = true;
        function _error(_err) {
            if (_sync)
                _resolve(Promise.resolve().then((function () { throw _err; })));
            else
                _reject(_err);
        };
        function _next0() {
            var _fn1 = _x[1];
            var _hasResult1 = false;
            var _promise1 = _fn1(value);
            if (!_promise1 || !_promise1.then)
                throw new Error('Tap function (tapPromise) did not return promise (returned ' + _promise1 + ')');
            _promise1.then((function (_result1) {
                _hasResult1 = true;
                if (_result1 !== undefined) {
                    value = _result1;
                }
                _resolve(value);
            }), function (_err1) {
                if (_hasResult1) throw _err1;
                _error(_err1);
            });
        }
        var _fn0 = _x[0];
        var _hasResult0 = false;
        var _promise0 = _fn0(value);
        if (!_promise0 || !_promise0.then)
            throw new Error('Tap function (tapPromise) did not return promise (returned ' + _promise0 + ')');
        _promise0.then((function (_result0) {
            _hasResult0 = true;
            if (_result0 !== undefined) {
                value = _result0;
            }
            _next0();
        }), function (_err0) {
            if (_hasResult0) throw _err0;
            _error(_err0);
        });
        _sync = false;
    }));
}
```
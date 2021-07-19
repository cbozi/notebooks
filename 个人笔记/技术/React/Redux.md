Redux


Redux核心库的代码很少，提供的api也就下面几个

**Top-Level Exports#**
- createStore(reducer, [preloadedState], [enhancer])
- combineReducers(reducers)
- applyMiddleware(...middlewares)
- bindActionCreators(actionCreators, dispatch)
- compose(...functions)

**Store API#**
- Store
  - getState()
  - dispatch(action)
  - subscribe(listener)
  - replaceReducer(nextReducer)

核心逻辑就在createStore里面

createStore首先初始化的时候就是处理了一下可选参数，如果有middleware就应用middleware，后面再分析middleware机制。

然后就是主要保证了这几点特性
> You may call dispatch() from a change listener, with the following caveats:
>
> 1. The listener should only call dispatch() either in response to user actions or under specific conditions (e. g. dispatching an action when the store has a specific field). Calling dispatch() without any conditions is technically possible, however it leads to an infinite loop as every dispatch() call usually triggers the listener again.
>
> 2. The subscriptions are snapshotted just before every dispatch() call. If you subscribe or unsubscribe while the listeners are being invoked, this will not have any effect on the dispatch() that is currently in progress. However, the next dispatch() call, whether nested or not, will use a more recent snapshot of the subscription list.
>
> 3. The listener should not expect to see all state changes, as the state might have been updated multiple times during a nested dispatch() before the listener is called. It is, however, guaranteed that all subscribers registered before the dispatch() started will be called with the latest state by the time it exits.

大致就是说：
1. dispatch 必须是有条件的调用，为了防止无限循环。比如说Reducer就不能再dispatch，实现上就是用一个isDispatch标记是否当前是处于Reducer执行的过程中。
2. Reducer执行的过程中不能进行subscribe和unsubscribe的操作。但是listeners执行的过程中可以subscribe或者unsubscribe，但是不影响这一次的dispatch，dispatch的时候当前有哪些listener是做了一个快照的，比如某个listener执行过程中subscribe了新的listener，这个新的listener是不会在这一次dispatch就被执行的。反之unsubscribe的listener在这一次的dispatch中仍然会被执行。一次
3. 第三点实际上是为了兼容concurrent mode。concurrent mode 下如果其他的re-render插进来的话，就会导值被打断的组件前后拿到不同的state。所以redux某个版本做了改进，

其他：
- 为什么不允许reducer用getState？因为如果用combineReducers，每个reducer只可以看到自己那部分state，如果允许getState，reducer之间的隔离就被破坏。

## Middleware

middleware的函数签名是`({ getState, dispatch }) => next => action`

参照redux-thunk的源码
```
function createThunkMiddleware(extraArgument) {
  return ({ dispatch, getState }) => (next) => (action) => {
    if (typeof action === 'function') {
      return action(dispatch, getState, extraArgument);
    }

    return next(action);
  };
}

const thunk = createThunkMiddleware();
thunk.withExtraArgument = createThunkMiddleware;

export default thunk;
```

```
const chain = middlewares.map(middleware => middleware(middlewareAPI))
dispatch = compose<typeof dispatch>(...chain)(store.dispatch)
```

```
export default function compose(...funcs: Function[]) {
  if (funcs.length === 0) {
    // infer the argument type so it is usable in inference down the line
    return <T>(arg: T) => arg
  }

  if (funcs.length === 1) {
    return funcs[0]
  }

  return funcs.reduce(
    (a, b) =>
      (...args: any) =>
        a(b(...args))
  )
}

```

applyMiddleware的时候先是用`{dispatch, getState}`去调用每个middleware，拿到（next)=>(action)=>any，然后调compose去拿掉一个通过修改后的dispatch函数，compose这一层是从后向前，最后一个middleware拿到的是原始的dispatch，前一个拿到next参数是一个回调函数，就是后一个修改包装过的dispatch函数。

redux-thunk在dispatch的时候就判断如果action是一个函数，就用它自己的api去调用这个函数，如果不是函数就不做处理，让下一个middleware包装的dispatch函数去处理这个action，知道最后一个middleware直接调用原始的dispatch。

总而言之，redux的middleware的作用就是去增强dispatch函数。

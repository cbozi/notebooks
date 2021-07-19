ReactDOM.render()

ReactDOM.render ->
- ReactDOMLegacy.js: render ->
- legacyRenderSubtreeIntoContainer 


```
function legacyRenderSubtreeIntoContainer(
  parentComponent: ?React$Component<any, any>,
  children: ReactNodeList,
  container: DOMContainer,
  forceHydrate: boolean,
  callback: ?Function,
) {
  if (__DEV__) {
    topLevelUpdateWarnings(container);
    warnOnInvalidCallback(callback === undefined ? null : callback, 'render');
  }

  // TODO: Without `any` type, Flow says "Property cannot be accessed on any
  // member of intersection type." Whyyyyyy.
  let root: RootType = (container._reactRootContainer: any);
  let fiberRoot;
  if (!root) {
    // Initial mount
    root = container._reactRootContainer = legacyCreateRootFromDOMContainer(
      container,
      forceHydrate,
    );
    fiberRoot = root._internalRoot;
    if (typeof callback === 'function') {
      const originalCallback = callback;
      callback = function() {
        const instance = getPublicRootInstance(fiberRoot);
        originalCallback.call(instance);
      };
    }
    // Initial mount should not be batched.
    unbatchedUpdates(() => {
      updateContainer(children, fiberRoot, parentComponent, callback);
    });
  } else {
    fiberRoot = root._internalRoot;
    if (typeof callback === 'function') {
      const originalCallback = callback;
      callback = function() {
        const instance = getPublicRootInstance(fiberRoot);
        originalCallback.call(instance);
      };
    }
    // Update
    updateContainer(children, fiberRoot, parentComponent, callback);
  }
  return getPublicRootInstance(fiberRoot);
}
```
根据`root = container._reactRootContainer`判断是否初次渲染，初次渲染首先调用`legacyCreateRootFromDOMContainer`创建一个`ReactRoot`，创建`ReactRoot`时会将`root._internalRoot`初始化为一个`FiberRootNode`，`FiberRootNode.current`是一个`FiberNode`

然后调用`updateContainer`进行渲染。

如果是首次渲染，调用`unbatchedUpdates(src/react/node_modules/react-reconciler/src/ReactFiberWorkLoop.js)`将`executionContext`设为`LegacyUnbatchedContext`，因为这是初次渲染，需要尽快完成。
 
---

`updateContainer`（packages/react-reconciler/src/ReactFiberReconciler.js），定义如下：

1. 首先获取`currentTime`，并计算`expirationTime`
2. 调用`createUpdate`生成一个`update`对象，接着调用`enqueueUpdate`将其添加到`fiber.updateQueue`
3. 调用`scheduleWork`进行任务调度。首次渲染进入`performSyncWorkOnRoot`

--- 

`performSyncWorkOnRoot`进入`workLoopSync`(packages/react-reconciler/src/ReactFiberWorkLoop.js)，循环调用`performUnitOfWork`

```
function workLoopSync() {
  // Already timed out, so perform work without checking if we need to yield.
  while (workInProgress !== null) {
    workInProgress = performUnitOfWork(workInProgress);
  }
}
```


最后进入返回到`performSyncWorkOnRoot`，进入commit阶段。
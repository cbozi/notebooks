Hooks

Hooks

## useImperativeHandle

[Hooks API Reference – React](https://reactjs.org/docs/hooks-reference.html#useimperativehandle)

useImperativeHandle一般和forwardRef一起使用，用来自定义暴漏给父组件的实例值。例如以下实例中，如果父组件这样调用`<FancyInput ref={inputRef} />`，就可以通过`inputRef.current.focus`调用input元素的focus方法。
```
function FancyInput(props, ref) {
  const inputRef = useRef();
  useImperativeHandle(ref, () => ({
    focus: () => {
      inputRef.current.focus();
    }
  }));
  return <input ref={inputRef} ... />;
}
FancyInput = forwardRef(FancyInput);
```

其他的使用案例，例如Ant Design Form中，可以通过ref调用FormInstance上的方法（getFieldValue, resetFields, submit)等，也是用useImperativeHandle实现的。

## useMemo

看到react-redux源码里面，感到奇怪，props不是每次rerender都会变得？实际上如果是这个组件自身触发的re-render而不是从父组件传下来的，那这个props就不会变。
```
      const [
        propsContext,
        reactReduxForwardedRef,
        wrapperProps,
      ] = useMemo(() => {
        // Distinguish between actual "data" props that were passed to the wrapper component,
        // and values needed to control behavior (forwarded refs, alternate context instances).
        // To maintain the wrapperProps object reference, memoize this destructuring.
        const { reactReduxForwardedRef, ...wrapperProps } = props
        return [props.context, reactReduxForwardedRef, wrapperProps]
      }, [props])
```
TypeScript

TypeScript


## `React.ForwardRefRenderFunction`
[Forwarding Refs – React](https://reactjs.org/docs/forwarding-refs.html)
```
    interface ForwardRefRenderFunction<T, P = {}> {
        (props: PropsWithChildren<P>, ref: ((instance: T | null) => void) | MutableRefObject<T | null> | null): ReactElement | null;
        displayName?: string;
        // explicit rejected with `never` required due to
        // https://github.com/microsoft/TypeScript/issues/36826
        /**
         * defaultProps are not supported on render functions
         */
        defaultProps?: never;
        /**
         * propTypes are not supported on render functions
         */
        propTypes?: never;
    }
```

```
 React.ForwardRefRenderFunction<
  HTMLDivElement,
  AppProps
>
```

## 原生DOM标签的类型
```
React.IframeHTMLAttributes<T> 
React.FormHTMLAttributes<T> 

// 用法
React.FormHTMLAttributes<HTMLFormElement>
```
TypeScript基础类型

## `any` vs `unkown`

`any`可以赋值给任何类型，任何类型都可以赋值给`any`类型的变量，`any`类型可自由访问其任何属性。使用`any`意味着关闭任何类型检查。

任何类型的值都可以赋值给`unknown`，`unkown`类型无法赋值给任何类型，除了`unknown`,`unkown`类型上不允许做任何操作，除非类型转换。`unknown`适合使用的场景，这个变量可以是任何类型，所以你在使用它之前必须进行类型检查。

## `void`

> void is a little like the opposite of any: the absence of having any type at all. You may commonly see this as the return type of functions that do not return a value:

```
function warnUser(): void {
    console.log("This is my warning message");
}
```

实际上`undefined`和`null`类型都可以赋值给`void`

## `never`

`never`通常表示那些不可能返回的函数和表达式的返回值类型。

```
// Function returning never must have unreachable end point
function error(message: string): never {
    throw new Error(message);
}

// Inferred return type is never
function fail() {
    return error("Something failed");
}

// Function returning never must have unreachable end point
function infiniteLoop(): never {
    while (true) {
    }
}
```

>  The never type is a subtype of, and assignable to, every type; however, no type is a subtype of, or assignable to, never (except never itself). Even any isn’t assignable to never.

```
let something: void = null;
let nothing: never = null; // Error: Type 'null' is not assignable to type 'never'
```

个人理解，`never`类型表示某段代码永远不会到达，它的作用就是如果你写了一个死循环或者必定抛出错误的代码，可以在编译时发现错误。

另一个用途就是可以用来确保你没有漏掉其中一种类型的处理逻辑
```
function forever(): never {
    while (true) {
        break; // Error because you can't leave the function.
    }
}

enum Values {
    A,
    B
}

function choose(value: Values) {
    switch (value) {
        case Values.A: return "A";
        default: let x: never = null; // Error because B is not a case in switch.
    }
}
```
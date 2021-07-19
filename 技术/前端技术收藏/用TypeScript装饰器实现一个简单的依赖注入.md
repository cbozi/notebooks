用TypeScript装饰器实现一个简单的依赖注入

1.安装 `typescript` 环境以及重要的 polyfill `reflect-metadata`。

2.在 `tsconfig.json` 中配置 `compilerOptions`

```
{    "experimentalDecorators": true, // 开启装饰器
```

同时在入口文件引入 `reflect-metadata`。

3

预备知识

### Reflect

#### 简介

`Proxy` 与 `Reflect` 是 `ES6` 为了操作对象引入的 `API`，`Reflect` 的 `API` 和 `Proxy` 的 `API` 一一对应，并且可以函数式的实现一些对象操作。另外，使用 `reflect-metadata` 可以让 `Reflect` 支持**元编程**。

#### 参考资料

Reflect - JavaScript | MDN  
Metadata Proposal - ECMAScript

#### 类型获取

- 类型元数据：design:type
    

- 参数类型元数据：design:paramtypes
    

- 返回类型元数据：design:returntype
    

**使用方法**

```
/** 
```

4

正式开始

## 编写模块

首先写一个服务提供者，作为被依赖模块:

`// @services/log.ts``class  LogService  {`

 `public debug(...args: any[]): void {  
    console.debug('[DEB]', new Date(), ...args);  
  }

  public info(...args: any[]): void {  
    console.info('[INF]', new Date(), ...args);  
  }

  public error(...args: any[]): void {  
    console.error('[ERR]', new Date(), ...args);`

 `}``}`

然后我们再写一个消费者：

`// @controllers/custom.ts``import  LogService  from  '@services/log';`

`class  CustomerController  {` `private log!: LogService;  
  private token = "29993b9f-de22-44b5-87c3-e209f4174e39";

  constructor() {  
    this.log = new LogService();  
  }

  public main(): void {  
    this.log.info('Its running.');`

 `}}`

现在我们看到，消费者在 `constructor` 构造函数内对 `LogService` 进行了实例化，并在 `main` 函数内进行了调用，这是**传统的调用方式**。

当 `LogService` 变更，修改了构造函数，而这个模块被依赖数很多，我们就得挨个找到引用此模块的部分，并一一修改实例化代码。

## 架构设计

1.  我们需要用一个 `Map` 来存储注册的依赖，并且它的 `key` 必须唯一。所以我们首先设计一个容器。
    
2.  注册依赖的时候尽可能简单，甚至不需要用户自己定义 `key`，所以这里使用 `Symbol` 和唯一字符串来确定一个依赖。
    
3.  我们注册的依赖不一定是类，也可能是一个函数、字符串、单例，所以要考虑不能使用装饰器的情况。
    

### Container

先来设计一个 `Container` 类，它包括 `ContainerMap`、`set`、`get`、`has` 属性或方法。`ContainerMap` 用来存储注册的模块，`set` 和 `get` 用来注册和读取模块，`has` 用来判断模块是否已经注册。

- `set` 形参 `id` 表示模块 `id`， `value` 表示模块。
    

- `get` 用于返回指定模块 `id` 对应的模块。
    

- `has` 用于判断模块是否注册。
    

`// @libs/di/Container.ts``class Container {`

 `private ContainerMap = new Map<string | symbol, any>();

  public set = (id: string | symbol, value: any): void => {  
    this.ContainerMap.set(id, value);  
  }

    public get = <T extends any>(id: string | symbol): T => {  
    return this.ContainerMap.get(id) as T;  
  }

  public has = (id: string | symbol): Boolean => {  
    return this.ContainerMap.has(id);`

 `}}``const  ContainerInstance  =  new  Container();``export  default  ContainerInstance;`

### Service

现在实现 `Service` 装饰器来注册类依赖。

`// @libs/di/Service.ts``import  Container  from  './Container';``interface  ConstructableFunction  extends  Function  {  
`

 `new (): any;``}``// 自定义 id 初始化``export function Service (id: string): Function;``// 作为单例初始化``export  function  Service  (singleton: boolean):  Function;``// 自定义 id 并作为单例初始化``export  function  Service  (id: string, singleton: boolean):  Function;``export  function  Service  (idOrSingleton?: string | boolean, singleton?: boolean):  Function  {  
`

 ``return (target: ConstructableFunction) => {  
    let _id;  
    let _singleton;  
    let _singleInstance;

    if (typeof idOrSingleton === 'boolean') {  
      _singleton = true;  
      _id = Symbol(target.name);  
    } else {  
      // 判断如果设置 id，id 是否唯一  
      if (idOrSingleton && Container.has(idOrSingleton)) {  
        throw new Error(`Service：此标识符（${idOrSingleton}）已被注册.`);  
      }

      _id = idOrSingleton || Symbol(target.name);  
      _singleton = singleton;  
    }

    Reflect.defineMetadata('cus:id', _id, target);

    if (_singleton) {  
      _singleInstance = new target();  
    }

    Container.set(_id, _singleInstance || target);``

 `};``};`

`Service` 作为一个类装饰器，`id` 是可选的一个标记模块的变量，`singleton` 是一个可选的标记是否单例的变量，`target` 表示当前要注册的类，拿到这个类之后，给它添加 `metadata`，方便日后使用。

### Inject

接下来实现 `Inject` 装饰器用来注入依赖。

`// @libs/di/Inject.ts``import Container from  './Container';// 使用 id 定义模块后，需要使用 id 来注入模块`

`export function Inject(id?: string): PropertyDecorator {  
  return (target: Object, propertyKey: string | symbol) => {  
    const Dependency = Reflect.getMetadata("design:type", target, propertyKey);

    const _id = id || Reflect.getMetadata("cus:id", Dependency);  
    const _dependency = Container.get(_id);

    // 给属性注入依赖  
    Reflect.defineProperty(target, propertyKey, {  
      value: _dependency,  
    });

`

 `};``}`

5

开始使用

### 服务提供者

log 模块：

`// @services/log.ts``import  { Service }  from  '@libs/di';``@Service(true)``class  LogService  {`

 `...``}`

config 模块：

`// @services/config.ts``import  { Container }  from  '@libs/di';``export const token =  '29993b9f-de22-44b5-87c3-e209f4174e39';// 可在入口文件处调用以载入`

`export default () => {`

 `Container.set('token', token);``};`

### 消费者

`// @controllers/custom.ts``import LogService from  '@services/log';``class  CustomerController  {`

 `// 使用 Inject 注入  
  @Inject()  
  private log!: LogService;

  // 使用 Container.get 注入  
  private token = Container.get('token');

  public main(): void {  
    this.log.info(this.token);`

 `}``}`

### 运行结果

```
[INF] 2020-08-07T11:56:48.775Z 29993b9f-de22-44b5-87c3-e209f4174e39
```

### 注意事项

`decorator` 有可能会在正式调用之前初始化，因此 `Inject` 步骤可能会在使用 `Container.set` 注册之前执行（如上文的 `config` 模块注册和 `token` 的注入），此时可以使用 `Container.get` 替代。

我们甚至可以让参数注入在 `constructor` 形参里面，使用 `Inject` 直接在构造函数里注入依赖。当然这就需要自己下去思考了：

```
constructor(@Inject() log: LogService) {
```

  

IMWeb团队隶属腾讯公司，是国内最专业的前端团队之一。  

我们专注前端领域多年，负责过QQ资料、QQ注册、QQ群等亿级业务。目前聚焦于在线教育领域，精心打磨腾讯课堂、企鹅辅导及ABCMouse三大产品。

![](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

扫码关注 **腾讯IMWeb前端团队**

![](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)
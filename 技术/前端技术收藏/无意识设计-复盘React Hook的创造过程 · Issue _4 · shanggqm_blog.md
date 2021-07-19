无意识设计-复盘React Hook的创造过程 · Issue #4 · shanggqm/blog

[![](https://camo.githubusercontent.com/6f3b2f32f84035be10788fb59c020f5f931a7a66f857839c990023b8f802650e/68747470733a2f2f62697a696d672e736f676f7563646e2e636f6d2f3230313930372f31362f31312f35322f32342f6b6f616c612f576563686174494d47323039322e6a7067)](https://camo.githubusercontent.com/6f3b2f32f84035be10788fb59c020f5f931a7a66f857839c990023b8f802650e/68747470733a2f2f62697a696d672e736f676f7563646e2e636f6d2f3230313930372f31362f31312f35322f32342f6b6f616c612f576563686174494d47323039322e6a7067)  
题图：杜鹃湖的秋夜 | 摄影师：郭美青

2018年的React Conf上Dan Abramov正式对外介绍了`React Hook`，这是一种让函数组件支持状态和其他React特性的全新方式，并被官方解读为这是下一个5年React与时俱进的开端。从中细品，可以窥见`React Hook`的重要性。今年2月6号，React Hook新特性随React v16.8.0版本正式发布，整个上半年React社区都在积极努力地拥抱它，学习并解读它。虽然官方声明，`React Hook`还在快速的发展和更新迭代过程中，很多`Class Component`支持的特性，`React Hook`还并未支持，但这丝毫不影响社区的学习热情。

`React Hook`上手非常简单，使用起来也很容易，但相比我们已经熟悉了5年的类组件写法，`React Hook`还是有一些理念和思想上的转变。React团队也给出了使用Hook的一些[规则](https://zh-hans.reactjs.org/docs/hooks-rules.html)和[eslint插件](https://www.npmjs.com/package/eslint-plugin-react-hooks)来辅助降低违背规则的概率，但规则并不是仅仅让我们去记忆的，更重要的是要去真正理解设计这些规则的原因和背景。

本文是我个人在学习React Hook的过程中，通过学习官方文档、阅读源码、浏览其他优秀同行撰写的经验文章，再结合自己的思考，通过逆向思维从React Hook希望解决的问题出发，复盘了React Hook的核心架构设计和创造的过程。非常适合希望对React Hook有更深了解，但又不愿意去读晦涩的源码的同学。

文章中的代码很多只是伪代码，重点在解读设计思路，因此并非完整的实现。很多链表的构建和更新逻辑也一并省略了，但并不影响大家了解整个React Hook的设计。事实上`React Hook`的大部分代码都在适配`React Fiber`架构的理念，这也是源码晦涩难懂的主要原因。不过没关系，我们完全可以先屏蔽掉`React Fiber`的存在，去一点点构建纯粹的React Hook架构。

因本人能力的局限性，文中难免有解读不正确之处，盼望大家可以交流指正（笔者github博客地址：[https://github.com/shanggqm/blog）。](https://github.com/shanggqm/blog%EF%BC%89%E3%80%82)

React Hook的产生主要是为了解决什么问题呢？官方的文档里写的非常清楚，这里只做简单的提炼，不做过多陈述，没读过文档的同学可以先移步阅读[`React Hook简介`](https://zh-hans.reactjs.org/docs/hooks-intro.html)。

总结一下要解决的痛点问题就是：

1.  在组件之间复用状态逻辑很难
    - 之前的解决方案是：[render props](https://react.docschina.org/docs/render-props.html) 和高阶组件
    - 缺点是难理解、存在过多的嵌套形成“嵌套地狱”  
        [![](https://camo.githubusercontent.com/3e0bca588e0cc6d9aa437cb3dc3b85e3ed6c848c815bd8c843be8246c2bdaa28/68747470733a2f2f62697a696d672e736f676f7563646e2e636f6d2f3230313930372f31312f31332f34352f34352f6b6f616c612f45383444334136362d303137342d343141432d384238452d3746363245424646363836452e706e67)](https://camo.githubusercontent.com/3e0bca588e0cc6d9aa437cb3dc3b85e3ed6c848c815bd8c843be8246c2bdaa28/68747470733a2f2f62697a696d672e736f676f7563646e2e636f6d2f3230313930372f31312f31332f34352f34352f6b6f616c612f45383444334136362d303137342d343141432d384238452d3746363245424646363836452e706e67)
2.  复杂组件变的难以理解
    - 生命周期函数中充斥着各种状态逻辑和副作用
    - 这些副作用难以复用，且很零散
3.  难以理解的Class
    - this指针问题
    - 组件预编译技术（[组件折叠](https://github.com/facebook/react/issues/7323)）会在class中遇到优化失效的case
    - class不能很好的压缩
    - class在热重载时会出现不稳定的情况

React官网有下面这样一段话：

> 为了解决这些问题，Hook 使你在==非 class 的情况下可以使用更多的 React 特性==。 从概念上讲，React 组件一直更像是函数。而 Hook 则拥抱了函数，同时也没有牺牲 React 的精神原则。Hook 提供了问题的解决方案，无需学习复杂的函数式或响应式编程技术

## 2.1 设计目标和原则

对应第一节所抛出的问题，React Hook的设计目标便是要解决这些问题，总结起来就以下四点：

1.  **无Class的复杂性**
2.  **无生命周期的困扰**
3.  **优雅地复用**
4.  **对齐React Class组件已经具备的能力**

## 2.2 设计方案

### 2.2.1 无Class的复杂性（去Class）

React 16.8发布之前，按照是否拥有状态的维护来划分的话，组件的类型主要有两种：

1.  **类组件Class Component**:主要用于需要内部状态，以及包含副作用的复杂的组件

class App extends React.Component{
    constructor(props){
        super(props);
        this.state = {
            //...
        }
    }
    //...
}

2.  **函数组件Function Component**：主要用于纯组件，不包含状态，相当于一个模板函数

function Footer(links){
    return (
        <footer>
            <ul>
            {links.map(({href, title})=>{
                return <li><a href={href}>{title}</a></li>
            })}
            </ul>
        </footer>
    )
}

如果设计目标是==去Class==的话，似乎选择只能落在改造`Function Component`，让函数组件拥有`Class Component`一样的能力上了。

我们不妨畅想一下最终的支持状态的函数组件代码：

// 计数器
function Counter(){
    let state = {count:0}
    
    function clickHandler(){
        setState({count: state.count+1})   
    }
    
    return (
        <div>
            <span>{count}</span>
            <button onClick={clickHandler}>increment</button>
        </div>
    )
}

上述代码使用函数组件定义了一个计数器组件`Counter`，其中提供了状态`state`，以及改变状态的`setState`函数。这些API对于`Class component`来说无疑是非常熟悉的，但在`Function component`中却面临着不同的挑战：

1.  class实例可以永久存储实例的状态，而函数不能，上述代码中Counter每次执行，state都会被重新赋值为0；
2.  每一个`Class component`的实例都拥有一个成员函数`this.setState`用以改变自身的状态，而`Function component`只是一个函数，并不能拥有`this.setState`这种用法，只能通过全局的setState方法，或者其他方法来实现对应。

以上两个问题便是选择改造`Function component`所需要解决的问题。

#### 2.2.1.1 解决方案

在JS中，可以存储持久化状态的无非几种方法：

1.  类实例属性

class A(){
    constructor(){
        this.count = 0;
    }
    
    increment(){
        return this.count ++;
    }
}
const a = new A();
a.increment();

2.  全局变量

const global = {count:0};

function increment(){
    return global.count++;
}

3.  DOM

const count = 0;
const $counter = $('#counter');
$counter.data('count', count);

funciton increment(){
    const newCount = parseInt($counter.data('count'), 10) + 1;
    $counter.data('count',newCount);
    return newCount;
}

4.  闭包

const Counter = function(){
    let count = 0;
    return {
        increment: ()=>{
            return count ++;
        }
    }
}()

Counter.increment();

5.  其他全局存储：indexDB、LocalStorage等等

`Function component`对状态的诉求只是能存取，因此似乎以上所有方案都是可行的。但作为一个优秀的设计，还需要考虑到以下几点：

1.  使用简单
2.  性能高效
3.  可靠无副作用

方案2和5显然不符合第三点；方案3无论从哪一方面都不会考虑；因此闭包就成为了唯一的选择了。

#### 2.2.1.2 闭包的实现方案

既然是闭包，那么在使用上就得有所变化，假设我们预期提供一个名叫`useState`的函数，该函数可以使用闭包来存取组件的state，还可以提供一个dispatch函数来更新state，并通过初始调用时赋予一个初始值。

function Counter(){
    const \[count, dispatch\] = useState(0)
    
    return (
        <div>
            <span>{count}</span>
            <button onClick={dispatch(count+1)}>increment</button>
        </div>
    )
}

如果用过redux的话，这一幕一定非常眼熟。没错，这不就是一个微缩版的redux单向数据流吗?

> 给定一个初始state，然后通过dispatch一个action，再经由reducer改变state，再返回新的state，触发组件重新渲染。

知晓这些，`useState`的实现就一目了然了：

function useState(initialState){
    let state = initialState;
    function dispatch = (newState, action)=>{
        state = newState;
    }
    return \[state, dispatch\]
}

上面的代码简单明了，但显然仍旧不满足要求。`Function Component`在初始化、或者状态发生变更后都需要重新执行`useState`函数，并且还要保障每一次`useState`被执行时`state`的状态是最新的。

很显然，我们需要一个新的数据结构来保存上一次的`state`和这一次的`state`，以便可以在初始化流程调用`useState`和更新流程调用`useState`可以取到对应的正确值。这个数据结构可以做如下设计，我们假定这个数据结构叫Hook：

type Hook = {
  memoizedState: any,   // 上一次完整更新之后的最终状态值
  queue: UpdateQueue<any, any> | null, //更新队列
};

考虑到第一次组件`mounting`和后续的`updating`逻辑的差异，我们定义两个不同的`useState`函数的实现，分别叫做`mountState`和`updateState`。

function useState(initialState){
    if(isMounting){
        return mountState(initialState);
    }
    
    if(isUpdateing){
        return updateState(initialState);
    }
}

// 第一次调用组件的useState时实际调用的方法
function mountState(initialState){
    let hook = createNewHook();
    hook.memoizedState = initalState;
    return \[hook.memoizedState, dispatchAction\]
}

function dispatchAction(action){
    // 使用数据结构存储所有的更新行为，以便在rerender流程中计算最新的状态值
    storeUpdateActions(action);
    // 执行fiber的渲染
    scheduleWork();
}

// 第一次之后每一次执行useState时实际调用的方法
function updateState(initialState){
    // 根据dispatchAction中存储的更新行为计算出新的状态值，并返回给组件
    doReducerWork();
    
    return \[hook.memoizedState, dispatchAction\];
}   

function createNewHook(){
    return {
        memoizedState: null,
        baseUpdate: null
    }
}

上面的代码基本上反映出我们的设计思路，但还存在两个核心的问题需要解决：

1.  调用`storeUpdateActions`后将以什么方式把这次更新行为共享给`doReducerWork`进行最终状态的计算
2.  同一个state，在不同时间调用`mountState`和`updateState`时，如何实现`hook`对象的共享

##### **更新逻辑的共享**

更新逻辑是一个抽象的描述，我们首先需要根据实际的使用方式考虑清楚一次`更新`需要包含哪些必要的信息。实际上，在一次事件handler函数中，我们完全可以多次调用`dispatchAction`：

function Count(){
    const \[count, setCount\] = useState(0);
    const \[countTime, setCountTime\] = useState(null);
    
    function clickHandler(){
        // 调用多次dispatchAction
        setCount(1);
        setCount(2);
        setCount(3);
        //...
        setCountTime(Date.now())
    }
    
    return (
    <div>
        <div>{count} in {countTime}</div>
        <button onClick={clickHandler} >update counter</button>
    </div>
    )
}

在执行对`setCount`的3次调用中，我们并不希望Count组件会因此被渲染3次，而是会按照调用顺序实现最后调用的状态生效。因此如果考虑上述使用场景的话，我们需要同步执行完`clickHandler`中所有的`dispatchAction`后，并将其更新逻辑顺序存储，然后再触发Fiber的re-render合并渲染。那么多次对同一个`dispatchAction`的调用，我们如何来存储这个逻辑呢？

比较简单的方法就是使用一个队列`Queue`来存储每一次更新逻辑`Update`的基本信息：

type Queue{
    last: Update,   // 最后一次更新逻辑
    dispatch: any,
    lastRenderedState: any  // 最后一次渲染组件时的状态
}

type Update{
    action: any,    // 状态值
    next: Update    // 下一次Update
}

这里使用了单向链表结构来存储更新队列，为什么要用单向链表而不用数组呢？这个问题应该是一道经典的数据结构的面试题，留给大家自己去思考。

有了这个数据结构之后，我们再来改动一下代码：

function mountState(initialState){
    let hook = createNewHook();
    hook.memoizedState = initalState;
    
    // 新建一个队列
    const queue = (hook.queue = {
        last: null,
        dispatch: null,
        lastRenderedState:null
    });
    
    //通过闭包的方式，实现队列在不同函数中的共享。前提是每次用的dispatch函数是同一个
    const dispatch = dispatchAction.bind(null, queue);
    return \[hook.memoizedState, dispatch\]
}

function dispatchAction(queue, action){
    // 使用数据结构存储所有的更新行为，以便在rerender流程中计算最新的状态值
    const update = {
        action,
        next: null
    }
    
    let last = queue.last;
    if(last === null){
        update.next = update;
    }else{
        // ... 更新循环链表
    }
    
    // 执行fiber的渲染
    scheduleWork();
}

function updateState(initialState){
    // 获取当前正在工作中的hook
    const hook = updateWorkInProgressHook();
    
    // 根据dispatchAction中存储的更新行为计算出新的状态值，并返回给组件
    (function doReducerWork(){
        let newState = null;
        do{
            // 循环链表，执行每一次更新
        }while(...)
        hook.memoizedState = newState;
    })();
     
    return \[hook.memoizedState, hook.queue.dispatch\];
} 

到这一步，更新逻辑的共享，我们就已经解决了。

##### **Hook对象的共享**

Hook对象是相对于组件存在的，所以要实现对象在组件内多次渲染时的共享，只需要找到一个和组件全局唯一对应的全局存储，用来存放所有的Hook对象即可。对于一个React组件而言，唯一对应的全局存储自然就是ReactNode，在`React` 16x之后，这个对象应该是`FiberNode`。这里为了简单起见，我们暂时不研究Fiber，我们只需要知道一个组件在内存里有一个唯一表示的对象即可，我们姑且把他叫做`fiberNode`：

type FiberNode {
    memoizedState:any  // 用来存放某个组件内所有的Hook状态
}

现在，摆在我们面前的问题是，我们对`Function component`的期望是什么？我们希望的是用`Function component`的`useState`来完全模拟`Class component`的`this.setState`吗？如果是，那我们的设计原则会是：

> 一个函数组件全局只能调用一次useState，并将所有的状态存放在一个大Object里

如果仅仅如此，那么函数组件已经解决了`去Class`的痛点，但我们并没有考虑`优雅地复用状态逻辑`的诉求。

试想一个状态复用的场景：我们有多个组件需要监听浏览器窗口的`resize`事件，以便可以实时地获取`clientWidth`。在`Class component`里，我们要么在全局管理这个副作用，并借助ContextAPI来向子组件下发更新；要么就得在用到该功能的组件中重复书写这个逻辑。

resizeHandler(){
    this.setState({
        width: window.clientWidth,
        height: window.clientHeight
    });
}

componentDidMount(){
    window.addEventListener('resize', this.resizeHandler)
}

componentWillUnmount(){
    window.removeEventListener('resize', this.resizeHandler);
}

ContextAPI的方法无疑是不推荐的，这会给维护带来很大的麻烦；`ctrl+c ctrl+v`就更是无奈之举了。

如果`Function component`可以为我们带来一种全新的状态逻辑复用的能力，那无疑会为前端开发在复用性和可维护性上带来更大的想象空间。

因此理想的用法是：

const \[firstName, setFirstName\] = useState('James');
const \[secondName, setSecondName\] = useState('Bond');

// 其他非state的Hook，比如提供一种更灵活更优雅的方式来书写副作用
useEffect()

综上所述，设计上理应要考虑一个组件对应多个Hook的用法。带来的挑战是：

> 我们需要在`fiberNode`上存储所有Hook的状态，并确保它们在每一次`re-render`时都可以获取到最新的正确的状态

要实现上述存储目标，直接想到的方案就是用一个hashMap来搞定：

{
    '1': hook1,
    '2': hook2,
    //...
}

如果用这种方法来存储，会需要为每一次hook的调用生成唯一的key标识，这个key标识需要在mount和update时从参数中传入以保证能路由到准确的hook对象。

除此方案之外，还可以使用hook.update采用的单向链表结构来存储，给hook结构增加一个next属性即可实现：

type Hook = {
    memoizedState: any,                     // 上一次完整更新之后的最终状态值
    queue: UpdateQueue<any, any> | null,    // 更新队列
    next: any                               // 下一个hook
}

const fiber = {
    //...
    memoizedState: {
        memoizedState: 'James', 
        queue: {
            last: {
                action: 'Smith'
            },  
            dispatch: dispatch,
            lastRenderedState: 'Smith'
        },
        next: {
            memoizedState: 'Bond',
            queue: {
                // ...
            },
            next: null
        }
    },
    //...
}

这种方案存在一个问题需要注意：

> 整个链表是在`mount`时构造的，所以在`update`时必须要保证执行顺序才可以路由到正确的hook。

我们来粗略对比一下这两种方案的优缺点：

| 方案  | 优点  | 缺点  |
| --- | --- | --- |
| hashMap | 查找定位hook更加方便对hook的使用没有太多规范和条件的限制 | 影响使用体验，需要手动指定key |
| 链表  | API友好简洁，不需要关注key | 需要有规范来约束使用，以确保能正确路由 |

很显然，hashMap的缺点是无法忍受的，使用体验和成本都太高了。而链表方案缺点中的规范是可以通过eslint等工具来保障的。从这点考虑，链表方案无疑是胜出了，事实上这也正是`React`团队的选择。

到这里，我们可以了解到为什么React Hook的规范里要求：

> 只能在函数组件的顶部使用，不能再条件语句和循环里使用

function Counter(){
    const \[count, setCount\] = useState(0);
    if(count >= 1){
        const \[countTime, setCountTime\] = useState(Date.now());
    }
}

// mount 阶段构造的hook链为
{
    memoizedState: {
        memoizedState: '0', 
        queue: {},
        next: null
}

// 调用setCount(1)之后的update 阶段，则会找不到对应的hook对象而出现异常

至此，我们已经基本实现了React Hooks `去Class`的设计目标，现在用函数组件，我们也可以通过`useState`这个hook实现状态管理，并且支持在函数组件中调用多次hook。

### 2.2.2 无生命周期的困扰

上一节我们借助闭包、两个单向链表（单次hook的update链表、组件的hook调用链表）、透传dispatch函数实现了React Hook架构的核心逻辑：**如何在函数组件中使用状态**。到目前为止，我们还没有讨论任何关于生命周期的事情，这一部分也是我们的设计要解决的重点问题。我们经常会需要在组件渲染之前或者之后去做一些事情，譬如：

- 在`Class component`的`componentDidMount`中发送`ajax`请求向服务器端拉取数据；
- 在`Class component`的`componentDidMount`和`componentDidUnmount`中注册和销毁浏览器的事件监听器

这些场景，我们同样需要在React Hook中予以解决。React为`Class component`设计了一大堆生命周期函数：

- 在实际的项目开发中用的比较频繁的，譬如渲染后期的：`componentDidMount`、`componentDidUpdate`、`componentWillUnmount`；
- 很少被使用的渲染前期钩子`componentWillMount`、`componentWillUpdate`；
- 一直以来被滥用且有争议的`componentWillReceiveProps`和最新的`getDerivedStateFromProps`；
- 用于性能优化的`shouldComponentUpdate`；

React 16.3版本已经明确了将在17版本中废弃`componentWillMount`、`componentWillUpdate`和`componentWillReceiveProps`这三个生命周期函数。设计用来取代`componentWillReceiveProps`的`getDerivedStateFromProps`也并不被推荐使用。

真正被重度使用的就是渲染后和用于性能优化的几个，在React hook之前，我们习惯于以render这种技术名词来划分组件的生命周期阶段，根据名字`componentDidMount`我们就可以判断现在组件的DOM已经在浏览器中渲染好了，可以执行副作用了。这显然是技术思维，那么在React Hook里，我们能否抛弃这种思维方式，让开发者无需去关注渲染这件事儿，只需要知道哪些是副作用，哪些是状态，哪些需要缓存即可呢？

根据这个思路我们来设计React Hook的生命周期解决方案，或许应该是场景化的样子：

// 用来替代constructor初始化状态
useState()

// 替代 componentDidMount和componentDidUpdate以及componentWillUnmount
// 统一称为处理副作用
useEffect()

// 替代shouldComponent
useMemo（）

这样设计的好处是开发者不再需要去理清每一个生命周期函数的触发时机，以及在里面处理逻辑会有哪些影响。而是更关注去思考哪些是状态，哪些是副作用，哪些是需要缓存的复杂计算和不必要的渲染。

#### 2.2.2.1 useEffect

`effect`的全称应该是[`Side Effect`](https://en.wikipedia.org/wiki/Side_effect_%28computer_science%29)，中文名叫`副作用`，我们在前端开发中常见的副作用有：

- dom操作
- 浏览器事件绑定和取消绑定
- 发送HTTP请求
- 打印日志
- 访问系统状态
- 执行IO变更操作

在React Hook之前，我们经常会把这些副作用代码写在`componentDidMount`、`componentDidUpdate`和`componentWillUnmount`里，比如：

componentDidMount(){
    this.fetchData(this.props.userId).then(data=>{
        //... setState
    })
    
    window.addEventListener('resize', this.onWindowResize);
    
    this.counterTimer = setInterval(this.doCount, 1000);
}

componentDidUpdate(prevProps){
   if (this.props.userID !== prevProps.userID) {
    this.fetchData(this.props.userID);
  }
}

componentWillUnmount(){
    window.removeEventListener('resize', this.onWindowResize);
    clearInterval(this.counterTimer);
}

这种写法存在一些体验的问题：

1.  同一个副作用的创建和清理逻辑分散在多个不同的地方，这无论是对于新编写代码还是要阅读维护代码来说都不是一个上佳的体验
2.  有些副作用可能要再多个地方写多份

第一个问题，我们可以通过thunk来解决：将清理操作和新建操作放在一个函数中，清理操作作为一个thunk函数被返回，这样我们只要在实现上保障每次effect函数执行之前都会先执行这个thunk函数即可：

useEffect(()=>{
    // do some effect work
    return ()=>{
        // clean the effect
    }
})

第二个问题，对于函数组件而言，则再简单不过了，我们完全可以把部分通用的副作用抽离出来形成一个新的函数，这个函数可以被更多的组件复用。

function useWindowSizeEffect(){
    const \[size, setSize\] = useState({width: null, height: null});
    
    function updateSize(){
        setSize({width: window.innerWidth, height: window.innerHeight});
    }
    
    useEffect(()=>{
        window.addEventListener('resize', updateSize);
        
        return ()=>{
            window.removeEventListener('resize', updateSize);
        }
    })
    
    return size;
}

##### useEffect的执行时机

既然是设计用来解决副作用的问题，那么最合适的时机就是组件已经被渲染到真实的DOM节点之后。因为只有这样，才能保证所有副作用操作中所需要的资源（dom资源、系统资源等）是ready的。

上面的例子中描述了一个在mount和update阶段都需要执行相同副作用操作的场景，这样的场景是普遍的，我们不能假定只有在mount时执行一次副作用操作就能满足所有的业务逻辑诉求。所以在update阶段，useEffect仍然要重新执行才能保证满足要求。

这就是useEffect的真实机制：

> `Function Component`函数（useState、useEffect、...）每一次调用，其内部的所有hook函数都会再次被调用。

这种机制带来了一个显著的问题，就是：

> 父组件的任何更新都会导致子组件内Effect逻辑重新执行，如果effect内部存在性能开销较大的逻辑时，可能会对性能和体验造成显著的影响。

React在`PureComponent`和底层实现上都有过类似的优化，只要依赖的state或者props没有发生变化（浅比较），就不执行渲染，以此来达到性能优化的目的。`useEffect`同样可以借鉴这个思想：

useEffect(effectCreator: Function, deps: Array)

// demo
const \[firstName, setFirstName\] = useState('James');
const \[count, setCount\] = useState(0);

useEffect(()=>{
    document.title = `${firstName}'s Blog`;
}, \[firstName\])

上面的例子中，只要传入的`firstName`在前后两次更新中没有发生变化，`effectCreator`函数就不会执行。也就是说，即便调用多次`setCount(*)`，组件会重复渲染多次，但只要firstName没有发生变化，`effectCreator`函数就不会重复执行。

##### useEffect的实现

useEffect的实现和useState基本相似，在`mount`时创建一个hook对象，新建一个effectQueue，以单向链表的方式存储每一个effect，将effectQueue绑定在fiberNode上，并在完成渲染之后依次执行该队列中存储的effect函数。核心的数据结构设计如下：

type Effect{
    tag: any,           // 用来标识effect的类型，
    create: any,        // 副作用函数
    destroy: any,       // 取消副作用的函数，
    deps: Array,        // 依赖
    next: Effect,       // 循环链表指针
}

type EffectQueue{
    lastEffect: Effect
}

type FiberNode{
    memoizedState:any  // 用来存放某个组件内所有的Hook状态
    updateQueue: any  
}

deps参数的优化逻辑就很简单了：

let componentUpdateQueue = null;
function pushEffect(tag, create, deps){
    // 构建更新队列
    // ...
}

function useEffect(create, deps){
    if(isMount)(
        mountEffect(create, deps)
    )else{
        updateEffect(create, deps)
    }
}

function mountEffect(create, deps){
    const hook = createHook();
    hook.memoizedState = pushEffect(xxxTag, create, deps);
    
}

function updateEffect(create, deps){
    const hook = getHook();
    if(currentHook!==null){
        const prevEffect = currentHook.memoizedState;
        if(deps!==null){
            if(areHookInputsEqual(deps, prevEffect.deps)){
                pushEffect(xxxTag, create, deps);
                return;
            }
        }
    }
    
    hook.memoizedState = pushEffect(xxxTag, create, deps);
}

##### useEffect小结

1.  执行时机相当于`componentDidMount`和`componentDidUpdate`，有return就相当于加了`componentWillUnmount`
2.  主要用来解决代码中的副作用，提供了更优雅的写法
3.  多个effect通过一个单向循环链表来存储，执行顺序是按照书写顺序依次执行
4.  deps参数是通过循环浅比较的方式来判断和上一次依赖值是否完全相同，如果有一个不同，就重新执行一遍Effect，如果相同，就跳过本次Effect的执行。
5.  每一次组件渲染，都会完整地执行一遍清除、创建effect。如果有return一个清除函数的话。
6.  清除函数会在创建函数之前执行

#### 2.2.2.2 useMemo

在`useEffect`中我们使用了一个`deps`参数来声明effect函数对变量的依赖，然后通过`areHookInputsEqual`函数来比对前后两次的组件渲染时`deps`的差异，如果浅比较的结果是相同，那么就跳过effect函数的执行。

仔细想想，这不就是生命周期函数`shouldComponentUpdate`要做的事情吗？何不将该逻辑抽取出来，作为一个通用的hook呢，这就是`useMemo`这个hook的原理。

function mountMemo(nextCreate,deps) {
  const hook = mountWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  const nextValue = nextCreate();
  hook.memoizedState = \[nextValue, nextDeps\];
  return nextValue;
}

function updateMemo(nextCreate,deps){
  const hook = updateWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  // 上一次的缓存结果
  const prevState = hook.memoizedState;
  if (prevState !== null) {
    if (nextDeps !== null) {
      const prevDeps = prevState\[1\];
      if (areHookInputsEqual(nextDeps, prevDeps)) {
        return prevState\[0\];
      }
    }
  }
  const nextValue = nextCreate();
  hook.memoizedState = \[nextValue, nextDeps\];
  return nextValue;
}

但useMemo和`shouldComponentUpdate`的区别在于useMemo只是一个通用的无副作用的缓存Hook，并不会影响组件的渲染与否。所以从这点上讲，useMemo并不能替代`shouldComponentUpdate`，但这丝毫不影响useMemo的价值。useMemo为我们提供了一种通用的性能优化方法，对于一些耗性能的计算，我们可以用useMemo来缓存计算结果，只要依赖的参数没有发生变化，就达到了性能优化的目的。

const result = useMemo(()=>{
    return doSomeExpensiveWork(a,b);
}, \[a,b\])

那么要完整实现`shouldComponentUpdate`的效果应该怎么办呢？答案是借助`React.memo`:

const Button = React.memo((props) => {
  // 你的组件
});

这相当于使用了PureComponent。

到目前为止，除了`getDerivedStateFromProps`，其他常用的生命周期方法在React Hook中都已经有对应的解决方案了，`componentDidCatch`官方已经声明正在实现中。这一节的最后，我们再来看看`getDerivedStateFromProps`的替代方案。

这个生命周期的作用是根据父组件传入的props，按需更新到组件的state中。虽然很少会用到，但在React Hook组件中，仍然可以通过在渲染时调用一次`"setState"`来实现：

function ScrollView({row}) {
  let \[isScrollingDown, setIsScrollingDown\] = useState(false);
  let \[prevRow, setPrevRow\] = useState(null);

  if (row !== prevRow) {
    // Row 自上次渲染以来发生过改变。更新 isScrollingDown。
    setIsScrollingDown(prevRow !== null && row > prevRow);
    setPrevRow(row);
  }

  return `Scrolling down: ${isScrollingDown}`;
}

如果在渲染过程中调用了"setState"，组件会取消本次渲染，直接进入下一次渲染。所以这里一定要注意"setState"一定要放在条件语句中执行，否则会造成死循环。

### 2.2.3 优雅地复用

React组件化开发方式，本质上就是组件的复用，开发一个应用就像搭积木一样把各种组件有机地堆叠在一起。但这是整个组件层面的复用，是一种粗粒度的复用。在不同的组件内部，我们仍然会经常做一些重复劳动，这些重复劳动可能包含以下几种：

- 状态及其逻辑的重复。比如loading状态，计数器等
- 副作用的逻辑重复。比如有同一个ajax请求、多个组件内对同一个浏览器事件的监听、同一类dom操作或者宿主API的调用等。

React Hook的设计目标中很重要的一点就是：

> 如何让状态及其逻辑和副作用逻辑具备真正的复用性而不需要使用`reder-props`和`HOC`？

#### React中的代码复用

使用过早期版本React的同学可能知道[`Mixins`](https://react.docschina.org/docs/react-without-es6.html#mixins) API，这是官方提供的一种比组件更细粒度的逻辑复用能力。在React推出基于ES6的`Class Component`的写法后，就被逐渐'抛弃'了。`Mixins`虽然可以非常方便灵活地解决[`AOP`](https://zh.wikipedia.org/wiki/%E9%9D%A2%E5%90%91%E5%88%87%E9%9D%A2%E7%9A%84%E7%A8%8B%E5%BA%8F%E8%AE%BE%E8%AE%A1)类的问题，譬如组件的性能日志监控逻辑的复用：

const logMixin = {
    componentWillMount: function(){
        console.log('before mount:', Date.now());
    }
    
    componentDidMount: function(){
        console.log('after mount:', Date.now())
    }
}

var createReactClass = require('create-react-class');
const CompA = createReactClass({
    mixins: \[logMixin\],
    render: function(){
        //... 
    }
})

const CompB = createReactClass({
    mixins: \[logMixin\],
    render: function(){
        //... 
    }
})

但这种模式本身会带来很多的危害，具体可以参考官方的一片博文：[《Mixins Considered Harmful》](https://react.docschina.org/blog/2016/07/13/mixins-considered-harmful.html)。

React官方在2016年建议拥抱`HOC`，也就是使用高阶组件的方式来替代`mixins`的写法。`minxins` API仅可以在`create-react-class`手动创建组件时才能使用。这基本上宣告了mixins这种逻辑复用的方式的终结。

`HOC`非常强大，React生态中大量的组件和库都使用了`HOC`，比如`react-redux`的`connect` API：

class MyComp extends Component{
    //...
}
export default connect(MyComp, //...)

用`HOC`实现上面的性能日志打印，代码如下：

function WithOptimizeLog(Comp){
    return class extends Component{
        constructor(props){
            super(props);
           
        }
        
        componentWillMount(){
            console.log('before mount:', Date.now());
        }
        
        componentDidMount(){
            console.log('after mount:', Date.now());
        }
        
        render(){
            return (
                <div>
                    <Comp {...props} />
                </div>
            )
        }
    }
} 

// CompA
export default WithOptimizeLog(CompA)

//CompB
export defaultWithOptimizeLog(CompB);

`HOC`虽然强大，但因其本身就是一个组件，仅仅是通过封装了目标组件提供一些上层能力，因此难以避免的会带来`嵌套地狱`的问题。并且因为`HOC`是一种将可复用逻辑封装在一个React组件内部的高阶思维模式，所以和普通的`React`组件相比，它就像是一个魔法盒子一样，势必会更难以阅读和理解。

可以肯定的是`HOC`模式是一种被广泛认可的逻辑复用模式，并且在未来很长的一段时间内，这种模式仍将被广泛使用。但随着`React Hook`架构的推出，`HOC`模式是否仍然适合用在`Function Component`中？还是要寻找一种新的组件复用模式来替代`HOC`呢？

React官方团队给出的答案是后者，原因是在`React Hook`的设计方案中，借助函数式状态管理以及其他Hook能力，逻辑复用的粒度可以实现的更细、更轻量、更自然和直观。毕竟在Hook的世界里一切都是函数，而非组件。

来看一个例子：

export default function Article() {
    const \[isLoading, setIsLoading\] = useState(false);
    const \[content, setContent\] = useState('origin content');
    
    function handleClick() {
        setIsLoading(true);
        loadPaper().then(content=>{
            setIsLoading(false);
            setContent(content);
        })
    }

    return (
        <div>
            <button onClick={handleClick} disabled={isLoading} >
                {isLoading ? 'loading...' : 'refresh'}
            </button>
            <article>{content}</article>
        </div>
    )
}

上面的代码中展示了一个带有loading状态，可以避免在加载结束之前反复点击的按钮。这种组件可以有效地给予用户反馈，并且避免用户由于得不到有效反馈带来的不断尝试造成的性能和逻辑问题。

很显然，loadingButton的逻辑是非常通用且与业务逻辑无关的，因此完全可以将其抽离出来成为一个独立的`LoadingButton`组件：

function LoadingButton(props){
    const \[isLoading, setIsLoading\] = useState(false);
    
    function handleClick(){
        props.onClick().finally(()=>{
            setIsLoading(false);
        });    
    }
    
    return (
        <button onClick={handleClick} disabled={isLoading} >
            {isLoading ? 'loading...' : 'refresh'}
        </button>
    )
}

// 使用
function Article(){
    const {content, setContent} = useState('');
    
    clickHandler(){
       return fetchArticle().then(data=>{
           setContent(data);
       })
    }
    
    return (
        <div>
            <LoadingButton onClick={this.clickHandler} />
            <article>{content}</article>
        </div>
    )
}

上面这种将某一个通用的UI组件单独封装并提取到一个独立的组件中的做法在实际业务开发中非常普遍，这种抽象方式同时将状态逻辑和UI组件打包成一个可复用的整体。

很显然，这仍旧是组件复用思维，并不是逻辑复用思维。试想一下另一种场景，在点击了loadingButton之后，希望文章的正文也同样展示一个loading状态该怎么处理呢？

如果不对loadingButton进行抽象的话，自然可以非常方便地复用isLoading状态，代码会是这样：

export default function Article() {
    const \[isLoading, setIsLoading\] = useState(false);
    const \[content, setContent\] = useState('origin content');
    
    function handleClick() {
        setIsLoading(true);
        loadArticle().then(content=>{
            setIsLoading(false);
            setContent(content);
        })
    }

    return (
        <div>
            <button onClick={handleClick} disabled={isLoading} >
                {isLoading ? 'loading...' : 'refresh'}
            </button>
            {
                isLoading
                    ? <img src={spinner}  alt="loading" />
                    : <article>{content}</article>
            }
        </div>
    )
}

但针对抽象出LoadingButton的版本会是什么样的状况呢？

function LoadingButton(props){
    const \[isLoading, setIsLoading\] = useState(false);
    
    function handleClick(){
        props.onClick().finally(()=>{
            setIsLoading(false);
        });    
    }
    
    return (
        <button onClick={handleClick} disabled={isLoading} >
            {isLoading ? 'loading...' : 'refresh'}
        </button>
    )
}

// 使用
function Article(){
    const {content, setContent} = useState('origin content');
    const {isLoading, setIsLoading} = useState(false);
    
    clickHandler(){
       setIsLoading(true);
       return fetchArticle().then(data=>{
           setContent(data);
           setIsLoading(false);
       })
    }
    
    return (
        <div>
            <LoadingButton onClick={this.clickHandler} />
            {
                isLoading
                    ? <img src={spinner}  alt="loading" />
                    : <article>{content}</article>
            }
        </div>
    )
}

问题并没有因为抽象而变的更简单，父组件Article仍然要自定一个isLoading状态才可以实现上述需求，这显然不够优雅。那么问题的关键是什么呢？

答案是`耦合`。上述的抽象方案将`isLoading`状态和`button`标签耦合在一个组件里了，这种复用的粒度只能整体复用这个组件，而不能单独复用一个状态。解决方案是：

// 提供loading状态的抽象
export function useIsLoading(initialValue, callback) {
    const \[isLoading, setIsLoading\] = useState(initialValue);

    function onLoadingChange() {
        setIsLoading(true);

        callback && callback().finally(() => {
            setIsLoading(false);
        })
    }

    return {
        value: isLoading,
        disabled: isLoading,
        onChange: onLoadingChange, // 适配其他组件
        onClick: onLoadingChange,  // 适配按钮
    }
}

export default function Article() {
    const loading = useIsLoading(false, fetch);
    const \[content, setContent\] = useState('origin content');

    function fetch() {
       return loadArticle().then(setContent);
    }

    return (
        <div>
            <button {...loading}>
                {loading.value ? 'loading...' : 'refresh'}
            </button>
           
            {
                loading.value ? 
                    <img src={spinner} alt="loading" />
                    : <article>{content}</article>
            }
        </div>
    )
}

如此便实现了更细粒度的状态逻辑复用，在此基础上，还可以根据实际情况，决定是否要进一步封装UI组件。譬如，仍然可以封装一个LoadingButton：

// 封装按钮
function LoadingButton(props){
    const {value, defaultText = '确定', loadingText='加载中...'} = props;
    return (
        <button {...props}>
            {value ? loadingText: defaultText}
        </button>
    )
}

// 封装loading动画
function LoadingSpinner(props) {
    return (
        < >
            { props.value && <img src={spinner} className="spinner" alt="loading" /> }
        </>
    )
}
// 使用

return (
    <div>
        <LoadingButton {...loading} />
        <LoadingSpinner {...loading}/>
        { loading.value || <article>{content}</article> }
    </div>
)

状态逻辑层面的复用为组件复用带来了一种全新的能力，这完全有赖于`React Hook`基于`Function`的组件设计，一切皆为函数调用。并且，`Function Component`也并不排斥`HOC`，你仍然可以使用熟悉的方法来提供更高阶的能力，只是现在，你的手中拥有了另外一种武器。

#### 自定义Hook的实现原理

上述例子中的useIsLoading函数被称之为`自定义Hook`，它所做的仅仅是将部分`hook`代码提取到一个独立的函数中，就像我们把可复用的逻辑提取到一个独立的函数中一样。

从上文中我们了解到，Hook队列需要存储在组件对应的FiberNode上才可以，那么自定义hook也会对应一个FiberNode吗？自定义Hook对入参和结果有什么要求呢？

我们对自定义Hook的定义是逻辑的复用，而不是组件的复用，因此它不应该像`Function Component`一样直接返回组件树，自然也就没有一个独立的FiberNode来对应了。如果没有独立存储，那自定义hook函数内部调用的useState、useEffect等hook函数的数据结构应该如何存储呢？

答案是绑定在调用这个自定义hook的`Function Component`对应的`FiberNode`上，被抽离出来的自定义Hook逻辑，在实际执行的过程中，就好像copy了一份自定义Hook代码，替换了原来的调用代码，这就是自定义Hook的本质。

因此自定义Hook在使用时也需要遵循Hook规范，需要在函数顶部调用hook，不能写在条件语句和循环里。除此之外，由于规范允许在自定义Hook中调用hook函数，但不允许在普通的function中调用，因此需要一种规范或者机制来保障开发者不会犯错。

React团队给出的方案是命名规范和eslint校验：

1.  自定义Hook必须以`use`开头，以便可以通过命名规范来区分。比如：'useIsLoading'
2.  使用ESLINT插件来确保当开发者犯错时可以进行提示

### 2.2.4 对齐React Class组件已经具备的能力

在本文撰写的时间点上，仍然有一些`Class Component`具备的功能是`React Hook`没有具备的，譬如：生命周期函数`componentDidCatch`，`getSnapshotBeforeUpdate`。还有一些第三方库可能还无法兼容hook，官方给出的说法是：

> `我们会尽快补齐`

未来可期，我们只需静静地等待。

武侠小说中有”无招胜有招“的境界，在设计领域也有”没有设计就是最好的设计“的论断。`React Hook`抛弃`Class`，拥抱函数式编程，使用JS语言独特的闭包来存储状态，这种设计就像是日本设计师深泽直人倡导的无意识设计一样，对于Javascript程序员而言，使用的时候不需要多余的思考，一切皆函数，一切都那么自然、优雅和顺理成章。

- [React Hook 官方文档](https://react.docschina.org/docs/hooks-intro.html)
- [Under the hood of React’s hooks system](https://medium.com/the-guild/under-the-hood-of-reacts-hooks-system-eb59638c9dba)
- [\[译\] 深入 React Hook 系统的原理](https://juejin.im/post/5c99a75af265da60ef635898)
- [react底层原理解析之fiber](https://blog.csdn.net/weixin_43606158/article/details/89425297)
- [react fiber源码分析 原理解析](https://blog.csdn.net/xuyunfei_2012/article/details/81945476)
- [你应该知道的requestIdleCallback](https://juejin.im/post/5ad71f39f265da239f07e862)
- [React Hook全面理解教程](http://caibaojian.com/react-hooks.html)
- [React Fiber架构](https://zhuanlan.zhihu.com/p/37095662)
- [从源码剖析useState的执行过程](https://juejin.im/post/5cc809d2f265da036c579620)
- [数组和链表的区别](https://www.jianshu.com/p/5233f2a2d523)
- [\[译\] 你可能不需要 Derived State](https://zhuanlan.zhihu.com/p/38090110)
- [You Probably Don't Need Derived State](https://react.docschina.org/blog/2018/06/07/you-probably-dont-need-derived-state.html)
- [【React深入】从Mixin到HOC再到Hook](https://juejin.im/post/5cad39b3f265da03502b1c0a#heading-5)
React背后的工具化体系

http://www.ayqy.net/blog/react%e8%83%8c%e5%90%8e%e7%9a%84%e5%b7%a5%e5%85%b7%e5%8c%96%e4%bd%93%e7%b3%bb/

一.概览

React工具链标签云：

Rollup    Prettier    Closure Compiler
Yarn workspace    [x]Haste    [x]Gulp/Grunt+Browserify
ES Module    [x]CommonJS Module
Flow    Jest    ES Lint    React DevTools
Error Code System    HUBOT(GitHub Bot)    npm


P.S.带[x]的表示之前在用，最近（React 16）不用了

简单分类如下：

开发：ES Module, Flow, ES Lint, Prettier, Yarn workspace, HUBOT
构建：Rollup, Closure Compiler, Error Code System, React DevTools
测试：Jest, Prettier
发布：npm


按照ES模块机制组织源码，辅以类型检查和Lint/格式化工具，借助Yarn处理模块依赖，HUBOT检查PR；Rollup + Closure Compiler构建，利用Error Code机制实现生产环境错误追踪，DevTools侧面辅助bundle检查；Jest驱动单测，还通过格式化bundle来确认构建结果足够干净；最后通过npm发布新package

整个过程并不十分复杂，但在一些细节上的考虑相当深入，例如Error Code System、双保险envification（dev/prod环境区分）、发布流程工具化

二.开发工具

CommonJS Module + Haste -> ES Module

React 15之前的版本都用CommonJS模块定义，例如：

var ReactChildren = require('ReactChildren');
module.exports = React;


目前切换到了ES Module，几个原因：

- 有助于及早发现模块引入/导出问题

CommonJS Module很容易require一个不存在的方法，直到调用报错时才能发现问题。ES Module静态的模块机制要求import与export必须按名匹配，否则编译构建就会报错

- bundle size上的优势

ES Module可以通过tree shaking让bundle更干净，根本原因是module.exports是对象级导出，而export支持更细粒度的原子级导出。另一方面，按名引入使得rollup之类的工具能够把模块扁平地拼接起来，压缩工具就能在此基础上进行更暴力的变量名混淆，进一步减小bundle size

只把源码切换到了ES Module，单测用例并未切换，因为CommonJS Module对Jest的一些特性（比如resetModules）更友好（即便切换到ES Module，在需要模块状态隔离的场景，仍然要用require，所以切换意义不大）

至于Haste，则是React团队自定义的模块处理工具，用来解决长相对路径的问题，例如：

// ref: react-15.5.4
var ReactCurrentOwner = require('ReactCurrentOwner');
var warning = require('warning');
var canDefineProperty = require('canDefineProperty');
var hasOwnProperty = Object.prototype.hasOwnProperty;
var REACT_ELEMENT_TYPE = require('ReactElementSymbol');


Haste模块机制下模块引用不需要给出明确的相对路径，而是通过项目级唯一的模块名来自动查找，例如：

// 声明
/**
 * @providesModule ReactClass
 */

// 引用
var ReactClass = require('ReactClass');


从表面上解决了长路径引用的问题（并没有解决项目结构深层嵌套的根本问题），使用非标准模块机制有几个典型的坏处：

- 与标准不和，接入标准生态中的工具时会面临适配问题

- 源码难读，不容易弄明白模块依赖关系

React 16去掉了大部分自定义的模块机制（ReactNative里还有一小部分），采用Node标准的相对路径引用，长路径的问题通过重构项目结构来彻底解决，采用扁平化目录结构（同package下最深2级引用，跨package的经Yarn处理以顶层绝对路径引用）

Flow + ES Lint

Flow负责检查类型错误，尽早发现类型不匹配的潜在问题，例如：

export type ReactElement = {
  $$typeof: any,
  type: any,
  key: any,
  ref: any,
  props: any,
  _owner: any, // ReactInstance or ReactFiber

  // __DEV__
  _store: {
    validated: boolean,
  },
  _self: React$Element<any>,
  _shadowChildren: any,
  _source: Source,
};


除了静态类型声明及检查外，Flow最大的特点是对React组件及JSX的深度支持：

type Props = {
  foo: number,
};
type State = {
  bar: number,
};
class MyComponent extends React.Component<Props, State> {
  state = {
    bar: 42,
  };

  render() {
    return this.props.foo + this.state.bar;
  }
}


P.S.关于Flow的React支持的更多信息，请查看Even Better Support for React in Flow

另外还有导出类型检查的Flow“魔法”，用来校验mock模块的导出类型是否与源模块一致：

type Check<_X, Y: _X, X: Y = _X> = null;
(null: Check<FeatureFlagsShimType, FeatureFlagsType>);


ES Lint负责检查语法错误及约定编码风格错误，例如：

rules: {
  'no-unused-expressions': ERROR,
  'no-unused-vars': [ERROR, {args: 'none'}],
  // React & JSX
  // Our transforms set this automatically
  'react/jsx-boolean-value': [ERROR, 'always'],
  'react/jsx-no-undef': ERROR,
}


Prettier

Prettier用来自动格式化代码，几种用途：

- 旧代码格式化成统一风格

- 提交之前对有改动的部分进行格式化

- 配合持续集成，保证PR代码风格完全一致（否则build失败，并输出风格存在差异的部分）

- 集成到IDE，日常没事格式化一发

- 对构建结果进行格式化，一方面提升dev bundle可读性，另外还有助于发现prod bundle中的冗余代码

统一的代码风格当然有利于协作，另外，对于开源项目，经常面临风格各异的PR，把严格的格式化检查作为持续集成的一个强制环节能够彻底解决代码风格差异的问题，有助于简化开源工作

P.S.整个项目强制统一格式化似乎有些极端，是个大胆的尝试，但据说效果还不错：

Our experience with Prettier has been fantastic, and we recommend it to any team that writes JavaScript.

Yarn workspace

Yarn的workspace特性用来解决monorepo的package依赖（作用类似于lerna bootstrap），通过在node_modules下建立软链接“骗过”Node模块机制

Yarn Workspaces is a feature that allows users to install dependencies from multiple package.json files in subfolders of a single root package.json file, all in one go.

通过package.json/workspaces配置Yarn workspaces：

// ref: react-16.2.0/package.json
"workspaces": [
  "packages/*"
],


注意：Yarn的实际处理与Lerna类似，都通过软链接来实现，只是在包管理器这一层提供monorepo package支持更合理一些，具体原因见Workspaces in Yarn | Yarn Blog

然后yarn install之后就可以愉快地跨package引用了：

import {enableUserTimingAPI} from 'shared/ReactFeatureFlags';
import getComponentName from 'shared/getComponentName';
import invariant from 'fbjs/lib/invariant';
import warning from 'fbjs/lib/warning';


P.S.另外，Yarn与Lerna可以无缝结合，通过useWorkspaces选项把依赖处理部分交由Yarn来做，详细见Integrating with Lerna

HUBOT

HUBOT是指GitHub机器人，通常用于：

- 接持续集成，PR触发构建/检查

- 管理Issue，关掉不活跃的讨论贴

主要围绕PR与Issue做一些自动化的事情，比如React团队计划（目前还没这么做）机器人回复PR对bundle size的影响，以此督促持续优化bundle size

目前每次构建把bundle size变化输出到文件，并交由Git追踪变化（提交上去），例如：

// ref: react-16.2.0/scripts/rollup/results.json
{
  "bundleSizes": {
    "react.development.js (UMD_DEV)": {
      "size": 54742,
      "gzip": 14879
    },
    "react.production.min.js (UMD_PROD)": {
      "size": 6617,
      "gzip": 2819
    }
  }
}


缺点可想而知，这个json文件经常冲突，要么需要浪费精力merge冲突，要么就懒得提交这个自动生成的麻烦文件，导致版本滞后，所以计划通过GitHub Bot把这个麻烦抽离出去

三.构建工具

bundle形式

之前提供两种bundle形式：

- UMD单文件，用作外部依赖

- CJS散文件，用于支持自行构建bundle（把React作为源码依赖）

存在一些问题：

- 自行构建的版本不一致：不同的build环境/配置构建出的bundle都不一样

- bundle性能有优化空间：用打包App的方式构建类库不太合适，性能上有提升余地

- 不利于实验性优化尝试：无法对散文件模块应用打包、压缩等优化手段

React 16调整了bundle形式：

- 不再提供CJS散文件，从npm拿到的就是构建好的，统一优化过的bundle

- 提供UMD单文件与CJS单文件，分别用于Web环境与Node环境（SSR）

以不可再分的类库姿态，把优化环节都收进来，摆脱bundle形式带来的限制

Gulp/Grunt+Browserify -> Rollup

之前的构建系统是基于Gulp/Grunt+Browserify手搓的一套工具，后来在扩展方面受限于工具，例如：

- Node环境下性能不好：频繁的process.env.NODE_ENV访问拖慢了SSR性能，但又没办法从类库角度解决，因为Uglify依靠这个去除无用代码

所以React SSR性能最佳实践一般都有一条“重新打包React，在构建时去掉process.env.NODE_ENV”（当然，React 16不需要再这样做了，原因见上面提到的bundle形式变化）

丢弃了过于复杂（overly-complicated）的自定义构建工具，改用更合适的Rollup：

It solves one problem well: how to combine multiple modules into a flat file with minimal junk code in between.

P.S.无论Haste -> ES Module还是Gulp/Grunt+Browserify -> Rollup的切换都是从非标准的定制化方案切换到标准的开放的方案，应该在“手搓”方面吸取教训，为什么业界规范的东西在我们的场景不适用，非要自己造吗？

mock module

构建时可能面临动态依赖的场景：不同的bundle依赖功能相似但实现存在差异的module，例如ReactNative的错误提醒机制是显示个红框，而Web环境就是输出到Console

一般解法有2种：

- 运行时动态依赖（注入）：把两份都放进bundle，运行时根据配置或环境选择

- 构建时处理依赖：多构建几份，不同的bundle含有各自需要的依赖模块

显然构建时处理更干净一些，即mock module，开发中不用关心这种差异，构建时根据环境自动选择具体依赖，通过手写简单的Rollup插件来实现：动态依赖配置 + 构建时依赖替换

Closure Compiler

google/closure-compiler是个非常强大的minifier，有3种优化模式（compilation_level）：

- WHITESPACE_ONLY：去除注释，多余的标点符号和空白字符，逻辑功能上与源码完全等价

- SIMPLE_OPTIMIZATIONS：默认模式，在WHITESPACE_ONLY的基础上进一步缩短变量名（局部变量和函数形参），逻辑功能基本等价，特殊情况（如eval('localVar')按名访问局部变量和解析fn.toString()）除外

- ADVANCED_OPTIMIZATIONS：在SIMPLE_OPTIMIZATIONS的基础上进行更强力的重命名（全局变量名，函数名和属性），去除无用代码（走不到的，用不着的），内联方法调用和常量（划算的话，把函数调用换成函数体内容，常量换成其值）

P.S.关于compilation_level的详细信息见Closure Compiler Compilation Levels

ADVANCED模式过于强大：

// 输入
function hello(name) {
  alert('Hello, ' + name);
}
hello('New user');

// 输出
alert("Hello, New user");


P.S.可以在Closure Compiler Service在线试玩

迁移切换有一定风险，因此React用的还是SIMPLE模式，但后续可能有计划开启ADVANCED模式，充分利用Closure Compiler优化bundle size

Error Code System

In order to make debugging in production easier, we’re introducing an Error Code System in 15.2.0. We developed a gulp script that collects all of our invariant error messages and folds them to a JSON file, and at build-time Babel uses the JSON to rewrite our invariant calls in production to reference the corresponding error IDs.

简言之，在prod bundle中把详细的报错信息替换成对应错误码，生产环境捕获到运行时错误就把错误码与上下文信息抛出来，再丢给错误码转换服务还原出完整错误信息。这样既保证了prod bundle尽量干净，还保留了与开发环境一样的详细报错能力

例如生产环境下的非法React Element报错：

Minified React error #109; visit https://reactjs.org/docs/error-decoder.html?invariant=109&args[]=Foo for the full message or use the non-minified dev environment for full errors and additional helpful warnings.

很有意思的技巧，确实在提升开发体验上花了不少心思

envification

所谓envification就是分环境build，例如：

// ref: react-16.2.0/build/packages/react/index.js
if (process.env.NODE_ENV === 'production') {
  module.exports = require('./cjs/react.production.min.js');
} else {
  module.exports = require('./cjs/react.development.js');
}


常用手段，构建时把process.env.NODE_ENV替换成目标环境对应的字符串常量，在后续构建过程中（打包工具/压缩工具）会把多余代码剔除掉

除了package入口文件外，还在里面做了同样的判断作为双保险：

// ref: react-16.2.0/build/packages/react/cjs/react.development.js
if (process.env.NODE_ENV !== "production") {
  (function() {
    module.exports = react;
  })();
}


此外，还担心开发者误用dev bundle上线，所以在React DevTools也加了一点提醒：

This page is using the development build of React. 🚧

DCE check

DCE(dead code eliminated) check是指检查无用代码是否被正常去除

考虑了一种特殊情况：process.env.NODE_ENV如果是在运行时设置的话也不合理（可能存在另一环境的多余代码），所以还通过React DevTools做了bundle环境检查：

// ref: react-16.2.0/packages/react-dom/npm/index.js
function checkDCE() {
  if (process.env.NODE_ENV !== 'production') {
    throw new Error('^_^');
  }
  try {
    __REACT_DEVTOOLS_GLOBAL_HOOK__.checkDCE(checkDCE);
  } catch (err) {
    console.error(err);
  }
}
if (process.env.NODE_ENV === 'production') {
  checkDCE();
}

// DevTools 即__REACT_DEVTOOLS_GLOBAL_HOOK__.checkDCE声明
checkDCE: function(fn) {
  try {
    var toString = Function.prototype.toString;
    var code = toString.call(fn);
    if (code.indexOf('^_^') > -1) {
      hasDetectedBadDCE = true;
      setTimeout(function() {
        throw new Error(
          'React is running in production mode, but dead code ' +
            'elimination has not been applied. Read how to correctly ' +
            'configure React for production: ' +
            'https://fb.me/react-perf-use-the-production-build'
        );
      });
    }
  } catch (err) { }
}


原理类似于Redux的minified检测，先声明一个含有dev环境判断的方法，在判断中包含一个标识字符串（上例中是^_^），然后运行时（通过DevTools）检查fn.toString()源码，如果含有该标识字符串就说明DCE失败（无用代码没在build过程中去除），异步throw出来

P.S.关于DCE check的详细信息，请查看Detecting Misconfigured Dead Code Elimination

四.测试工具

Jest

Jest是Facebook推出的测试工具，亮点如下：

- Snapshot Testing：通过DOM树快照来对React/React Native组件做UI测试，把组件渲染结果与之前的快照做对比，没有差异就算通过

- 零配置：不像Mocha强大灵活但配置繁琐，Jest开箱即用，自带测试驱动、断言库、mock机制、测试覆盖率等

Snapshot Testing与UI自动化测试的一般做法类似，对正确结果截屏作为基准（这个基准需要持续更新，所以快照文件一般随源码提交上去），后续每次改动后与之前的截图做像素级对比，存在差异则说明有问题

另外，提到React App测试，还有一个更狠的：Enzyme，可以采用Jest + Enzyme对React组件进行深度测试，更多信息请查看Unit Testing React Components: Jest or Enzyme?

P.S.关于前端UI自动化测试的一般方法，见如何进行前端自动化测试？ – 张云龙的回答 – 知乎

P.S.可以在repl.it – try-jest by @amasad在线试玩

preventing Infinite Loops

即死循环检查，不希望测试过程被死循环阻塞（React 16递归改循环之后有很多while (true)，他们不太放心）。处理方式与死递归检查类似：限制最大深度（TTL）。通过Babel插件来做，在测试环境构建时注入检查：

// ref: https://github.com/facebook/react/blob/master/scripts/jest/preprocessor.js#L38
require.resolve('../babel/transform-prevent-infinite-loops'),

// ref: https://github.com/facebook/react/blob/master/scripts/babel/transform-prevent-infinite-loops.js#L37
'WhileStatement|ForStatement|DoWhileStatement': (path, file) => {
  const guard = buildGuard({
    ITERATOR: iterator,
    MAX_ITERATIONS: t.numericLiteral(MAX_ITERATIONS),
  });
  if (!path.get('body').isBlockStatement()) {
    const statement = path.get('body').node;
    path.get('body').replaceWith(t.blockStatement([guard, statement]));
  } else {
    path.get('body').unshiftContainer('body', guard);
  }
}


用来防护的buildGuard如下：

const buildGuard = template(`
  if (ITERATOR++ > MAX_ITERATIONS) {
    global.infiniteLoopError = new RangeError(
      'Potential infinite loop: exceeded ' +
      MAX_ITERATIONS +
      ' iterations.'
    );
    throw global.infiniteLoopError;
  }
`);


注意这里使用了一个全局错误变量global.infiniteLoopError，用来中断后续测试流程：

// ref: https://github.com/facebook/react/blob/master/scripts/jest/setupTests.js#L56
 env.afterEach(() => {
  const error = global.infiniteLoopError;
  global.infiniteLoopError = null;
  if (error) {
    throw error;
  }
});


在每个case结束都看一眼是否发生死循环，防止guard中throw的错误被外层catch住后，测试流程仍然正常进行

manual test fixture

除了Node环境工程化的单测外，还创建了浏览器环境人工测试的用例集，包括：

- 基于WebDriver的应用测试（在Facebook，这个应用就指主站）

- 人工测试用例，需要的时候人工验证DOM相关的改动

不做浏览器环境的自动化测试主要有3个原因：

- 浏览器环境的测试工具不那么可靠（flaky），依以往经验来看，并不能如愿发现很多问题

- 会拖慢持续集成，影响开发工作流效率，而且会让持续集成也变得相对脆弱

- 自动化测试并不总能发现DOM问题，例如浏览器显示的输入值可能与通过DOM属性取到的不一致

不愿意做浏览器环境的自动化测试，又想确保维护中添加的一些边界case处理不被更新改动破坏，所以决定采用最有效的方式：针对边界case写测试用例，人工测试验证

具体做法是对着Demo App手动切换React版本，看不同版本/不同浏览器下表现是否一致：

The fixture app lets you choose a version of React (local or one of the published versions) which is handy for comparing the behavior before and after the changes.

看起来很蠢，但对于发现DOM相关问题确实是最直接有效的方式，而且这些用例积累到一定程度时，对于保证质量会起到相当大的作用（自信的进行DOM相关改动，避免到后面没人敢动的境地），例如：

the DOM attribute handling in React 16 was very hard to pull off with confidence at first. We kept discovering different edge cases, and almost gave up on doing it in time for the React 16 release.

积累有价值的人工测试用例要投入很多精力，除了通过工程化手段尽可能自动化外，还计划通过GitHub Bot让社区伙伴也能轻松参与进来，所以这样的“蠢事”也不是不可为，而可预见的好处是：大改不虚

五.发布工具

npm publish

为了规范/简化发布流程，做了几件事情：

- 采用master + feature flag的分支策略

- 工具化发布流程

之前采用stable分支策略，发版时手动cherry-pick，发个版就要花一整天。后来调整为直接从master发布，对于不想要的breaking change，通过feature flag在构建时去掉，免去了手动cherry-pick的繁琐

对发布流程做了全套工具，能自动的就自动顺序执行，依赖人工操作的就提示出来保存退出，人工处理完毕后恢复进度接着往下走，例如：

自动
  test
  build
人工
  changelog
  smoke test
自动
  commit changelog
  publish npm package
人工
  GitHub release
  update site version
  test new release
  notify involved team


这样通过工具化checklist减少人为失误，保证规范一致的发布流程

P.S.为了便于检查发布工具自身，还提供了模拟发布选项，可以跳过发版的实际操作，空跑流程


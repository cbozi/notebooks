react源码学习环境搭建

阅读源码时，有许多变量在程序运行过程中不断的产生，其中存放着什么东西，一直是一个比较头疼的问题。不停的推导增加了验算的负担，随着代码逐渐的深入，也会产生一定的记忆负担。如果靠脑袋去记，简单点的代码还好。复杂的代码。。。你懂的。  
随着react被广泛使用，很多人会好奇react是怎么实现的。会有一探源码的想法。如果直接阅读react.development.js是很简单，页面引入就好了。但是react.development.js终于是经过编译工具编译过的代码，很多的代码看起来并不直观。理想的情况是直接引用源文件，也就是github上react仓库中，packages目录下的代码，直接阅读es6的代码。  
但是es6代码浏览器支持并不友好。所以需要配置webpack打包成es5。同时需要配上sourceMap。这样，既可以让源码跑在浏览器环境，也可以直接读es6的代码，而且可以随时打断点，查看变量里保存的值。

那么，闲言少叙，开始本章的主题。

在配置调试环境的过程中，参考了许多相关资料，这里先列出来，感兴趣的同学可以参考。

- 主流程相关的资料,点击跳转,同时致敬以下用户
    
    - [知乎用户“AirCloud”](https://zhuanlan.zhihu.com/p/38118887)
    - [github用户“JesseZhao1990”](https://github.com/JesseZhao1990/blog/issues/132)
    - [github用户"jsonz1993“](https://github.com/jsonz1993/react-source-learn/issues/1)
    - [github用户"nannongrousong"](https://github.com/nannongrousong/blog/issues/1)
- 主流程无关但是收益颇丰的资料
    
    - [以前的这篇文章的重写](https://segmentfault.com/a/1190000015908833)
    - [react官方的codebase-overview](https://zh-hans.reactjs.org/docs/codebase-overview.html)
    - [react仓库代码的一次重要变更，这个放后面讲](https://github.com/facebook/react/commit/42c3c967d1e4ca4731b47866f2090bc34caa086c#diff-e09a38ce727b5712192571085fb7bbcf)
    - [一个报错函数的修改](https://github.com/facebook/react/pull/15071)

本人所在测试环境为mac，其他环境类似,调试版本**React Version 16.9.0**  
需要准备的一些环境： Node/npm/create-react-app/git。  
ps：本文是对参考资料的梳理以及优化，力求言简意赅。

- Fork react的仓库到自己的git上。（为了方便自己做记录,fork后自己的git上也会有react，改完后可以往自己的这个上push，如果是clone，则没有权限push,例如我fork出来的地址为`git@github.com:pws019/react.git`）.
- `npx create-react-app my-app`（利用`create-react-app`创建自己的demo项目）
- `cd my-app`(进入上一步创建出来的my-app目录)
- `yarn run eject`(将webpack的配置提取出来,执行完后项目中会多一个config文件夹，存放webpack相关脚本)
- 进入到项目的`src`目录，`git clone git@github.com:pws019/react.git(替换为你刚fork的react路径)`
- 为`/config/webpack.config.js`中的`resolve`选项增加`alias`如下(为了让项目中的引用的react是源码包里的react)：

```
（{
    xxxx: 'xxx',
    resolve: {
        alias: {
            
            
            
            'react': path.resolve(__dirname, '../src/react/packages/react'),
            'react-dom': path.resolve(__dirname, '../src/react/packages/react-dom'),
            'legacy-events': path.resolve(__dirname, '../src/react/packages/legacy-events'),
            'shared': path.resolve(__dirname, '../src/react/packages/shared'),
            'react-reconciler': path.resolve(__dirname, '../src/react/packages/react-reconciler'),
            
            
        },
    },
    xxxx: 'xxx',
}）
```

- 将上一步文件中的`devtool`的值改为`source-map`(为了让打出的包有sourcemap)
- 为`/config/env.js`中的`stringifed`对象增加属性:

```
const stringified = {
    'xxxx': 'xxx',
    "__DEV__": true,
    "__PROFILE__": true,
    "__UMD__": true
};
```

- 由于react的源码中采用了`flow`这个东东做类型检查，执行`yarn add @babel/plugin-transform-flow-strip-types -D`，安装对应的babel插件忽略flow的类型检查，并且在webpack的`babel-loader`中增加该插件

```
{
    test: /\.(js|mjs|jsx|ts|tsx)$/,
    include: paths.appSrc,
    loader: require.resolve('babel-loader'),
    options: {
        customize: require.resolve(
            'babel-preset-react-app/webpack-overrides'
        ),

        plugins: [
            [
            require.resolve('babel-plugin-named-asset-import'),
            {
                loaderMap: {
                svg: {
                    ReactComponent:
                    '@svgr/webpack?-svgo,+titleProp,+ref![path]',
                },
                },
            },
            ],
            [require.resolve('@babel/plugin-transform-flow-strip-types')] 
        ],
        
        
        
        cacheDirectory: true,
        cacheCompression: isEnvProduction,
        compact: isEnvProduction,
    },
},
```

- 注释掉`webpack`中`module.rules[1]`，也就是eslint的配置，因为我也没搞明白，怎么配，总报错。
- 这时候执行`npm start`启动项目，会发现报错，主要有三处，依次解决之。
    
    - 修改文件`/src/react/packages/react-reconciler/src/ReactFiberHostConfig.js`。注释中说明，这块还需要根据环境去导出HostConfig。
    
    ```
    export * from './forks/ReactFiberHostConfig.dom';
    ```
    
    - 修改文件`/src/react/packages/shared/ReactSharedInternals.js`。react此时未export内容，直接从ReactSharedInternals拿值
    
    ```
    
    
    import ReactSharedInternals from '../react/src/ReactSharedInternals';
    ```
    
    - checkReact文件报错，是由于`src/react/packages/shared/invariant`中的函数，直接抛错，这里纠结了好久才找到解法，改下这个函数的内容为
    
    ```
    export default function invariant(condition, format, a, b, c, d, e, f) {
        if(condition) return;
        throw new Error(
            'Internal React error: invariant() is meant to be replaced at compile ' +
            'time. There is no runtime version.',
        );
    }
    ```
    

配这个环境的时候,github用户`nannongrousong`的那篇文章给了我很多帮助，基本是按着他的配的，但是在最后，总是在react-dom的checkReact中总报错，这个问题困扰了我很久。

后来突然想起来，官方的文档里提到**当 invariant 判别条件为 false 时，会将 invariant 的信息作为错误抛出**,而我调试的那个地方，值为true依然报错了。

后来查看了github用户`nannongrousong`提供的调试环境成品库，主要差距就是`src/react/packages/shared/invariant`里函数的实现。

后来我去翻了该文件的`commit history`，发现了这个文件的改动历史，发现是由于有同学想减少包的大小，更好的归拢错误提示，而吧这里抽出来，由自动化工具去替换。

感兴趣的可以看下[这里](https://github.com/facebook/react/commit/42c3c967d1e4ca4731b47866f2090bc34caa086c#diff-e09a38ce727b5712192571085fb7bbcf),  
通过这里的相关代码，可以看到，他是利用ast语法树的分析&替换，去通过工具处理了错误，这个思路值得我们学习借鉴。

[点我跳转](https://github.com/pws019/react-sourcecode-debug-env)  
这个仓库是已经配好的环境，安装完依赖包就可以开始。
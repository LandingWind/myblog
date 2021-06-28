# React源码阅读笔记


记录React源码阅读的一些收获 always updating...

## Flow and Rollup

### flow

flow是辅助JavaScript的静态类型检查工具，相对于TypeScript来说，不需要改写全部的功能模块

而且使用Flow只需要依赖于Babel的插件，具有极大的灵活性

##### 怎么用？

```bash
# 安装flow
npm install --save-dev flow-bin
# package.json添加对应的script
# init
npm run flow init
# babel安装依赖插件
npm install --save-dev babel-preset-flow
#  .babelrc添加flow的preset
```

配置好了Flow，只要在要使用的模块里`@flow`注解，然后增加想要的类型检查

```javascript
/* @flow */

function add(x: number, y: number): number {
  return x + y
}

add(22, 11)
```

注意：Flow拥有上下文推断类型的能力

### rollup

打包工具

开发库用rollup 开发项目用webpack 

rollup主要优点是体积小 但是不支持热加载



## React源码目录结构

```
├── build -------------------------------  构建后的输出目录
├── fixtures ----------------------------  React 开发的测试用例
├── packages ----------------------------  源码目录
│   ├── create-subscription -------------  在组件里订阅额外数据的工具
│   ├── events --------------------------  事件处理 
│   ├── interaction-tracking ------------  跟踪交互事件
│   ├── react ---------------------------  核心代码
│   ├── react-art -----------------------  矢量图形库
│   ├── react-dom -----------------------  DOM 渲染相关
│   ├── react-is ------------------------  React 元素类型相关
│   ├── react-native-renderer -----------  react-native 渲染相关 
│   ├── react-noop-renderer -------------  Fiber 测试相关 
│   ├── react-reconciler ----------------  React 调制器
│   ├── react-scheduler -----------------  规划 React 初始化，更新等等
│   ├── react-test-renderer -------------  实验性的 React 渲染器
│   ├── shared --------------------------  通用代码
│   ├── simple-cache-provider -----------  为 React 应用提供缓存
│   ├── server --------------------------  服务端渲染
│   ├── sfc -----------------------------  .vue 文件解析
│   ├── shared --------------------------  整个项目通用代码
├── scripts -----------------------------  公共的lint，build，test和release等相关的文件
│   ├── eslint --------------------------  语法规则和代码风格
│   ├── flow ----------------------------  Flow 类型声明
│   ├── git -----------------------------  git钩子的目录
│   ├── jest ----------------------------  JavaScript 测试目录
│   ├── release -------------------------  自动发布新版本脚本
│   ├── rollup --------------------------  rollup 构建目录
├── .babelrc ----------------------------  babel 配置文件
├── .editorconfig -----------------------  编辑器语法规范配置
├── .eslintignore -----------------------  eslint 忽略配置 
├── .eslintrc ---------------------------  eslint 配置文件
├── .gitattributes ----------------------  给 attributes 路径名的简单文本文件
├── .gitignore --------------------------  git 忽略配置
├── .mailmap ----------------------------  邮件列表档案 
├── .nvmrc ------------------------------  nvm 配置文件
├── .prettierrc.js ----------------------  prettierrc 配置文件
├── .watchmanconfig ---------------------  watchman 配置文件
├── appveyor.yml ------------------------  GitHub 托管项目的自动化集成
├── AUTHORS -----------------------------  开发者列表档案
├── CHANGELOG.md ------------------------  更新日志
├── CODE_OF_CONDUCT.md ------------------  Code of Conduct
├── CONTRIBUTING.md ---------------------  Contributing to React
├── dangerfile.js -----------------------  提高 Code Review 体验
├── netlify.toml ------------------------  持续集成静态网站
├── package-lock.json -------------------  npm 加锁文件
├── package.json ------------------------  项目管理文件
├── README.md ---------------------------  项目文档
├── yarn.lock ---------------------------  yarn 加锁文件
```

由于将react-dom单独提取出来了，所以react本身的核心代码只有几千行，先来看看咯



## react核心代码

### 对外暴露的API

react对外是一个Javascript对象，暴露了它的API，位置在`packages/react/src/index.js`

```javascript
export {
  Children, //处理props.children的方法
  createRef, //新的ref用法
  Component, //组件基类
  PureComponent, //组件基类
  createContext, //新的context用法
  forwardRef, //ref传递
  lazy,
  memo,
  useCallback,
  useContext,
  useEffect,
  useImperativeHandle,
  useDebugValue,
  useLayoutEffect,
  useMemo,
  useReducer,
  useRef,
  useState,
  useMutableSource,
  createMutableSource,
  Fragment,
  Profiler,
  StrictMode,
  Suspense,
  createElement, //创建element
  cloneElement, //克隆element
  isValidElement, //验证是否是react element
  version,
  __SECRET_INTERNALS_DO_NOT_USE_OR_YOU_WILL_BE_FIRED,
  createFactory, //创建某一类element
  useTransition,
  useDeferredValue,
  SuspenseList,
  unstable_withSuspenseConfig,
  block,
  DEPRECATED_useResponder,
  DEPRECATED_createResponder,
  unstable_createFundamental,
  unstable_createScope,
} from './src/React';
```

可以看到许多由于最近加了hooks多了不少use开头的API



### Component和PureComponent

```javascript
function Component(props, context, updater) {
  this.props = props;
  this.context = context;
  this.refs = emptyObject;
  this.updater = updater || ReactNoopUpdateQueue;
}
Component.prototype.isReactComponent = {};
// setState
Component.prototype.setState = function(partialState, callback) {
  // partialState:产生下一个state的部分state
  // callback:update state以后的回调函数
  invariant(
    typeof partialState === 'object' ||
      typeof partialState === 'function' ||
      partialState == null,
    'setState(...): takes an object of state variables to update or a ' +
      'function which returns an object of state variables.',
  );
  this.updater.enqueueSetState(this, partialState, callback, 'setState');
};
// forceUpdate
Component.prototype.forceUpdate = function(callback) {
  this.updater.enqueueForceUpdate(this, callback, 'forceUpdate');
};
```

pureComponent继承了component的prototype，中间造了个dummyComponent（不知道有什么用）

```javascript
function PureComponent(props, context, updater) {
  this.props = props;
  this.context = context;
  this.refs = emptyObject;
  this.updater = updater || ReactNoopUpdateQueue;
}

const pureComponentPrototype = (PureComponent.prototype = new ComponentDummy());
pureComponentPrototype.constructor = PureComponent;
Object.assign(pureComponentPrototype, Component.prototype);
pureComponentPrototype.isPureReactComponent = true;
```

#### `两者使用上有什么差别`

可以看到源码的定义是几乎完全一样，但是根据渲染的代码可以知道pureComponent在某些情况下可以提升性能

通过浅比较（react定义的shallowEqual）

```javascript
if (ctor.prototype && ctor.prototype.isPureReactComponent) {
  return (
    !shallowEqual(oldProps, newProps) || !shallowEqual(oldState, newState)
  );
}
```

所谓浅比较就是引用发生改变的时候会判定不同，否则相同

可以参考一下下面的例子👇

```jsx
class BasicType extends React.PureComponent{
  constructor() {
    super();
    this.state = {
      isShow: false
    };
    console.log('constructor BasicType');
  }
  changeState = () => {
    this.setState({
      isShow: true
    })
  };
  render() {
    // render will not be output after first render
    console.log('render');
    return (
      <div>
        <button onClick={this.changeState}>点击</button>
        <div>{this.state.isShow.toString()}</div>
      </div>
    );
  }
}

class ArrayType extends React.PureComponent{
  constructor() {
    super();
    this.state = {
      arr: [1,2,3]
    };
    console.log('constructor ArrayType');
  }
  changeState = () => {
    const tarr = this.state.arr;
    tarr.push(1);
    this.setState({
      arr: tarr
    })
    console.log("this.state:",this.state.arr)
  };
  render() {
    console.log('render');
    return (
      <div>
        <button onClick={this.changeState}>点击</button>
        <div>{this.state.arr.toString()}</div>
      </div>
    );
  }
}
const R = () => {
  return (
    <div>
      <BasicType />
      <ArrayType />
    </div>
  )
}
ReactDOM.render(
  <R/>,
  document.getElementById('root')
);
```

所以PureComponent适用场景是展示组件或者引用数据不发生改变的地方

比如注意下面这个例子👇

我们不希望每秒钟子组件Child也跟着渲染

```jsx
class ABC extends React.Component {
    constructor(props){
        super(props);
        this.state = {
            date : new Date()
        }
    }

    componentDidMount(){
        setInterval(()=>{
            this.setState({
                date:new Date()
            })
        },1000)
    }

    render(){
        return (
            <div>
                <Child seconds={1}/>
                <div>{this.state.date.toString()}</div>
            </div>
        )
    }
}
```



### memo

取的名字有点意思——备忘录，可能就是代表规模小的数据展示组件吧

react核心代码里只写了定义

```javascript
export function memo<Props>(
  type: React$ElementType,
  compare?: (oldProps: Props, newProps: Props) => boolean,
) {
  if (__DEV__) {
    if (!isValidElementType(type)) {
      console.error(
        'memo: The first argument must be a component. Instead ' +
          'received: %s',
        type === null ? 'null' : typeof type,
      );
    }
  }
  return {
    $$typeof: REACT_MEMO_TYPE,
    type,
    compare: compare === undefined ? null : compare,
  };
}
```

所以就是传入一个react element和一个props自定义比较函数，返回一个对象

使用起来就是这样👇

```jsx
function Child({seconds}){
    console.log('I am rendering');
    return (
        <div>I am update every {seconds} seconds</div>
    )
};

function areEqual(prevProps, nextProps) {
    if(prevProps.seconds===nextProps.seconds){
        return true
    }else {
        return false
    }
}
export default React.memo(Child,areEqual)
```

看上去就是一个针对函数组件的PureComponent呢



### createElement

传入type、config和children，return一个ReactElement

```javascript
function createElement(type, config, children)
{
  // 这里是对config和children的一些处理
  return ReactElement(
    type,
    key,
    ref,
    self,
    source,
    ReactCurrentOwner.current,
    props,
  );
}
```

可以看到传入config中的key和ref被保留了下来

我们看看ReactElement是什么？

#### ReactElement

```javascript
const ReactElement = function(type, key, ref, self, source, owner, props) {
  const element = {
    // This tag allows us to uniquely identify this as a React Element
    $$typeof: REACT_ELEMENT_TYPE,
    // Built-in properties that belong on the element
    type: type,
    key: key,
    ref: ref,
    props: props,
    // Record the component responsible for creating this element.
    _owner: owner,
  };
  return element;
};
```

1. `type`类型，用于判断如何创建节点
2. `key`和`ref`这些特殊信息
3. `props`新的属性内容
4. `$$typeof`用于确定是否属于`ReactElement`



### Children

前面说到这里提供了几个处理children的API

```javascript
const Children = {
  map,
  forEach,
  count,
  toArray,
  only,
};
```

先看下比如map的使用

```jsx
var NotesList = React.createClass({
  render: function() {
      return (
        <ol>
          {
            React.Children.map(this.props.children, function (child) {
              return <li>{child}</li>;
            })
          }
        </ol>
      );
  }
});

React.render(
  <NotesList>
      <span>hello</span>
      <span>hello</span>
  </NotesList>,
  document.body
);
```



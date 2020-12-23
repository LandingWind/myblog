# recoiljs 快速上手指南


尝试了最近拥抱hooks的react官方状态管理工具Recoil [todolist demo代码仓库](https://github.com/wkk5194/Recoil-TodoList-Demo)

## recoiljs 快速上手指南

> react官方的实验性状态管理库，拥抱hooks

### Problem

与react本身state管理相比...

share state的问题，需要传到公共节点，以致于需要维护一个huge node tree，一个是麻烦复杂，一个是对于状态改变的重渲染不友好

一个context只能解决一个single value的生产消费问题

这两个问题导致代码不能**细粒度拆分**

### Issue

定义了**有向图**（data-flow graph）来储存状态以及状态的依赖关系...（常规思路...）注意不用**树**是因为需要解决state的交叉依赖关系

### Pros

符合react hook语法， boilerplate-free API

支持新出的Concurrent Mode

状态非常细粒度，方便代码切分

同步/异步数据来源的 state 不影响使用的组件的代码

state数据容易做持久化



开始一些概念...



### atom and selector

atom，意即原子（努力把state拆分为最小粒度）

```js
const fontSizeState = atom({
  key: 'fontSizeState',
  default: 14,
})
```

一旦atom中的state发生改变，使用atom的所有react组件都会被重新渲染

比起useState，key是用来调试、数据持久化、查看atom树的全局唯一值（不知道为什么不由recoil自己来生成...

使用atom用useRecoilState，用法和useState一致，毕竟是官方库

```react
function FontButton() {
  const [fontSize, setFontSize] = useRecoilState(fontSizeState);
  return (
    <button onClick={() => setFontSize((size) => size + 1)} style={{fontSize}}>
      Click to Enlarge
    </button>
  );
}
```



selector，输入atom或者其他selector，通过pure function映射数据，得到一个新的selector

其实atom就是特殊的selector，只是它是源数据节点

所有订阅了selector的组件当selector数据发生变动的时候同样会重新渲染

```js
// data stream: 
// fontSizeState(atom) --> fontSizeLabelState(selector) --> react function component
const fontSizeLabelState = selector({
  key: 'fontSizeLabelState',
  get: ({get}) => {
    const fontSize = get(fontSizeState);
    const unit = 'px';

    return `${fontSize}${unit}`;
  },
});
```

由于selector是不可写的，所以我们使用useRecoilValue()

```react
function FontButton() {
  const [fontSize, setFontSize] = useRecoilState(fontSizeState);
  const fontSizeLabel = useRecoilValue(fontSizeLabelState);

  return (
    <>
      <div>Current font size: ${fontSizeLabel}</div>

      <button onClick={() => setFontSize(fontSize + 1)} style={{fontSize}}>
        Click to Enlarge
      </button>
    </>
  );
}
```



说到这里我们可以思考一下，其实如果单单谈论state依赖关系和数据流动，atom和selector是可以合并的

selector旨在做的是解决异步数据来源，就像redux-promise, redux-saga在redux中做的事情

但是从纯函数的角度来考虑，异步数据不应该嵌套在纯函数映射的data flow中，官方这样的解决方案让人存疑...



### demo code

写了一个简单的TodoList应用来看看怎么使用Recoil

[完整代码链接](https://github.com/wkk5194/Recoil-TodoList-Demo)



#### RecoilRoot

需要使用Recoil进行状态管理的Component，首先用RecoilRoot包裹。

注意：Recoil支持多个根例，会搜索最近的root

```react
function App() {
  return (
    <div className="App" style={{textAlign:"center"}}>
      <h3>The TodoList based on React && Recoil</h3>
      <RecoilRoot>
        <TodoList></TodoList>
      </RecoilRoot>
    </div>
  );
}
```



#### atom定义和使用

首先定义listdata的atom，放在store文件夹下（当然想放哪里都可以，统一放在store中方便管理）

```js
const listDataState = atom({
    key: "listDataState", default: [
        { id: 1, label: "learn react and recoil", visible: true, isDone: false },
        { id: 2, label: "play GTA5", visible: true, isDone: true },
        { id: 3, label: "listen to music", visible: true, isDone: false },
        { id: 4, label: "hangout with gf", visible: true, isDone: true }
    ]
})
export {listDataState}
```



通过useRecoilState开始管理该atom，然后map一下list

```react
function todoList() {
	const [listData, setListData] = useRecoilState(listDataState);
	return (
		<div className="list">
            <ul>
                {listData.map((item, index) => {
                    return (
                        <li key={index + md5(item.label)}><TodoItem itemdata={item} /></li>
                    )
                })}
            </ul>
     {/* other code */}
     </div>
	)
}
```



#### 修改state

然后我们现在想实现todolist的标记完成、添加、删除的功能，也就是对state进行修改

添加

```react
// 在todoList中
<span>
  <input
    value={inputData}
    onChange={(value) => setInputData(value.target.value)}
    placeholder="todo...">
  </input>
  <button onClick={() => {
      const newListData = [...listData, cstrTodoItem(md5(inputData), inputData)];
      setListData(newListData);
      setInputData("");
    }}>Add</button>
</span>
```

删除、标记完成、标记未完成

```react
// 在todoItem中
function getNewListData(id, listData) {
    const list = List(listData);
    return list.update(
        list.findIndex((value,_)=>value.id===id),
        (item)=>{
            return {...item, isDone: !item.isDone}
        }
    );
}

function delFromListData(id, listData) {
    return listData.filter(item => item.id !== id);
}

function TodoBtn(id, isDone) {
    const listData = useRecoilValue(listDataState);
    const setListDataState = useSetRecoilState(listDataState);

    const type = !isDone ? "done" : "redo"
    return (
        <span style={{ marginLeft: 30 }}>
            <button onClick={() => {
                setListDataState(getNewListData(id, listData))
            }}>{type}</button>
            <button onClick={() => {
                setListDataState(delFromListData(id, listData))
            }}>del</button>
        </span>
    )
}
function TodoItem(props) {
    const { id, label, visible, isDone } = props.itemdata;
    return (
        <div className="item">
            <div>
                {!isDone ? label : <s>{label}</s>}
            </div>
            <div>
                {TodoBtn(id, isDone)}
            </div>
        </div>
    )
}
```



#### 接下来，尝试一下selector依赖atom

做一个筛选todo item的功能

定义两个新的state，分别是筛选后的列表数据、筛选值，放在store中

```js
const filterState = atom({
    key: "filterState", default: "all"
})

const filteredListState = selector({
    key: "filteredListState",
    get: ({ get }) => {
        const filter = get(filterState);
        const list = get(listDataState);
        switch (filter) {
            case 'all':
                return list
            case 'undos':
                return list.filter((item) => !item.isDone);
            default:
                return list;
        }
    },
})
```

然后在todoList中map list用filteredListState替代原来的list，再增加修改过滤值的逻辑即可

```react
function todoList(){
		const [filter, setFilter] = useRecoilState(filterState);
    const filteredListData = useRecoilValue(filteredListState);
 	 	{/* other code */}
		return (
      <div className="list">
        {/* other code */}
        <ul>
        {filteredListData.map((item, index) => {
        return (
          <li key={index + md5(item.label)}><TodoItem itemdata={item} /></li>
    )
  })}
       </ul>
        <select value={filter} onChange={({ target: { value } }) => setFilter(value)}>
          <option value="all">All</option>
          <option value="completed">Completed</option>
          <option value="undos">Uncompleted</option>
        </select>
    	</div>
    )
}
```



#### 接下来再尝试一下selector获取异步数据的能力

把默认值放在mock中

```js
// mocker/index.js
const proxy = {
    'GET /remotelistdata': {data: [
        { id: 1, label: "from remote planet", visible: true, isDone: false },
        { id: 2, label: "sing from moon", visible: true, isDone: false },
    ]}
}

module.exports = proxy;
```

增加异步获取数据的selector

```js
const queryListDataState = selector({
    key: "queryListDataState",
    get: async () => {
        const res = await axios.get('/remotelistdata');
        console.log(res);
        return res.data.data;
    }
})
```

判断加载完成后，设置远程数据替换掉本地默认数据

```js
const queryListData = useRecoilValueLoadable(queryListDataState);
if(queryListData.state === "hasValue") {
  setListData(queryListData.getValue())
}
```



### 评价

#### pros

- selector这种Derived State的设计，适合处理依赖性逻辑，方便快速开发
- 将state拆分为最小状态，并且可以随意分散管理，应用的模块分割可以进行得更加彻底
- 非常适合取代state层级很深的redux管理模块，由于状态细粒度，性能可以得到很好的优化，可以在应用内小范围使用与redux兼容
- 支持react concurrent模式

#### cons

- API太多了，不想上手
- 使用state需要引入定义的atom/selector以及use... API，复杂，不想上手
- 现在不支持ts

#### btw

recoil的实现原理是发布订阅的方式，许多概念像不像rxjs？atom、selector，还有官方给的waitAll，waitAny等等API



> 接下来和reto/mobx对比一下...
>
> 还想测试一下recoil的性能...
>
> 然后讨论一下recoil的实现原理...

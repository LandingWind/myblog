# Recoil vs Hox



### 写在前面

Recoil作为React官方的试验性状态管理库，想法新颖

Hox作为蚂蚁体验技术部的成果，业务经验丰富

到底使用起来哪一个比较顺手呢？

这是一篇不严谨的React state manager体验文


### 概况对比

#### Recoil

[源码在此](https://github.com/facebookexperimental/Recoil)

定义atom作为state数据源头，切割大state为最小单元

定义selector定义函数映射后的atom，相当于computed atom

从atom-selector-selector链路来看定义了一个state data flow map，但是要注意的是它实际是支持双向衍生的，也就是说数据流动可以是双向的，selector同样可以去set atom

所以，我们必须注意**依赖循环**！



#### Hox

基于原生的React Hooks——useState、useEffect进行state存储以及依赖更新

采用subscriber的数据订阅模式

此外通过Object.defineProperty进行获取状态数据时的拦截

推荐阅读其极其精简的[源码](https://github.com/umijs/hox)



### TodoList Demo对比

通过实际案例来对比，显得极其清楚

我们做一个todolist



##### 首先看看Recoil

[源码](https://github.com/wkk5194/Recoil-TodoList-Demo)

目录结构如下，mocker中存放了模拟后端数据，src/store文件存放了所有atom/selector

{{<image src="https://i.loli.net/2020/12/25/2wYphFbLQaTHA7l.png" caption="recoil-todolist-codetree" >}}


使用recoil的节点需要被包裹

```react
<RecoilRoot>
  <TodoList></TodoList>
</RecoilRoot>
```

定义todolist state atom

```js
const listDataState = atom({
    key: "listDataState", default: [
        { id: 1, label: "learn react and recoil", visible: true, isDone: false },
        { id: 2, label: "play GTA5", visible: true, isDone: true },
        { id: 3, label: "listen to music", visible: true, isDone: false },
        { id: 4, label: "hangout with gf", visible: true, isDone: true }
    ]
})
```

使用atom，然后展示

```react
const [listData, setListData] = useRecoilState(listDataState);
return (
        <div className="list">
            <ul>
                {filteredListData.map((item, index) => {
                    return (
                        <li key={index + md5(item.label)}><TodoItem itemdata={item} /></li>
                    )
                })}
            </ul>
    		</div>
 // ...following code
)
```

TodoItem中通过useSetRecoilState更新atom中数据，从而实现todolist的toggle、delete

```react
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
```

定义筛选的state，及相关维护

```js
// store.js
const filterState = atom({
    key: "filterState", default: "all"
})
// TodoList.js
const [filter, setFilter] = useRecoilState(filterState);
	// ...
return (
  <select value={filter} onChange={({ target: { value } }) => setFilter(value)}>
    <option value="all">All</option>
    <option value="completed">Completed</option>
    <option value="undos">Uncompleted</option>
  </select>
)
```

定义筛选后的todolist selector，依赖于原有的listdata atom和filter atom

```js
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

map的listdata替换为filteredListState



##### 再来看看hox

[源码](https://github.com/wkk5194/Hox-TodoList-Demo)

目录结构

Model中存放hox定义的state manager model

{{<image src="https://i.loli.net/2020/12/25/tMJNF7bmfpvKkAw.png" caption="hox-todolist-codetree" >}}


useTodoModel中，将所有和todolist有关功能逻辑全部放在一块

```js
import { useState } from 'react'
import { createModel } from "hox";
import md5 from 'blueimp-md5'


function useTodo() {
    const defaultTodos = [
        { id: 1, label: "learn react and recoil",isDone: false },
        { id: 2, label: "play GTA5",isDone: true },
        { id: 3, label: "listen to music",isDone: false },
        { id: 4, label: "hangout with gf",isDone: true }
    ]
    const [todos, setTodos] = useState(defaultTodos);
    const [filter, setFilter] = useState('all');

    function addTodo(content) {
        setTodos([...todos, {
            id: md5(new Date().toTimeString()),
            label: content,
            isDone: false
        }])
    }
    function delTodo(id) {
        setTodos(todos.filter(item => item.id !== id))
    }
    function toggleTodo(id) {
        setTodos(todos.map(item => {
            if (item.id === id) {
                return {
                    ...item,
                    isDone: !item.isDone
                }
            } else {
                return item;
            }
        }))
    }
    function filteredTodo() {
        return todos.filter(item => {
            if(filter==='undos') {
                return !item.isDone;
            }
            if(filter==='completed') {
                return item.isDone;
            }
            return item;
        })
    }
    return {
        todos: filteredTodo(),
        setTodos,
        addTodo,
        delTodo,
        toggleTodo,
        setFilter,
        filter
    }
}

export default createModel(useTodo);
```



接下来具体环境中就只是useTodoModel的接口调用，不予以赘述了...



### 思考

#### recoil 走的是学院派的路线：

将state切分最小化atom，定义selector构建媲美计算属性的依赖关系；这样一来可以轻松切割应用内功能模块，甚至可以在模块内继续切割，但是注意不利于抽离业务数据逻辑

主创人员显然走的是纯函数的思考模式，在处理副作用这一块使用起来不太方便，有一定缺陷，但是根据技术论坛消息有望在下一大版本迭代中更新

双向衍生数据，明晰的数据流动图，可追溯的时光机器以及手动定义key的方便数据持久化等等功能，学院派无疑，但是在业务中没有很大的用处，而且一些为了功能pure做出的妥协增加了开发的工作量，但是我们有希望看到recoil二次封装库解决这些问题



#### hox走的是符合用户习惯的路线：

实现方式采用原生react hook，使用起来只需一个API（部分依赖则需要两个API）极其精简

hox同样支持细粒度切分state（虽然不建议切分成atom这么细粒度），支持state依赖，支持归类业务数据逻辑并抽离出来（像是自定义了一个hook）

支持recoil还不支持的ts和class状态管理（recoil得下下大版本了吧）

也支持数据持久化，但是没有recoil手动模式这么定制化，可扩展的优化性没有那么高



总的来说hox目前还是优于recoil，API更友好，类umi的模式也十分有助于项目迁移。但是学院派recoil是值得尝试的道路，我们可以期待在recoil上二次封装适应业务开发表现。

## 语法糖

### 展开语法

* ...<数组> 或者 ...<对象>可以将数组/对象中的每一个元素/属性进行展开
* 可以利用展开语法来创建新的副本

## 组件通信

### 父子组件通过props通信

* 标签中本身就有许多预定义存在的props，即标签本身的一些“属性”(className,src,alt...)
* 父组件中引用子组件，利用<property> = {<value>}的形式来向子组件标签传入参数
* 子函数组件中用“{ }”来包含这些自定义的property(属性),即可直接在子组件中使用参数

```react
//父组件
export default function Profile() {
  return (
    <Avatar
      person={{ name: 'Lin Lanying', imageId: '1bX5QH6' }}
      size={100}
    />
  );
}
```

```react
//子组件
function Avatar({ person, size }) {
  // 在这里 person 和 size 是可访问的
}
```

* 实际上React组件是接受一个props对象，里面包含了各种属性的“键值对”

```react
function Avatar(props) {
  let person = props.person;
  let size = props.size;
}
```

* 子组件定义具体传入的property可以给予一个默认值
* 展开语法

```react
function Profile(props) {
  return (
    <div className="card">
      <Avatar {...props} />  //直接将props所有“键值对”展开
    </div>
  );
}
```

* 组件嵌套的情况下，要被嵌入的组件会在名为children的property中传入

## JavaScript & React数组操作

* 数组复制是浅拷贝(每一项底层对象是一样的，而不是重新创建一个独立的对象)，即对数组中每一个项实行的是浅拷贝，但是创建了一个新的数组，数组中的每一项是原本数组的浅拷贝
* 复制得到的新数组和原本的数组不是同一个数组，但是里面的对象是同一个对象

### 过滤

* 利用filter来对数组进行过滤，重新创建一个只满足特定条件的数组

  ```react
  //filter传入一个箭头函数
  //参数是当前正在处理的数组中的元素对象
  //箭头函数体里面返回需要满足过滤的条件
  //返回创建出来的满足特定条件的数组，但底层元素是浅拷贝
  const chemists_1 = people.filter(person =>
    person.profession === '化学家'
  );
  //或者
  const chemists_2 = people.filter((person)=>{
      //do something else
      return person.profession ==='化学家'
  })
  
  const people = [{
    id: 0,
    name: 'Creola Katherine Johnson',
    profession: 'mathematician',
  }, {
    id: 1,
    name: 'Mario José Molina-Pasquel Henríquez',
    profession: 'chemist',
  }, {
    id: 2,
    name: 'Percy Lavon Julian',
    profession: 'chemist',  
  }];
  ```

### 遍历

* 利用map来对数组元素进行遍历

  ```React
  //map方法传入一个箭头函数
  //箭头函数的参数是正在处理数组中的元素项
  //箭头函数体中实现具体的逻辑
  //可以利用map来遍历数组返回结构化的标签信息
  const listItems = chemists.map(person =>
    <li>
       <img
         src={getImageUrl(person)}
         alt={person.name}
       />
       <p>
         <b>{person.name}:</b>
         {' ' + person.profession + ' '}
         因{person.accomplishment}而闻名世界
       </p>
    </li>
  );
  ```

## 

## 事件处理函数

* 标签中类似于onClick的属性可以传入事件处理函数，当事件被触发的时候会调用自定义的事件处理函数

* onClick传入的是函数，而不是函数的调用

  ```react
  //传入一个箭头函数
  <button onClick={()=>{alert("触发点击")}}>点击</button>
  
  //传入一个自定义的事件处理函数
  function handleClick(){alert("触发点击")}
  <button onClick={handleClick}>点击</button>
  
  //花括号里是处理函数的调用
  //在组件被渲染的时候就会调用这个函数
  //因为花括号的内容会在渲染时被执行
  //这是一种错误的写法
  <button onClick={handleClick()}>点击</button>
  ```

* 事件处理函数可以作为一个参数传递给子组件

  ```react
  function App(){
      function handleClick(){alert("点击");}
      return (
          <MyButton onClick={handleClick}>
          </MyButton>
      )
  }
  
  function MyButton(props){
      return (
          <button onClick={props.onClick}></button>
      )
  }
  ```

### 阻止事件传播

* 事件在组件中的传播，是由子组件到父组件传播的
* 事件处理函数会接收一个事件对象作为唯一参数，利用这个参数可以读取所触发事件的相关信息
* 调用event.stopPropagation()来截断事件的向上传播

### 阻止浏览器默认行为

* event.preventDefault()

## 状态记忆

### 局部变量失效

* 每一次重新渲染局部变量也会跟着一起从头设置，意味着所有对局部变量的更改失效
* 对局部变量的更改并不会引起React的重新渲染

### React State

* 利用State变量保存渲染之间需要保存的状态

* 用State setter函数来设置新的State值

  ```react
  const [index, setIndex] = useState ( 0 );
  //      ↑         ↑          ↑       ↑		
  //  State变量  设置函数   创建State  给State变量的初始值
  //
  ```

* 声明出来的State局限在当前组件的作用域内，同一个函数中写的State，被创建出来多个标签，各自的State是独立互不影响的

### 渲染过程

#### state"快照"

[state 如同一张快照 – React 中文文档](https://zh-hans.react.dev/learn/state-as-a-snapshot)

* React的渲染过程为：触发渲染 -> 渲染 -> 提交
* 只有初次渲染页面和组件中的状态(State)发生变化，才会触发渲染
* 每一次触发的重新渲染都会被放入渲染的**队列**
* 渲染过程中可能对状态变量做出若干操作，但是操作的结果不会立马反映到对应的变量上，此时读在渲染过程中要被改变的变量会发现还是原始的值，只有渲染结束的最后状态才真正提交更改，这时候状态才会发生改变(官方文档解释：设置state只会为下一次渲染变更state值，不会在本次渲染中将state值变更,**一个 state 变量的值永远不会在一次渲染的内部发生变化**)

#### 一次渲染多次更新同一State

* 对相应的state setter函数传入箭头函数，相当于告诉React根据渲染队列中前一个State来计算下一个State

  ```react
  export default function Counter() {
    const [number, setNumber] = useState(0);
  
    return (
      <>
        <h1>{number}</h1>
        <button onClick={() => {
          setNumber(n => n + 1);
          setNumber(n => n + 1);
          setNumber(n => n + 1);
        }}>+3</button>
      </>
    )
  }
  ```

  

### 更改State中的对象

* 直接对State中的对象值进行修改无用，必须使用setter
* 通过创建一个新的对象，用setter来替代原对象
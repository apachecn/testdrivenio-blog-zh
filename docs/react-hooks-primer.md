# React 挂钩上的底漆

> 原文：<https://testdriven.io/blog/react-hooks-primer/>

React 很棒，但是在开发更大的应用程序时，需要解决一些稍微令人恼火的架构原则。例如，使用复杂的高阶组件来重用有状态逻辑，以及依赖触发不相关逻辑的生命周期方法。嗯，[反应钩](https://reactjs.org/docs/hooks-intro.html)有助于缓解这种恶化...让我们开始吧。

> 这是两部分系列的第一部分:
> 
> 1.  *(本文)*React 挂钩上的底漆
> 2.  [React Hooks——以 useContext 和 useReducer 为特色的深入探讨](https://testdriven.io/blog/react-hooks-advanced)

## 学习目标

在本文结束时，你将能够回答以下问题:

1.  什么是钩子？
2.  为什么要在 React 中实现它们？
3.  钩子是如何使用的？
4.  使用钩子有什么规则吗？
5.  什么是自定义挂钩？
6.  什么时候应该使用定制挂钩？
7.  使用定制钩子有什么好处？

## 什么是钩子？

挂钩允许您:

1.  在功能组件中使用状态和“挂钩”生命周期方法。
2.  在组件之间重用有状态逻辑，这简化了组件逻辑，最重要的是，让您可以跳过编写类。

## 为什么要在 React 中实现它们？

如果你用过 React，你就知道有状态逻辑有多复杂，对吗？当应用程序增加了几个新特性来增强功能时，就会出现这种情况。为了尝试和简化这个问题，React 背后的智囊们试图找到一种方法来解决这个问题。

1.  **重用组件间的有状态逻辑**

    钩子允许开发人员编写简单的、有状态的、功能性的组件，并且在开发时花费更少的时间来设计和重构组件层次结构。怎么会？使用钩子，你可以*提取*和*共享*组件之间的有状态逻辑。

2.  **简化组件逻辑**

    当不可避免的逻辑指数增长出现在您的应用程序中时，简单的组件就变成了令人眩晕的有状态逻辑和副作用的深渊。生命周期方法变得与不相关的方法混杂在一起。一个组件的职责不断增长，变得不可分割。反过来，这使得编码繁琐，测试困难。

3.  **可以逃课**

    类是 React 架构的重要组成部分。上课有很多好处，但是给初学者造成了入门的障碍。对于类，您还必须记住将`this`绑定到事件处理程序，因此代码会变得冗长且有点多余。编码的未来也不会很好地与类一起玩，因为它们可能会鼓励倾向于落后于其他设计模式的模式。

## 钩子是如何使用的？

React 版本 [16.8](https://reactjs.org/blog/2019/02/06/react-v16.8.0.html) 有钩子。

```py
`import  {  useState,  useEffect  }  from  'react';` 
```

很简单，但是你实际上如何使用这些新方法呢？下面的例子很简单，但是这些方法的能力非常强大。

### useState 挂钩方法

使用状态挂钩的最佳方式是将其析构并设置初始值。第一个参数将用于存储状态，第二个用于更新状态。

例如:

```py
`const  [weight,  setWeight]  =  useState(150); onClick={()  =>  setWeight(weight  +  15)}` 
```

1.  `weight`是状态
2.  `setWeight`是一种用于更新状态的方法
3.  `useState(150)`是用来设置初始值的方法(任何原始类型)

值得注意的是，您可以在单个组件中多次析构状态挂钩:

```py
`const  [age,  setAge]  =  useState(42); const  [month,  setMonth]  =  useState('February'); const  [todos,  setTodos]  =  useState([{  text:  'Eat pie'  }]);` 
```

因此，该组件可能类似于:

```py
`import React, { useState } from 'react';

export default function App() {
  const [weight, setWeight] = useState(150);
  const [age] = useState(42);
  const [month] = useState('February');
  const [todos] = useState([{ text: 'Eat pie' }]);

  return (
    <div className="App">
      <p>Current Weight: {weight}</p>
      <p>Age: {age}</p>
      <p>Month: {month}</p>
      <button onClick={() => setWeight(weight + 15)}>
        {todos[0].text}
      </button>
    </div>
  );
}` 
```

### useEffect 挂钩方法

最好像使用任何常见的生命周期方法一样使用效果挂钩，如`componentDidMount`、`componentDidUpdate`和`componentWillUnmount`。

例如:

```py
`// similar to the componentDidMount and componentDidUpdate methods useEffect(()  =>  { document.title  =  `You clicked ${count} times`; });` 
```

每当组件更新时，`useEffect`将在渲染后被调用。现在，如果您只想在变量计数发生变化时更新`useEffect`，您只需将该事实添加到数组中方法的末尾，类似于高阶`reduce`方法末尾的累加器。

```py
`// check out the variable count in the array at the end... useEffect(()  =>  { document.title  =  `You clicked ${count} times`; },  [  count  ]);` 
```

让我们结合两个例子:

```py
`const  [weight,  setWeight]  =  useState(150); useEffect(()  =>  { document.title  =  `You weigh ${weight}, you ok with that?`; },  [  weight  ]); onClick={()  =>  setWeight(weight  +  15)}` 
```

因此，当`onClick`被触发时，`useEffect`方法也将被调用，并在 DOM 更新后不久在文档标题中呈现新的权重。

示例:

```py
`import React, { useState, useEffect } from 'react';

export default function App() {
  const [weight, setWeight] = useState(150);
  const [age] = useState(42);
  const [month] = useState('February');
  const [todos] = useState([{ text: 'Eat pie' }]);

  useEffect(() => {
    document.title = `You weigh ${weight}, you ok with that?`;
  });

  return (
    <div className="App">
      <p>Current Weight: {weight}</p>
      <p>Age: {age}</p>
      <p>Month: {month}</p>
      <button onClick={() => setWeight(weight + 15)}>
        {todos[0].text}
      </button>
    </div>
  );
}` 
```

`useEffect`非常适合进行 API 调用:

```py
`useEffect(()  =>  { fetch('https://jsonplaceholder.typicode.com/todos/1') .then(results  =>  results.json()) .then((data)  =>  {  setTodos([{  text:  data.title  }]);  }); },  []);` 
```

React 钩子看起来很棒，但是如果你花一分钟，你可能会意识到在多个组件中重新初始化多个钩子方法，比如`useState`和`useEffect`，可能会轻视我们都珍视的 DRY 原则。好吧，让我们看看如何通过创建定制钩子来重用这些美妙的新内置方法。在我们开始一些涉及钩子的新技巧之前，我们先来看看使用钩子的规则，让我们的定制钩子之旅更加愉快。

## 使用钩子有什么规则吗？

是的，React 钩子是有规则的。乍一看，这些规则似乎不合常规，但是一旦你理解了 React 钩子是如何启动的，这些规则就很容易遵循了。另外，React 有一个棉绒来防止你违反规则。

#### (1)钩子必须以相同的顺序调用，在顶层。

钩子创建一个钩子调用数组来保持秩序。这种顺序有助于 React 区分，例如，单个组件中或整个应用程序中的多个`useState`和`useEffect`方法调用之间的差异。

例如:

```py
`// This is good! function  ComponentWithHooks()  { // top-level! const  [age,  setAge]  =  useState(42); const  [month,  setMonth]  =  useState('February'); const  [todos,  setTodos]  =  useState([{  text:  'Eat pie'  }]); return  ( //... ) }` 
```

1.  第一次渲染时，`42`、`February`、`[{ text: 'Eat pie'}]`都被推入一个状态数组。
2.  当组件重新呈现时，`useState`方法参数被忽略。
3.  `age`、`month`和`todos`的值从组件的状态中检索，该状态是前述的状态数组。

#### (2)不能在条件语句或循环中调用钩子。

由于钩子的启动方式，它们不允许在条件语句或循环中使用*。对于钩子，如果在重新渲染时初始化的顺序改变了，你的应用程序很有可能不能正常工作。您仍然可以在组件中使用条件语句和循环，但是不能在代码块中使用钩子。*

例如:

```py
`// DON'T DO THIS!! const  [DNAMatch,  setDNAMatch]  =  useState(false) if  (name)  { setDNAMatch(true) const  [name,  setName]  =  useState(name) useEffect(function  persistFamily()  { localStorage.setItem('dad',  name); },  []); }` 
```

```py
`// DO THIS!! const  [DNAMatch,  setDNAMatch]  =  useState(false) const  [name,  setName]  =  useState(null) useEffect(()  =>  { if  (name)  { setDNAMatch(true) setName(name) localStorage.setItem('dad',  name); } },  []);` 
```

#### (3)钩子不能用在类组件中。

钩子必须在函数组件或自定义钩子函数中初始化。自定义钩子函数只能在一个功能组件中调用，并且必须遵循与非自定义钩子相同的规则。

> 您仍然可以在同一个应用程序中使用类组件。你可以用钩子把你的功能组件作为一个类组件的子组件。

#### (4)定制挂钩应以单词*开头，使用*并采用驼色外壳。

这与其说是一个规则，不如说是一个强有力的建议，但是它将有助于应用程序的一致性。您还会知道，当您看到以单词`use`为前缀的函数时，它可能是一个定制的钩子。

## 什么是自定义挂钩？

自定义钩子只是遵循与非自定义钩子相同规则的函数。它们允许您整合逻辑、共享数据以及跨组件重用钩子。

## 什么时候应该使用定制挂钩？

当您需要在组件之间共享逻辑时，最好使用定制挂钩。在 JavaScript 中，当您想要在两个独立的函数之间共享逻辑时，您可以创建另一个函数来支持它。和组件一样，钩子也是函数。您可以提取钩子逻辑，在应用程序的各个组件之间共享。在编写定制钩子时，您命名它们(再次以单词`use`开始)，设置参数，并告诉它们应该返回什么(如果有的话)。

例如:

```py
`import  {  useEffect,  useState  }  from  'react'; const  useFetch  =  ({  url,  defaultData  =  null  })  =>  { const  [data,  setData]  =  useState(defaultData); const  [loading,  setLoading]  =  useState(true); const  [error,  setError]  =  useState(null); useEffect(()  =>  { fetch(url) .then(res  =>  res.json()) .then((res)  =>  { setData(res); setLoading(false); }) .catch((err)  =>  { setError(err); setLoading(false); }); },  []); const  fetchResults  =  { data, loading, error, }; return  fetchResults; }; export  default  useFetch;` 
```

你是不是想出一个需要自定义钩子的情况？发挥你的想象力。虽然在钩子旁边有非常规的规则，但它们仍然非常灵活，并且刚刚开始展示它们的潜力。

## 使用定制钩子有什么好处？

钩子允许你随着应用程序的增长抑制复杂性，编写更容易理解的代码。下面的代码是具有相同功能的两个组件的比较。在第一次比较之后，我们将展示在带有容器的组件中使用自定义钩子的更多好处。

下面的类组件应该看起来很熟悉:

```py
`import React from 'react';

class OneChanceButton extends React.Component {
  constructor(props) {
    super(props);
    this.state = {
      clicked: false,
    };
    this.handleClick = this.handleClick.bind(this);
  }

  handleClick() {
    return this.setState({ clicked: true });
  }

  render() {
    return (
      <div>
        <button
          onClick={this.handleClick}
          disabled={this.state.clicked}
        >
          You Have One Chance to Click
        </button>
      </div>
    );
  }
}

export default OneChanceButton;` 
```

用钩子实现相同的功能来简化代码和增加可读性怎么样:

```py
`import React, { useState } from 'react';

function OneChanceButton(props) {
  const [clicked, setClicked] = useState(false);

  function doClick() {
    return setClicked(true);
  }

  return (
    <div>
      <button
        onClick={clicked ? undefined : doClick}
        disabled={clicked}
      >
        You Have One Chance to Click
      </button>
    </div>
  );
}

export default OneChanceButton;` 
```

### 更复杂的比较

想要更多吗？

1.  [media-query-custom-hooks](https://github.com/testdrivenio/react-hooks/tree/master/media-query-custom-hooks)——这个比较将展示利用`useState`和`useEffect`方法实现定制挂钩的强大功能
2.  [media-query-custom-hooks](https://github.com/testdrivenio/react-hooks/tree/master/media-query-custom-hooks)——一个使用`useRef`和`useReducer`的更复杂的例子

## 结论

React 挂钩是一个惊人的新功能！实施的理由是正当的；此外，我相信这将极大地降低 React 中的编码壁垒，并使其保持在最受欢迎的框架列表的顶端。看到这如何改变第三方库的工作方式，尤其是状态管理工具和路由器，将是非常令人兴奋的。

总之，React 挂钩:

1.  无需使用类组件就可以轻松“挂钩”React 的生命周期方法
2.  通过提高可重用性和抽象复杂性来帮助减少代码
3.  帮助简化组件之间共享数据的方式

我迫不及待地想看到 React 钩子如何被利用的更有力的例子。感谢阅读！

> 准备好了吗？查看下一部分:[React Hooks——以 useContext 和 useReducer](https://testdriven.io/blog/react-hooks-advanced) 为特色的更深入的探讨。
# react Hooks——以 useContext 和 useReducer 为特色的更深入的探讨

> 原文：<https://testdriven.io/blog/react-hooks-advanced/>

本文的主要目的是了解`useContext`和`useReducer`以及它们如何一起工作来使 React 应用程序及其状态管理变得干净和高效。

新的 [Hooks](https://reactjs.org/docs/hooks-reference.html) API 允许跨 React 应用访问一些惊人的特性，这可能会消除对状态管理库的需要，如 Redux 或 Flux。更好的是，我们可以用功能组件来实现这一点。不需要类组件，只需要 JavaScript。

首先，我们将对 React 应用程序中的“预挂钩”上下文 API、最佳实践和实现进行概述。一旦奠定了基础并描绘了一个示例，该示例将被重构以实现`useContext`。从`useContext`开始，重点关注围绕 reducer 函数的概念，这将导致`useReducer`的实现以及如何使用它来管理复杂状态。在对这两个钩子有了深刻的理解之后，我们将把它们结合起来创建一个只支持 React 的状态管理工具。

> 这是两部分系列的第二部分:
> 
> 1.  [反应钩上的底漆](https://testdriven.io/blog/react-hooks-primer/)
> 2.  *(本文)*React Hooks——以 useContext 和 useReducer 为特色的深入探讨

## 观众

这是面向 React 开发人员的中级教程，对以下内容有基本了解:

1.  反应数据流
2.  `useState`和`useEffect`挂钩
3.  全局状态管理工具和模式(如 Redux 和 Flux)
4.  React 上下文 API

> 如果你是 React 钩子的新手，看看 React 钩子上的[底漆。](https://testdriven.io/blog/react-hooks-primer/)

## 学习目标

本教程结束时，您将能够:

1.  解释什么是语境
2.  确定何时应实施上下文
3.  通过`useContext`钩子实现上下文
4.  确定何时应该实施`useReducer`
5.  实现`useReducer`组件状态管理
6.  利用`useReducer`和`useContext`作为应用状态管理工具
7.  通过实现钩子来简化 React

## 项目概述

演示的项目将是一个“race series”管理工具，到本文结束时，它将为应用程序的状态管理实现`useContext`和`useReducer`。本文开头的项目的当前状态没有使用任何状态管理库，也没有实现上下文或钩子 API。它只是通过道具传递逻辑，并利用 React 的渲染道具。本文不会对整个项目进行分解，而是关注于实现状态管理的`useContext`和`useReducer`所必需的重构部分。

对于完整的项目访问它的回购 *[这里](https://github.com/joha0033/race-series-hooks/tree/master/src)* 。

**什么是赛事系列管理工具？**

简单来说，一个比赛系列有多个比赛和多个参与者(用户)。不是所有的用户都在每个种族。该应用程序将作为一个管理工具，允许管理员访问一个种族列表和一个系列内的用户列表。当管理员登录并选择查看比赛列表时，他们将能够看到该系列中的所有比赛。从列表中选择一个比赛将显示该比赛的注册用户。相反，当选择查看系列时，管理员将能够从用户中进行选择，并查看特定用户正在参加的比赛。

## 综述:基本钩子

如果您不熟悉 Hooks API，请从这里开始。钩子允许您访问状态和生命周期方法，以及许多其他在功能组件中使用的技巧。不需要写类组件。同样，不需要编写类组件！如果你正在寻找一个坚实的基础，从`useState`和`useEffect`开始。

1.  **[使用状态](https://reactjs.org/docs/hooks-state.html) :** 允许访问和控制功能组件内的状态。以前，只有基于类的组件才能访问状态。

2.  **[useEffect](https://reactjs.org/docs/hooks-effect.html) :** 允许抽象访问功能组件内 React 的生命周期方法。您不会按名称调用各个生命周期方法，但是您可以通过更多的功能和更干净的代码获得类似的控制。

示例:

```py
`import  React,  {  useState,  useEffect  }  from  'react'; export  const  SomeComponent  =  ()  =>  { const  [  DNAMatch,  setDNAMatch  ]  =  useState(false); const  [  name,  setName  ]  =  useState(null); useEffect(()  =>  { if  (name)  { setDNAMatch(true); setName(name); localStorage.setItem('dad',  name); } },  [  DNAMatch  ]); return  ( // ... ); };` 
```

> 同样，关于基本挂钩的更多细节，请阅读初级读本:[React 挂钩初级读本](https://testdriven.io/blog/react-hooks-primer/)。

## 使用上下文

在深入上下文挂钩之前，让我们先看一下[上下文](https://reactjs.org/docs/context.html) API 的概述，以及它在[挂钩](https://reactjs.org/docs/hooks-reference.html) API 之前是如何实现的。这将有助于更好地理解`useContext`钩子，以及更深入地了解何时应该使用上下文。

### 上下文 API 概述

React 中一个众所周知的缺陷是访问全局状态对象。当数据需要深入嵌套的组件树(主题、UI 样式或用户授权)时，通过 props 在几个可能需要也可能不需要该数据的组件之间传输数据会非常麻烦；组件变成了数据骡子。首选的解决方案是像 Redux 或 Flux 这样的状态管理库。这些库非常强大，但是在设置、保持有组织性以及知道何时需要实现它们时，可能会有点困难。

这里有一个例子，前上下文 API，道具被偷偷带入子组件的生命深处。

```py
`// NO CONTEXT YET - just prop smuggling import  React,  {  Component,  useReducer,  useContext  }  from  'react'; const  Main  =  (props)  =>  ( <div  className={'main'}> {/* // Main hires a Component Mule (ListContainer) to smuggle data */} <List isAuthenticated={props.isAuthenticated} toggleAuth={props.toggleAuth} /> </div> ); const  List  =  ({  isAuthenticated,  toggleAuth,  shake  })  =>  ( isAuthenticated ?  ( <div  className={'title'}  > "secure"  list,  check. </div >) :  ( <div  className={'list-login-container'}> {/* // And List hires a Component Mule (AdminForm) to smuggle data */} <AdminForm  shake={shake}  toggleAuth={toggleAuth}  /> </div>) ); class  AdminForm  extends  Component  { constructor(props)  { super(props); this.state  =  {}; this.handleSubmit  =  this.handleSubmit.bind(this); } handleSubmit(event,  toggleAuth)  { event.preventDefault(); return  toggleAuth(true); } render()  { return  ( <div  className={'form-container'}> <form onSubmit={event  =>  this.handleSubmit(event,  this.props.toggleAuth)} className={'center-content login-form'} > // ... form logic </form> </div> ); } } class  App  extends  Component  { constructor(props)  { super(props); this.state  =  { isAuthenticated:  false, }; this.toggleAuth  =  this.toggleAuth.bind(this); } toggleAuth(isAuthenticated)  { return  isAuthenticated ?  this.setState({  isAuthenticated:  true  }) :  alert('Bad Credentials!'); } render()  { return  ( <div  className={'App'}> <Main isAuthenticated={this.state.isAuthenticated} toggleAuth={this.toggleAuth} > </Main> </div> ); } } export  default  App;` 
```

### 实现上下文 API

创建 Context API 是为了解决这个全球数据危机，并阻止数据走私、滥用无辜子组件的道具——如果你愿意，这是一个国家紧急情况。太好了，让我们看看。

下面是使用上下文 API 重构的同一个示例。

```py
`// CONTEXT API import  React,  {  Component,  createContext  }  from  'react'; // We used 'null' because the data we need // resides in App's state. const  AuthContext  =  createContext(null); // You can also destructure above // const { Provider, Consumer } = createContext(null) const  Main  =  (props)  =>  ( <div  className={'main'}> <List  /> </div> ); const  List  =  (props)  =>  ( <AuthContext.Consumer> auth  =>  { auth ?  ( <div  className={'list'}> // ... map over some sensitive data for a beautiful secure list </div> ) :  ( <div  className={'form-container'}> // And List hires a Component Mule to smuggle data <AdminForm  /> </div> ) } </AuthContext.Consumer>); class  AdminForm  extends  Component  { constructor(props)  { super(props); this.state  =  {}; this.handleSubmit  =  this.handleSubmit.bind(this); } handleSubmit(event,  toggleAuth)  { event.preventDefault(); return  toggleAuth(true); } render()  { return  ( <AdminContext.Consumer> {state  =>  ( <div> <form onSubmit={event  =>  this.handleSubmit(event,  state.toggleAuth)} className={'center-content login-form'} > // ... form logic </form> </div> )} </AdminContext.Consumer> ); } } class  App  extends  Component  { constructor(props)  { super(props); this.state  =  { isAuthenticated:  false, }; this.toggleAuth  =  this.toggleAuth.bind(this); } toggleAuth(isAuthenticated)  { this.setState({  isAuthenticated:  true  }); } render()  { return  ( <div> <AuthContext.Provider  value={this.state.isAuthenticated}> <Main  /> </AuthContext.Provider> </div> ); } } export  default  App;` 
```

> 随着钩子的引入，这种使用上下文 API 的方法仍然有效。

### 考虑上下文

在实现上下文时，还需要考虑其他一些事情。上下文 API 将重新呈现*作为提供者后代的所有*组件。如果你不小心，你可能会在每次点击或击键时重新渲染整个应用程序。解决方案？你打赌！

在提供者中包装组件时要考虑周全。控制台日志不会杀死猫。

创建一个只接受子道具的新组件。将提供程序及其必要的数据移动到该组件中。这使提供者的子道具在渲染之间保持相等。

```py
`// Navigate.js import  React,  {  useReducer,  useContext  }  from  'react'; const  AppContext  =  React.createContext(); const  reducer  =  (state,  action)  =>  { switch  (action.type)  { case  'UPDATE_PATH':  return  { ...state, pathname:  action.pathname, }; default:  return  state; } }; export  const  AppProvider  =  ({  children  })  =>  { const  [state,  dispatch]  =  useReducer(reducer,  { pathname:  window.location.pathname, navigate:  (pathname)  =>  { window.history.pushState(null,  null,  pathname); return  dispatch({  type:  'UPDATE_PATH',  pathname  }); }, }); return  ( <AppContext.Provider  value={state}> {children} </AppContext.Provider> ); }; export  const  LinkItem  =  ({  activeStyle,  ...props  })  =>  { const  context  =  useContext(AppContext); console.log(context,  'CONTEXT [[email protected]](/cdn-cgi/l/email-protection)'); return  ( <div> <a {...props} style={{ ...props.style, ...(context.pathname  ===  props.href  ?  activeStyle  :  {}), }} onClick={(e)  =>  { e.preventDefault(); context.navigate(props.href); }} /> </div> ); }; export  const  Route  =  ({  children,  href  })  =>  { const  context  =  useContext(AppContext); return  ( <div> {context.pathname  ===  href  ?  children  :  null} </div> ); };` 
```

如果你在下面的模式中使用上述方法，你将不会有不必要的重新渲染(渲染道具`<AppProvider>`里面的一切)。

```py
`// App.js import  React,  {  useContext  }  from  'react'; import  {  AppProvider,  LinkItem,  Route  }  from  './Navigate.js'; export  const  AppLayout  =  ({  children  })  =>  ( <div> <LinkItem  href="/participants/"  activeStyle={{  color:  'red'  }}> Participants </LinkItem> <LinkItem  href="/races/"  activeStyle={{  color:  'red'  }}> Races </LinkItem> <main> {children} </main> </div> ); export  const  App  =  ()  =>  { return  ( <AppProvider> <AppLayout> <Route  href="/races/"> <h1>Off  to  the  Races</h1> </Route> <Route  href="/participants/"> <h1>Off  with  their  Heads!</h1> </Route> </AppLayout> </AppProvider> ); };` 
```

将您的消费元素包装在更高阶的组件中。

```py
`const  withAuth  =  (Component)  =>  (props)  =>  { const  context  =  useContext(AppContext) return  ( <div> {<Component  {...props}  {...context}  />} </div> ); } class  AdminForm  extends  Component  { // ... } export  withAuth(AdminForm);` 
```

Context 永远不会完全取代 Redux 这样的状态管理库。例如，Redux 有助于轻松调试，允许使用中间件进行定制，保持单一状态，并使用`connect`遵循纯函数实践。语境很有能力；它可以保存状态，而且功能惊人地多，但是它最适合作为数据的*提供者*，其中的组件*消费*。

> Render Props 是传递数据的很好的解决方案，但是它可以创建一个有趣的组件树层次结构。想想回调地狱，但与 HTML 标签。如果需要在一个组件树的深处获得数据，可以考虑使用上下文。这也可以提高性能，因为当数据更改时，父组件不会重新呈现。

### 实现 useContext

`useContext`钩子使得消费上下文数据的实现变得容易，并且有助于使组件可重用。为了说明上下文 API 可能带来的困难，我们将展示一个使用多个上下文的组件。在挂钩之前，多上下文消费组件变得难以重用和理解。这就是为什么上下文应该少用的一个原因。

这是上面的一个例子，但是使用了额外的上下文。

```py
`const  AuthContext  =  createContext(null); const  ShakeContext  =  createContext(null); class  AdminForm  extends  Component  { // ... render()  { return  ( // this multiple context consumption is not a good look. <ShakeContext.Consumer> {shake  =>  ( <AdminContext.Consumer> {state  =>  ( // ... consume! )} </AdminContext.Consumer> )} </ShakeContext.Consumer> ); } } class  App  extends  Component  { //  ... <ShakeContext.Provider  value={()  =>  this.shake()}> <AuthContext.Provider  value={this.state}> <Main  /> </AuthContext.Provider> </ShakeContext.Provider> } export  default  App;` 
```

现在想象三个或更多的上下文来消费...[头部爆炸:场景结束]

在游过上下文的海洋之后，深呼吸，陶醉于你的新知识的便捷。这是一个漫长的旅程，如果你回顾我们的努力，它们仍然可见。但随着时间的推移，水面平静下来，屈服于重力。

Context 也做了同样的事情，屈服于 JavaScript。输入`useContext`。

消费上下文的新挂钩并没有改变上下文的概念，因此出现了上面的暴跌。这个上下文挂钩只是给了我们一个额外的、更漂亮的方法来使用上下文。当将它应用于使用多种上下文的组件时，它非常有用。

这里我们有一个上述组件的重构版本，用钩子消耗多个上下文！

```py
`const  AdminForm  =  ()  =>  { const  shake  =  useContext(ShakeContext); const  auth  =  useContext(AuthContext); // you have access to the data within each context // the context still needs be in scope of the consuming component return  ( <div  className={ shake  ?  'shake'  :  'form-container' }> { auth.isAuthenticated ?  user.name :  auth.errorMessage } </div> ); };` 
```

就是这样！无论您的上下文包含什么，无论是对象、数组还是函数，您都可以通过`useContext`访问它。太神奇了。接下来，让我们深入减速器，讨论它的优点，然后实现`useReducer`钩子。

## useReducer

为了使围绕实现的决策更容易，我们将回顾相关概念、实际用例，然后是实现。我们将使用`useState`从一个类到功能组件。接下来，我们将实现`useReducer`。

### 围绕用户的概念

如果您至少熟悉以下三个概念中的两个，您可以跳到下一节。

如果你不相信你对以上任何一点都足够满意，请回顾以下几点进行更深入的研究:

**useReducer 和 useState:**`useState`钩子允许你访问一个功能组件中的一个状态变量，用一个方法来更新它——也就是`setCount`。`useReducer`使更新状态更加灵活和隐式。就像`Array.prototype.map`和`Array.prototype.reduce`可以解决类似的问题一样，`Array.prototype.reduce`的用途要多得多。

> `useState`用途`useReducer`引擎盖下。

下面的例子将任意演示拥有多个`useState`方法的不便之处。

```py
`const  App  =  ()  =>  { const  [  isAdminLoading,  setIsAdminLoading]  =  useState(false); const  [  isAdmin,  setIsAdmin]  =  useState(false); const  [  isAdminErr,  setIsAdminErr]  =  useState({  error:  false,  msg:  null  }); // there's a lot more overhead with this approach, more variables to deal with const  adminStatus  =  (loading,  success,  error)  =>  { // you must have your individual state on hand // it would be easier to pass your intent // and have your pre-concluded outcome initialized, typed with intent if  (error)  { setIsAdminLoading(false);  // could be set with intent in reducer setIsAdmin(false); setIsAdminErr({ error:  true, msg:  error.msg, }); throw  error; }  else  if  (loading  &&  !error  &&  !success)  { setIsAdminLoading(loading); }  else  if  (success  &&  !error  &&  !loading)  { setIsAdminLoading(false);  // .. these intents are convoluted setIsAdmin(true); } }; };` 
```

**useReducer 和 Array.prototype.reduce:** `useReducer`的行为与`Array.protoType.reduce`非常相似。它们都采用非常相似的参数；一个回调和一个初始值。在我们的例子中，我们称它们为`reducer`和`initialState`。它们都接受两个参数并返回一个值。最大的不同是`useReducer`随着派遣的回归增加了更多的功能。

**useReducer 和 Redux Reducer:** Redux 是一个复杂得多的应用状态管理工具。Redux reducers 通常通过调度、props、连接的类组件、动作和(可能的)服务来访问，并在 Redux 存储中维护。然而，实现`useReducer`将所有的复杂性抛在脑后。

> 我相信许多流行的 React 包会有令人难以置信的更新，这将使我们不带 Redux 的 React 开发更加有趣。对这些新实践的了解将有助于我们轻松过渡到即将到来的新奇更新。

### 用户的机会

Hooks API 简化了组件逻辑，去掉了关键字`class`。它可以在您的组件中根据需要进行多次初始化。困难的部分，很像上下文，是知道什么时候使用它。

**机遇**

您可能想看看在以下情况下实现`useReducer`:

*   组件需要复杂的状态对象
*   组件的属性将用于计算组件状态的下一个值
*   特定的`useState`更新方法取决于另一个`useState`值

在这些情况下应用`useReducer`将会减轻这种模式所带来的视觉和心理上的复杂性，并随着应用程序的增长提供持续的喘息空间。简而言之:你最小化了分散在组件中的依赖于状态的逻辑的数量，并增加了在更新状态时表达你的意图的能力，而不是表达结果的能力。

**优势**

*   组件和容器分离不需要
*   在我们的组件范围内，`dispatch`函数很容易*访问*
**   使用第三个参数、`initialAction`在初始渲染时操作状态的能力*

 *让我们开始编码吧！

### 实现 useReducer

让我们来看上面第一个上下文例子的一小部分。

```py
`class  App  extends  React.Component  { constructor(props)  { super(props); this.state  =  { isAuthenticated:  false, }; this.toggleAuth  =  this.toggleAuth.bind(this); } toggleAuth(success)  { success ?  this.setState({  isAuthenticated:  true  }) :  'Potential errorists threat. Alert level: Magenta?'; } render()  { // ... } }` 
```

这是上面用`useState`重构的例子。

```py
`const  App  =  ()  =>  { const  [isAuthenticated,  setAuth]  =  useState(false); const  toggleAuth  =  (success)  => success ?  setAuth(true) :  'Potential errorists threat. Alert level: Magenta?'; return  ( // ... ); };` 
```

令人惊讶的是这看起来是如此的漂亮和熟悉。现在，随着我们向这个应用程序添加更多的逻辑，`useReducer`将变得有用。让我们从`useState`开始附加逻辑。

```py
`const  App  =  ()  =>  { const  [isAuthenticated,  setAuth]  =  useState(false); const  [shakeForm,  setShakeForm]  =  useState(false); const  [categories,  setCategories]  =  useState(['Participants',  'Races']); const  [categoryView,  setCategoryView]  =  useState(null); const  toggleAuth  =  ()  =>  setAuth(true); const  toggleShakeForm  =  ()  => setTimeout(()  => setShakeForm(false),  500); const  handleShakeForm  =  ()  => setShakeForm(shakeState  => !shakeFormState ?  toggleShakeForm() :  null); return  ( // ... ); };` 
```

这要复杂得多。用`useReducer`会是什么样子？

```py
`// Here's our reducer and initialState // outside of the App component const  reducer  =  (state,  action)  =>  { switch  (action.type)  { case  'IS_AUTHENTICATED': return  { ...state, isAuthenticated:  true, }; // ... you can image the other cases default:  return  state; } }; const  initialState  =  { isAuthenticated:  false, shake:  false, categories:  ['Participants',  'Races'], };` 
```

我们将在带有`useReducer`的应用程序组件的顶层使用这些变量。

```py
`const  App  =  ()  =>  { const  [state,  dispatch]  =  useReducer(reducer,  initialState); const  toggleAuth  =  (success)  => success ?  dispatch({  type:  'IS_AUTHENTICATED'  })  // BAM! :  alert('Potential errorists threat. Alert level: Magenta?'); return  ( // ... ); };` 
```

这个代码很好地表达了我们的意图。这很好，但是我们需要将这个逻辑放到上下文中，这样我们就可以沿着树向下使用它。让我们看看我们的身份验证提供者，看看我们如何提供它的逻辑。

```py
`const  App  =  ()  =>  { // ... <AdminContext.Provider  value={{ isAuthenticated:  state.isAuthenticated, toggle:  toggleAuth, }}> <Main  /> </AdminContext.Provider> // ... };` 
```

有很多方法可以做到这一点，但是我们有一个状态和一个函数来分派传递给提供者的动作。除了逻辑的位置和组织不同之外，这与前面的很相似。大呼小叫...对吗？

如果您将`value={{ state, dispatch }}`传递给提供商的价值主张会怎样？如果我们将提供者提取到一个包装函数中会怎么样？您可以消除组件与其他组件紧密耦合的逻辑。进一步说，传递你的意图(行动)比传递你打算怎么做(逻辑)要好得多。

## 带挂钩的状态管理

这里我们有和以前一样的`App`、`reducer`和`initialState`，除了我们能够移除通过上下文传递的逻辑。相反，我们将关注组件中的逻辑，这将反过来执行我们的意图。

```py
`const  App  =  ()  =>  { const  [state,  dispatch]  =  useReducer(reducer,  initialState); // ... <AdminContext.Provider  value={{  state,  dispatch  }}> <Main  /> </AdminContext.Provider> // ... };` 
```

这是我们意图的必要逻辑，它应该在哪里。

```py
`const  AdminForm  =  ()  =>  { const  auth  =  useContext(AdminContext); const  handleSubmit  =  (event)  =>  { event.preventDefault(); authResult() ?  dispatch({  type:  'IS_AUTHENTICATED'  }) :  alert('Potential errorists threat. Alert level: Magenta?'); }; const  authResult  =  ()  => setTimeout(()  =>  true,  500); return  ( <div  className={'form-container'}> <form onSubmit={event  =>  handleSubmit(event)} className={'center-content login-form'} > // ... form stuff.. </form> </div > ); };` 
```

但是我们可以做得更好。用包装器提取提供者组件将为我们提供一种有效的方式来传递数据，而无需不必要的重新呈现。

```py
`// AdminContext.js const  reducer  =  (state,  action)  =>  { switch  (action.type)  { case  'IS_AUTHENTICATED': return  { ...state, isAuthenticated:  true, }; // ... you can image other cases default:  return  state; } }; const  initialState  =  { isAuthenticated:  false, // ... imagine so much more! }; const  ComponentContext  =  React.createContext(initialState); export  const  AdminProvider  =  (props)  =>  { const  [  state,  dispatch  ]  =  useReducer(reducer,  initialState); return  ( <ComponentContext.Provider  value={{  state,  dispatch  }}> {props.children} </ComponentContext.Provider> ); };` 
```

```py
`// App.js // ... const  App  =  ()  =>  { // ... <AdminProvider> <Main  /> </AdminProvider> // ... };` 
```

所以它稍微有点“复杂”，但是它组织得很好，可读性很强，也很容易理解。当您的应用程序增长并且需要更多上下文时，它也更容易管理。你可以按照组件或者你认为合适的方式来拆分你的上下文文件。这是您发挥和探索最适合您、您的团队和您的应用程序的时间！去找他！

## 结论

在我看来，你绝对应该考虑在你的应用程序中使用钩子进行状态管理。正如`useState`非常适合简单组件的状态管理一样，`useReducer`也能够处理一组组件。“应用程序的整体状态”会让位于“鸟巢的微观状态”吗？这种模式会像设计良好的微服务舰队一样创建更易于管理的微状态吗？

目前，hooks 还不能完全取代 Redux，但当你冒险进入大型生产应用程序的未知领域时，它们可以让你的旅程变得更容易。

实现`useReducer`相当简单，没有痛苦。从`useReducer`到 Redux 的重构*可能*比从头开始更容易。你手边会有大部分你需要的东西。因此，我建议在中小型应用程序上使用 hooks 进行状态管理，并期待看到新的方法，以更少的开销高效地实现健壮的、可管理的状态管理模式，如 Redux。谢谢反应，也谢谢阅读。

*[Git 回购](https://github.com/joha0033/race-series-hooks/tree/master/src)*

## 钩子 vs Redux:思想和观点

你是否在考虑通过像 Redux 这样更结构化的框架来实现状态管理的钩子？请注意以下利弊:

**优点**

1.  较少样板文件
2.  更快的开发(在一定程度上，对于中小型应用程序)
3.  主要的重构可以变得不那么复杂和痛苦
4.  更多的控制，但你必须考虑与大规模重新渲染相关的性能问题
5.  更好的是，如果你需要在这里和那里撒一些状态
6.  React 核心 API 的一部分

**缺点**

1.  较少的开发人员工具支持(如时间旅行调试)
2.  没有标准全局状态对象
3.  更少的资源和约定
4.  对于刚从 Redux 提供结构中获益的开发人员来说，这并不太好
5.  对于较大的应用程序，很难只提取和传递相关数据

对于那些刚开始在现有项目上工作的人来说，hooks 有可能不那么令人难以招架，术语也不那么多。因此，对于中小型应用程序，新开发人员可能会更好地使用 hooks 而不是 Redux。另一方面，对于较大的应用程序，如果有足够多的复杂性被抽象到已经建立的 Redux 商店中，那么就有一个很容易传达给任何开发人员的约定。

Redux 有明显的好处，主要是由于广泛的采用和成熟。在约定、资源(比如博客帖子和堆栈溢出问题等)、库和更好的开发工具领域。如果你知道你在做什么，用 Redux 调试一个讨厌的复杂状态对象可能会简单一些。换句话说，仅仅因为一个开发人员可以看到一个问题，并不一定意味着该开发人员知道如何跟踪并修复它，或者甚至可以访问问题逻辑。此外，对于大型应用程序，商店可能会变得不堪重负。这就像在全食超市挑选橄榄油一样——要吸收的东西太多了。钩子允许我们将数据/归约器/逻辑划分到特定的嵌套*，这些嵌套可以在任何地方共享*。Redux 的性能*应该*更好，尤其是对于较大的应用。这并不意味着这是一个保证。正确实现 Redux，勤于基准测试，关注状态结构，这些都取决于开发人员。使用钩子，在 Redux 之前，您可能会遇到性能问题，但是从我使用钩子的有限时间来看，我觉得使用上面提到的一些技巧可以更容易地跟踪和修复这些问题。这也是一个实现模仿`mapStateToProps`和`mapDispatchToProps`的定制逻辑的机会，这将随着应用的增长而提高性能。这取决于你进行实验，并自己决定什么最适合当前的情况。记住:这不是一个“非此即彼”的情况——你可以同时使用 hooks 和 Redux 来管理你的应用程序的状态。*
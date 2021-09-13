# 在react 18 中通过批量合并`setState()`减少渲染

## 概述

---

React 18 通过默认执行更多批量状态更新来作为开箱即用的性能改进，无需在应用程序或库代码中手动批量状态更新。
这篇文章将解释什么是批量状态更新，它以前是如何工作的，以及发生了什么变化。

> 注意：这是一个我们不希望大多数用户需要考虑的深入功能。但是，它可能与教育工作者和图书馆开发人员有关。

## 什么是批量状态更新？

批量状态更新是 **React将多个状态更新分配到单个重新渲染中以获得更好的性能**。

例如，如果你在同一个点击事件中有两个状态更新，React 总是将它们分批状态更新到一个重新渲染中。如果你运行下面的代码，你会看到每次点击时，React 只执行一次渲染，尽管你设置了两次状态：

```jsx
function App() {
  const [count, setCount] = useState(0);
  const [flag, setFlag] = useState(false);

  function handleClick() {
    setCount(c => c + 1); // 没有渲染
    setFlag(f => !f); // 没有渲染
    // React 将会在最后更新结束后执行一次渲染（批量状态更新）
  }

  return (
    <div>
      <button onClick={handleClick}>Next</button>
      <h1 style={{ color: flag ? "blue" : "black" }}>{count}</h1>
    </div>
  );
}

```

- ✅ [codeSandBox（请注意控制台中的每次点击渲染一次。）](https://codesandbox.io/s/spring-water-929i6?file=/src/index.js)


这对性能提升非常有用，因为它避免了不必要的重新渲染。
它还可以防止您的组件呈现仅更新一个状态变量的“半完成”状态，这可能会导致产生某些bug。
这可能会让您想起餐厅服务员在您选择第一道菜时不会跑到厨房，而是等待您完成点餐后去厨房报菜名。

然而，React 批量更新的时间并不一致。
例如，如果您需要获取数据，然后更新handleClick上面的状态，那么 React不会批量更新，而是执行两次独立的更新。

这是因为 React 过去只在浏览器事件（如点击）期间批量更新，但这里我们在事件已经被处理（在 `fetch` 回调中）之后更新状态：

```jsx
function App() {
  const [count, setCount] = useState(0);
  const [flag, setFlag] = useState(false);

  function handleClick() {
    fetchSomething().then(() => {
      // React 17 及更早版本不会对这些进行批量更新，因为
      // 它们在回调中 *after* 阶段运行，而不是 *during* 阶段
      setCount(c => c + 1); // 产生一次重新渲染
      setFlag(f => !f); // 产生一次重新渲染
    });
  }

  return (
    <div>
      <button onClick={handleClick}>Next</button>
      <h1 style={{ color: flag ? "blue" : "black" }}>{count}</h1>
    </div>
  );
}

```

- [🟡 codeSandBox（请注意控制台中每次点击两次渲染。）](https://codesandbox.io/s/trusting-khayyam-cn5ct?file=/src/index.js)

在 React 18 之前，我们只在 React 事件处理程序期间批量更新。
默认情况下，React 中不会对 `promise`、`setTimeout`、`原生事件处理程序`或任何其他事件中的更新进行批量状态更新。

## 什么是自动批量状态更新？

从 React 18 开始，通过使用[createRoot](./create-root.md)，所有状态更新都将使用自动批量状态更新，无论它们来自何处。

这意味着`promise`、`setTimeout`、`原生事件处理程序`或任何其他事件内的更新将以与 `React 事件`内的更新相同的方式进行批量状态更新。
我们希望这会导致更少的渲染工作，从而在您的应用程序中获得更好的性能：

```jsx
function App() {
  const [count, setCount] = useState(0);
  const [flag, setFlag] = useState(false);

  function handleClick() {
    fetchSomething().then(() => {
       // React 18 及更高版本确实批量状态更新这些：
      setCount(c => c + 1);
      setFlag(f => !f);
      // React 只会在最后重新渲染一次（即批量状态更新！) 
    });
  }

  return (
    <div>
      <button onClick={handleClick}>Next</button>
      <h1 style={{ color: flag ? "blue" : "black" }}>{count}</h1>
    </div>
  );
}

```

- ✅ [codeSandBox（注意控制台中的每次点击渲染一次！）](https://codesandbox.io/s/morning-sun-lgz88?file=/src/index.js)
🟡 演示：React 18 with legacyrender保留了旧的行为（注意控制台中每次点击两次渲染。）
注意：作为采用 React 18 的一部分，预计您将升级到createRoot。 旧行为 的render存在只是为了更容易地对两个版本进行生产实验。

无论更新发生在何处，React 都会自动批量更新，因此：

函数 handleClick ( )  { 
  setCount ( c  =>  c  +  1 ) ; 
  setFlag ( f  =>  ! f ) ; 
  // React 只会在最后重新渲染一次（这是批量状态更新！）
}
行为与此相同：

setTimeout ( ( )  =>  { 
  setCount ( c  =>  c  +  1 ) ; 
  setFlag ( f  =>  ! f ) ; 
  // React 只会在最后重新渲染一次（这是批量状态更新！）
} ,  1000 ) ;
行为与此相同：

获取( /*...*/ ) 。then ( ( )  =>  { 
  setCount ( c  =>  c  +  1 ) ; 
  setFlag ( f  =>  ! f ) ; 
  // React 只会在最后重新渲染一次（这是批量状态更新！）
} ）
行为与此相同：

榆树。addEventListener ( 'click' ,  ( )  =>  { 
  setCount ( c  =>  c  +  1 ) ; 
  setFlag ( f  =>  ! f ) ; 
  // React 只会在最后重新渲染一次（这是批量状态更新！）
} ) ;
注意：React 仅在通常安全的情况下才批量更新。例如，React 确保对于每个用户启动的事件（如单击或按键），DOM 在下一个事件之前完全更新。例如，这可确保在提交时禁用的表单不​​能被提交两次。

如果我不想批量状态更新怎么办？
通常，批量状态更新是安全的，但某些代码可能依赖于在状态更改后立即从 DOM 中读取某些内容。对于这些用例，您可以使用ReactDOM.flushSync()选择退出批量状态更新：

 从'react-dom'导入{  flushSync  }  ；// 注意：react-dom，不是react  

function  handleClick ( )  { 
  flushSync ( ( )  =>  { 
    setCounter ( c  =>  c  +  1 ) ; 
  } ) ; 
  // React 现在已经更新了 DOM 
  flushSync ( ( )  =>  { 
    setFlag ( f  =>  ! f ) ; 
  } ) ; 
  // React 现在已经更新了 DOM 
}
我们不希望这很常见。

这对 Hooks 有什么影响吗？
如果您使用 Hooks，我们希望自动批量状态更新在绝大多数情况下都能“正常工作”。（如果没有，请告诉我们！）

这对 Classes 有什么影响吗？
请记住，React 事件处理程序期间的更新始终是批量状态更新的，因此对于这些更新，没有任何更改。

在类组件中存在边缘情况，这可能是一个问题。

类组件有一个实现的怪癖，它可以同步读取事件内部的状态更新。这意味着您将能够this.state在调用之间阅读setState：

handleClick  =  （） =>  {
  的setTimeout （（） =>  {
    此。的setState （（{计数} ） =>  （{ 计数：计数 +  1  } ））;

    // { count: 1, flag: false } 
    console . 日志（这个。状态）；

    这个。setState ( ( { flag } )  =>  ( {  flag : ! flag  } ) ) ; 
  } ) ; 
} ;
在 React 18 中，情况不再如此。由于所有更新setTimeout都是批量状态更新的，因此 React 不会setState同步渲染第一个的结果——渲染发生在下一个浏览器滴答。所以渲染还没有发生：

handleClick  =  （） =>  {
  的setTimeout （（） =>  {
    此。的setState （（{计数} ） =>  （{ 计数：计数 +  1  } ））;

    // { count: 0, flag: false }
    控制台. 日志（这个。状态）；

    这个。setState ( ( { flag } )  =>  ( {  flag : ! flag  } ) ) ; 
  } ) ; 
} ;
请参阅沙箱。

如果这是升级到 React 18 的障碍，您可以使用ReactDOM.flushSync强制更新，但我们建议谨慎使用：

handleClick  =  （） =>  {
  的setTimeout （（） =>  { 
    ReactDOM 。flushSync （（） =>  {
      此。的setState （（{计数} ） =>  （{ 计数：计数 +  1  } ））; 
    } ）;

    // { count: 1, flag: false } 
    console . 日志（这个。状态）；

    这个。setState ( ( { flag } )  =>  ( {  flag : ! flag  } ) ) ; 
  } ) ; 
} ;
请参阅沙箱。

此问题不会影响带有 Hooks 的函数组件，因为设置状态不会从useState以下位置更新现有变量：

功能 handleClick （） {
  的setTimeout （（） =>  {
    控制台。登录（计数）;  // 0 
    setCount （ç  =>  C ^  +  1 ）; 
    setCount （ç  =>  C ^  +  1 ）; 
    setCount （ç  =>  C ^  +  1 ）;
    控制台。日志（计数）;  // 0 
  }, 1000)
虽然当您采用 Hooks 时这种行为可能令人惊讶，但它为自动批量状态更新铺平了道路。

怎么样unstable_batchedUpdates？
一些 React 库使用这个未记录的 API 来强制对setState外部事件处理程序进行批量状态更新：

 从'react-dom'导入{  unstable_batchedUpdates  }  ； 

unstable_batchedUpdates ( ( )  =>  { 
  setCount ( c  =>  c  +  1 ) ; 
  setFlag ( f  =>  ! f ) ; 
} ) ;
这个 API 在 18 中仍然存在，但不再需要它了，因为批量状态更新是自动发生的。我们不会在 18 中删除它，尽管在流行的库不再依赖于它的存在之后，它可能会在未来的主要版本中被删除。

回复
10 条评论
·
31 回复
@chantastic
美妙的
on 28 May
合作者
这是一篇非常清晰的文章，@gaearon。我喜欢沙盒示例。我 - 诚然 - 不知道不同的行为，但觉得我可以在这个描述之后解释它。

这篇文章的次要作用是将rendervs澄清createRoot为“作为采用 React 18 的一部分”。这是一个非常好的“升级胡萝卜”⬆️🥕. 你给人们提供了一些东西来确定成功。全胜。


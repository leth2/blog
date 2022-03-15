# React 18 소개

## Batch? Automatic Batch!

```jsx
  const [count, setCount] = useState(0);
  const [flag, setFlag] = useState(false);

  function handleClick() {
    setCount(count + 1); // 렌더링이 수행되지 않습니다.
    setFlag(!flag); // 렌더링이 수행되지 않습니다.
    // 렌더링이 한꺼번에 수행됩니다.
  }
```

배치는 하나의 함수에서 두 건 이상의 상태의 업데이트를 수행할 경우, 각각의 상태를 업데이트할 때마다 렌더링을 수행하는 것이 아닌, 모든 상태편경이 완료된 시점에서 렌더링을 모아서 한번에 실행하는 작업을 말합니다.

17버전까지는 브라우저의 이벤트를 처리하는 동안에만 배치가 작동했습니다. 

```jsx
const [count, setCount] = useState(0);
const [flag, setFlag] = useState(false);

function handleClick() {
  fetchSomething().then(() => {
		// 이벤트가 수행되는 동안이 아닌 이벤트 후 callback 구간에서 상태가 업데이트 될 때 
    // React17 이하의 환경에서는 배치가 작동하지 않습니다.
    setCount(count + 1); // 렌더링이 수행됩니다.
    setFlag(!flag); // 렌더링이 수행됩니다.
  });
}
```

 

이제는 이벤트 도중뿐만아니라 timeout, promise, 그외의 대부분 이벤트 처리에서 배치가 작동합니다. (createRoot를 이용해야합니다.)

```jsx
function handleClick() {
  setCount(count + 1);
  setFlag(!flag);
  // 배치가 작동하므로 한 번의 렌더링만 수행됩니다.
}
```

```jsx
setTimeout(() => {
  setCount(count + 1);
  setFlag(!flag);
  // 배치가 작동하므로 한 번의 렌더링만 수행됩니다.
}, 1000);
```

```jsx
fetch(/*...*/).then(() => {
  setCount(count + 1);
  setFlag(!flag);
  // 배치가 작동하므로 한 번의 렌더링만 수행됩니다.
})
```

```jsx
elm.addEventListener('click', () => {
  setCount(count + 1);
  setFlag(!flag);
  // 배치가 작동하므로 한 번의 렌더링만 수행됩니다.
});
```

만일 Automatic Batch를 사용하고 싶지 않으면 ReactDOM.flushSync() 를 사용하세요. 꼭 필요할 때만 사용하세요.

```jsx
import {flushSync} from 'react-dom'

function handleClick() {
  flushSync(()=>{setCount(count + 1)}) // 배치를 수행하지 않고 바로 렌더링됩니다.
  flushSync(()=>{setFlag(!flag)}) // 배치를 수행하지 않고 바로 렌더링됩니다.
}
```

## 데이터를 가져오기 위한 Suspense ?

### 다시한번 CSR vs SSR

CSR의 화면 렌더링 단계 (**TTV === TTI)**

1. 사용자 요청 및 서버응답 with HTML with JS link
2. 자바스크립트 다운로드  (**view 불가능, interaction 불가능**)
3. 자바스크립트 실행, React 실행(수화)
4. 화면 렌더링 완료**(view 및 interaction 가능)** 

CSR의 리액트는 아래와 같은 html구조에 Javascript를 실행하면서 화면을 구성하게 됩니다.

때문에 **hydrate가 완료되기 전까지 사용자는 화면을 볼 수도 없고 상호작용도 할 수  없습니다.**

```jsx
<html>
	<body>
		<div id="app"></div>
	</body>
</html>
```

SSR의 화면 렌더링 단계 (**TTV !== TTI)**

1. 사용자 요청 및 서버응답 with pre-renderd HTML with JS Link
2. html 렌더링 완료, 자바스크립트 다운로드 (**view 가능, interaction 불가능**)
3. 자바스크립트 실행, **React 실행(hydration-수화)**
4. React 적용완료 (**view 가능, interaction 불가능**)

SSR의 리액트는 아래와 같이 Pre-render된 html을 서버로부터 직접 수신합니다.

때문에 **hydrate가 완료되기 전까지 사용자는 화면은 볼 수 있지만 상호작용은 하지 못합니다.**

```jsx
<html>
	<body>
		<div id="app">
			<h1>Hello world</h1>
			<span>SRR은 서버에서 렌더링된 HTML을 다운로드 받습니다.</span>
		</div>
	</body>
</html>
```

### React 18이전의 Suspense

React.lazy()와 함께쓰는 지금의 Suspense 는 code-splitting을 위한 방법중 하나로 소개 되어있습니다. 그리고 SSR을 지원하지 않습니다.

```jsx
import React, { Suspense } from 'react';

const OtherComponent = React.lazy(() => import('./OtherComponent'));

function MyComponent() {
  return (
    <div>
      <Suspense fallback={<div>Loading...</div>}>
        <OtherComponent />
      </Suspense>
    </div>
  );
}
```

하지만 Router에 React.lazy()를 적용하는 것이 code-splitting을 위한 방법으로서는 조금 더 직관적인 방법이었습니다.

### React 18의 Suspense, Suspense for Data Fetching

Suspense는 새로운 SSR 아키텍쳐를 지원하는 방법이 추가됩니다. 기존의 Suspense가 컴포넌트만 지원했다면 이제는 어떤 것이든 지원할 수 있는 기능이 됩니다.

### Streaming HTML 과 Selective Hybration

```jsx
<Layout>
  <NavBar />
  <Sidebar />
  <RightPane>
    <Post />
    <Suspense fallback={<Spinner />}>
      <Comments />
    </Suspense>
  </RightPane>
</Layout>
```

기존의 SSR 방식은 서버에서 필요한 HTML을 모두 렌더링한 다음 응답하는 것이라면 React18에서는 Suspense를 이용해서 HTML을 모두 렌더링하지 않고 HTML을 작은 단위로 분할한 다음 렌더링되는대로 사용자에게 전달하게 됩니다. 이를 Steaning HTML이라고 합니다.

즉 SSR 환경에서 HTML이 조각조각 렌더링될 수 있게 되고 사용자가 렌더링된 첫 화면을 받아보는 시간을 보다 더 앞당길 수 있습니다. TTV가 화면의 조각, 즉 컴포넌트별로 달라지게 됩니다.

하지만 Hydration이 완료되지 않으면 사용자는 단지 화면을 일찍 받아볼 뿐 화면과의 상호작용은 마지막 컴포넌트가 화면에 렌더링될 때까지 지연되게 됩니다. 이 때문에 **Selective Hydrating**이 도입됩니다.

Suspense를 사용하게 되면 hydration도 개별적으로 수행하게 됩니다. TTI도 화면이 아니라 개별컴포넌트별로 가져가게 됩니다.

[New Suspense SSR Architecture in React 18 · Discussion #37 · reactwg/react-18](https://github.com/reactwg/react-18/discussions/37)

### 바로 사용할 수 있나?

Suspense를 지원하는 Data Fetching 라이브러리가 필요합니다. 

현재는 아래의 라이브러리 정도가 React18에서 도입될 Suspense를 지원하고 있습니다.

- Relay
- React Query
- SWR
- Recoil
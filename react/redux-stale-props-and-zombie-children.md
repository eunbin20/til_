# Redux에서 Zombie Children이란 무엇일까?

이 문서는 https://kaihao.dev/posts/Stale-props-and-zombie-children-in-Redux

위 문서를 의역한 문서이다.

개인적으로 직역은 가독성이 무척 낮다고 생각하는 편이라,영어 원문에 대한 나의 개인적인 생각이 마니마니 담겨있는 의역본임을 미리 밝혀둔다.

react-redux의 v7 release 문서를 보면, “stale props and ‘zombie children’”이라는 섹션을 만날 수 있을 것 이다. 이 포스트에서는 이 문제와 react-redux가 이문제를 어떻게 해결했는지 깊이있게 알아본다.

Redux와 react-redux의 주요한 기능을 다시 구현해봄으로써 이 문제를 파헤쳐보자. 어디까지나 설명을 위한 구현이므로 Redux의 모든 기능들을 다루지는 않겠지만, 이것만으로도 우리가 해결하려는 문제를 이해하는 데는 충분할 것이다.

리덕스는 flux 패턴으로 구성되는 전역 레벨의 구독 모델이다. 자바스크립트의 구독 모델은 종종 끌어올려지는 이벤트 리스터들을 통해 만들어진다. 우리는 변화를 “구독”하고 “리듀서”를 통해 상태를 변화시킨다. 그리고 화면 업데이트를 위해 그 결과를 모든 리스너들에게 전달한다.

```tsx
const createStore = (reducer, initialState = {}) => {
  let state = initialState;
  const listeners = [];

  return {
    getState() {
      return state;
    },
    subscibe(listener) {
      listeners.push(listener);

      // Returns an unsubscribe function
      return () => {
        const index = listeners.indexOf(listener);
        listeners.splice(index, 1);
      };
    },
    dispatch(action) {
      state = reducer(state, acction);

      listeners.forEach((listener) => {
        listener();
      });
    },
  };
};
```

상단의 코드는 리덕스의 cretaeStore를 아주 단순하게 구현한 예제이다. 이 store에서 getState()를 이용해 상태를 가져올 수 있고, subscribe()를 이용해 리스너 함수를 구독할 수 있다. 또한 dispatch()를 이용해 action을 내보낼(dispatch)수 있다.

다음 단계는 이것을 리액트를 이용해서 어떻게 구현해는가를 살펴보는 것이다. 우리는 컨텍스트를 통해 store를 내려보내기 위한 <Provider> 컴포넌트, presentation 컴포넌트를 감싸기위한 connect HOC를 만들것이고, 가장 최근에는 connect를 대체하기위한 useSelector hook을 만들 것이다.

### Problem

이제 우리가 해결하고자하는 문제의 정의를 간단히 요약해보자.

- Stale props는 다음과 같은 어떤 케이스이다.
  - selector 함수는 데이터를 추출하기 위해 현재 컴포넌트의 props에 의존한다.
  - 액션이 발생하면, 부모 컴포넌트는 리렌더링될 것이고, 새로운 props를 내려보낼 것이다.
  - 하지만 현재 컴포넌트의 selector함수는 현재 컴포넌트가 새로운 props와 함께 리렌더링되기 전에 실행된다.
- Zombie child는 다음과 같은 경우를 말한다.
  - 복잡하게 연결된 컴포넌트들은 초기에 마운트되고, 이는 자식컴포넌트가 부모 컴포넌트보다 먼저 store를 구독하게한다.
  - store에서 todo item과 같은 어떤 데이터를 제거하는 액션이 발생하면,
  - 부모 컴포넌트는 그 제거된 자식 컴포넌트 렌더링을 중단할 것이다.
  - 그러나, 자식 컴포넌트 부모보다 먼저 구독했기 때문에, 부모 컴포넌트가 이 자식 컴포넌트의 렌더링을 중단하기 직전에 **구독이** 실행된다. 구독이 실행되면 props를 기반으로 store에 있는 값을 읽을 것이고, 해당 데이터는 더이상 없는 데이터이고, 만약 데이터를 가져오는 로직이 튼튼하지 않다면, 에러를 발생시킬 것이다.
    예를 들어,,
    우리가 만들어볼 것은 아주 간단한 todo 어플리케이션이다. 할일 목록을 렌더링하고, DELETE 액션을 이용해 개별 아이템을 제거할 수 있다.
    먼저, store와 리듀서를 만들었다.
  ```tsx
  const reducer = (state, action);
  ```

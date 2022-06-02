---
title: "[Debate-Ducks] Debateroom - Record"
date: 2022-06-01
categories:
  - <Projects>
tags:
  - "'Debate-Ducks'"
  - (Devlog)
---

## Summary

추후 작성 예정

## Details

## Problems

### 변수가 할당되기 전에 사용됨

조건부로 변수에 값을 할당하고 해당 변수를 다음 변수에 활용했다. 이때 변수가 할당되기 전에 사용된다는 에러가 떴다. 해당 변수의 타입에 undefined를 포함하면 해결되는 간단한 문제였다.

```bash
Variable 'mergedStream' is used before being assigned.
```

### 최초 연결 시 마이크 꺼짐

마이크는 켜진 상태로 연결되게 했었는데 최초 연결 시 꺼져있었다. 이유는 간단했다. 단계 전환 시 자신의 차례일 때는 마이크를 켜고 아닐 때는 꺼지게 해뒀는데 최초의 단계는 찬반 양측 모두에게 자신의 턴이 아니라 마이크가 꺼진 것이었다. 그래서 최초의 차례일 때를 조건에 추가해 해결했다.

```ts
// Good!!
useEffect(() => {
  if (turn === "none") {
  } else if (isPros) {
    if (turn === "pros" || turn === "prosCross") {
      toggleMic(stream, true, setIsMicOn);
    } else {
      toggleMic(stream, false, setIsMicOn);
    }
  } else {
    if (turn === "cons" || turn === "consCross") {
      toggleMic(stream, true, setIsMicOn);
    } else {
      toggleMic(stream, false, setIsMicOn);
    }
  }
  offScreen(peerRef, stream, videoRef, screenStreamRef, setIsScreenOn);
}, [stream, turn, isPros]);
```

### 재연결 시 비디오 및 화면 공유 오작동

우선 재연결 시 상대방 비디오의 현재 상태와 상관없이 `useRef(false)`로 인해 꺼져있는 상태로 인식하는 문제가 있었다. 이 문제는 비디오의 상태를 보내는 `useEffect`의 dependency에 peerStream을 추가하는 것으로 쉽게 해결했다.

화면 공유의 경우는 재연결 시 상대방 화면 공유가 켜져 있어도 인식하지 못하는 문제와 이 상태에서 화면 공유를 끌 경우 `Error: Cannot replace track that was never added.` 에러가 발생하는 문제가 있었다.

<img width="800" alt="reconnect-screen_share-error-1" src="https://user-images.githubusercontent.com/84524514/171619486-b947bdfd-3053-4965-a66a-a7fb2ac142da.png">

그래서 상대방 재연결 시 화면 공유를 꺼주는 방식으로 해결해야겠다고 생각했다. 처음에는 `offScreen` 함수를 peerStream이 변경될 때 적용하는 방식을 사용했지만 화면 종료 시 발생하는 에러가 여전히 발생했다.

```ts
// Bad1!!
useEffect(() => {
  offScreen(peerRef, stream, videoRef, screenStreamRef, setIsScreenOn);
}, [stream, peerStream]);
```

이번에는 자신의 화면을 끄는 부분만 peerStream이 변경될 때 적용했다. 그러자 상대방의 마지막 공유 화면이 상대방의 비디오를 대체하는 문제와 자신의 비디오를 표시하지 못하는 문제가 동시에 발생했다.

```ts
// Bad2!!
useEffect(() => {
  if (stream && screenStreamRef.current && videoRef.current) {
    screenStreamRef.current.getTracks()[0].stop();
    videoRef.current.srcObject = stream;
    setIsScreenOn(false);
    screenStreamRef.current = undefined;
  }
}, [stream, peerStream]);
```

<img width="800" alt="reconnect-screen_share-error-2" src="https://user-images.githubusercontent.com/84524514/171620807-06efbe51-54c5-4373-895b-9f5ff1005a30.png">

이유는 화면 공유 시 `.onended`에 `peerRef.current?.replaceTrack()`이 등록되는 것이었다. 처음 화면 공유를 켤때는 상대방이 없어서 `replaceTrack`이 작동하지 않지만 `.onended`의 `replaceTrack`이 작동하는 시점에는 상대방이 있어서 문제가 생겼다.

그래서 화면 공유 함수 자체에서 상대방이 있을 때와 없을 때로 조건을 나누어 문제를 해결했다.

```ts
// Good!!
export const screenShare = async (
  ...
) => {
  try {
    const screenStream = await navigator.mediaDevices.getDisplayMedia({
      video: true,
      audio: false,
    });
    // * Peer가 있을 때
    if (peerRef.current) {
      ...
    // * Peer가 없을 때
    } else {
      ...
    }
  } catch (err) {
    console.log(err);
  }
};
```

## Etc

### useState와 useRef

저장해둔 값이 변경될 때 `useState`는 재랜더링을 일으키고 `useRef`는 재랜더링을 일으키지 않는다. 나는 이 프로젝트를 진행할 때 기본적으로는 재랜더링을 일으키지 않는 `useRef`를 사용했고 재랜더링이 필요할 경우 `useState`를 사용했다.

### 작성된 코드에는 이유가 필요하다

좋은 코드를 작성하기 위한 하나의 노력으로 코드를 한 줄 한 줄 복기하면서 "왜 이 코드가 필요한가?"를 스스로 생각해 보는 시간을 가졌다. 그러자 중간에 기능을 수정하면서 필요 없어진 부분들이 하나 둘 발견되었다. 또한 쓸데없이 복잡하게 작성된 부분들도 있었다. 이를 수정하면서 코드에 대한 이해가 한층 깊어짐을 느꼈다. 앞으로도 주기적으로 내가 작성한 코드를 복기하는 데 시간을 투자할 것이다.

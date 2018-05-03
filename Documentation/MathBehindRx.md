Math Behind Rx
==============

## 옵저버와 이터레이터 / 이너머레이터 / 제네레이터 / 시퀀스 간의 이중성

옵저버와 제네레이터 패턴 사이에는 이중성이 있습니다. 이것은 우리가 비동기 콜백 세계에서 동기 세계의 시퀀스 변형을 가능하게 해줍니다.

짧게 말해서, 이너머레이터와 옵저버 패턴은 둘 다 시퀀스를 설명합니다. 이너머레이터가 시퀀스를 정의하는 것은 상당히 분명하지만, 옵저버의 경우에는 조금 복잡합니다.

그렇지만 여기 수학적 지식이 많이 필요하지 않는 쉬운 예제가 있습니다. 여러분이 주어진 시간에 화면의 마우스 커서의 위치를 관찰하고 있다고 가정해봅시다. 시간이 지나면, 이 마우스 위치들은 시퀀스를 이룹니다. 이것들은 본질적으로 옵저버블 시퀀스입니다.

시퀀스의 요소가 접근될 수 있는 방법은 기본적으로 두 가지 입니다:

* 푸시 인터페이스 - 옵저버 (요소가 시퀀스를 이룰동안 계속 관찰합니다)
* 풀 인터페이스 - 이터레이터 / 이너머레이터 / 제네레이터

또한 더 많은 공식적인 설명을 비디오로도 보실 수 있습니다:

* [Expert to Expert: Brian Beckman and Erik Meijer - Inside the .NET Reactive Framework (Rx) (video)](https://www.youtube.com/watch?v=looJcaeboBY)
* [Reactive Programming Overview (Jafar Husain from Netflix)](https://www.youtube.com/watch?v=dwP1TNXE6fc)

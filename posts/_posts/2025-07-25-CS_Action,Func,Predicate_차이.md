---
layout: post
title: "C# - Action, Func, Predicate 차이"
description: >
  C#에서 자주 사용하는 delegate인 Action, Func, Predicate에 대해 자세히 알아봅니다.
sitemap: true
hide_last_modified: true
---

C#에서 람다식이나 이벤트, 콜백을 다루다 보면 `Action`, `Func`, `Predicate` 같은 델리게이트 타입이 자주 등장합니다. 이들은 메서드를 변수처럼 다룰 수 있게 해 주고, 코드를 더 간결하고 유연하게 만듭니다.

## Action : 반환값이 없는 메서드

```cs
Action<string> greet = name => Console.WriteLine($"Hello, {name}!");
greet("Jeongwon");
```

`Action`은 반환값이 없는(delegate void) 메서드를 표현할 때 사용합니다. 단순히 어떤 동작을 수행하고 결과를 반환할 필요가 없을 때 유용합니다.

- 최대 16개의 입력 매개변수까지 받을 수 있음
- void를 반환함
- 제네릭 인자가 입력값

### Action의 다양한 형태

```cs
Action hello = () => Console.WriteLine("Hello!");
hello();  // 출력: Hello!
```

`Action`은 입력값이 없는 형태로도 사용할 수 있습니다. 이때는 제네릭을 사용하지 않습니다.

## Func : 반환값이 있는 메서드

```cs
Func<int, int, int> add = (a, b) => a + b;
int result = add(3, 4); // 7
```

`Func`은 입력값을 받아서 결과를 반환하는 함수를 표현할 때 사용합니다. 마지막 제네릭 인자가 반환값의 타입입니다.

- 마지막 제네릭 인자: 반환값 타입
- 앞의 제네릭 인자들: 입력값
- 최대 16개의 인자를 받을 수 있음

## Predicate : bool을 반환하는 메서드

```cs
Predicate<int> isEven = n => n % 2 == 0;
bool result = isEven(4); // true
```

`Predicate`는 특정 조건을 만족하는지를 검사할 때 사용합니다. 항상 bool을 반환하며, 입력값은 1개입니다.

- 입력값 1개
- 반환값은 항상 bool
- 사실상 Func<T, bool>과 동일하지만, 조건 검사를 명확하게 표현하는 용도

## 사용 예시

```cs
List<int> numbers = new() { 1, 2, 3, 4, 5 };

var evenNumbers = numbers.FindAll(n => n % 2 == 0); // Predicate 사용
var squared = numbers.Select(n => n * n); // Func 사용
numbers.ForEach(n => Console.WriteLine(n)); // Action 사용
```

## 마무리

`Action`, `Func`, `Predicate`는 델리게이트를 보다 간단하고 명확하게 표현할 수 있는 도구입니다. 처음에는 다소 헷갈릴 수 있지만, 몇 번 사용해 보면 금방 익숙해지고 보다 **유연하고 재사용 가능한 코드**를 작성하는 데 큰 도움이 됩니다.

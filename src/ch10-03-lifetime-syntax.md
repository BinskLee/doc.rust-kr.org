## 라이프타임으로 참조자의 유효성 검증하기

라이프타임 (lifetime) 은 이미 사용해 본 적 있는 또 다른 종류의 제네릭입니다.
라이프타임은 어떤 타입이 원하는 동작이 구현되어 있음을 보장하기 위한 것이 아니라,
어떤 참조자가 필요한 기간 동안 유효함을 보장하도록 합니다.

4장 [‘참조와 대여’][references-and-borrowing]<!-- ignore -->절에서
다루지 않은 내용이 있습니다. 러스트의 모든 참조자는 *라이프타임*이라는
참조자의 유효성을 보장하는 범위를 갖습니다. 대부분의 상황에서
타입이 암묵적으로 추론되듯, 라이프타임도 암묵적으로 추론됩니다.
하지만 여러 타입이 될 가능성이 있는 상황에서는 타입을 명시해 주어야 하듯,
참조자의 수명이 여러 방식으로 서로 연관될 수 있는 경우에는
라이프타임을 명시해 주어야 합니다. 러스트에서 런타임에 사용되는
실제 참조자가 반드시 유효할 것임을 보장하려면 제네릭 라이프타임
매개변수로 이 관계를 명시해야 합니다.

라이프타임을 명시하는 것은 다른 프로그래밍 언어에서는 찾아보기 어려운
개념이며, 따라서 친숙하지 않은 느낌이 들 것입니다. 이번 장에서 라이프타임의
모든 것을 다루지는 않겠지만, 라이프타임이라는 개념에 익숙해질 수 있도록 여러분이
접하게 될 일반적인 방식의 라이프타임 문법만 다루겠습니다.

### 라이프타임으로 댕글링 참조 방지하기

라이프타임의 주목적은 *댕글링 참조 (dangling reference)* 방지입니다.
댕글링 참조는 프로그램이 참조하려고 한 데이터가 아닌
엉뚱한 데이터를 참조하게 되는 원인입니다. 예제 10-16처럼
안쪽 스코프와 바깥쪽 스코프를 갖는 프로그램을 생각해 봅시다:

```rust,ignore,does_not_compile
{{#rustdoc_include ../listings/ch10-generic-types-traits-and-lifetimes/listing-10-16/src/main.rs}}
```

<span class="caption">예제 10-16: 스코프 밖으로 벗어난 값을
참조하는 코드</span>

> Note: 예제 10-16, 10-17, 10-23 예제는 변수를 초깃값 없이 선언하여,
> 스코프 밖에 변수명을 위치시킵니다.
> 널 값을 갖지 않는 러스트가 이런 형태의 코드를 허용하는 게 이상하다고 생각하실 수도 있지만,
> 만약 값을 넣기 전에 변수를 사용하는 코드를 실제로 작성할 경우에는
> 러스트가 컴파일 에러를 발생시킵니다.
> 널 값이 허용되는 것은 아닙니다.

바깥쪽 스코프에서는 `r` 변수를 초깃값 없이 선언하고 안쪽
스코프에서는 `x` 변수를 초깃값 5로 선언합니다. 안쪽 스코프에서는
`r` 값에 `x` 참조자를 대입합니다. 안쪽 스코프가 끝나면 `r` 값을
출력합니다. 이 코드는 컴파일되지 않습니다. `r`이 참조하는 값이
사용하려는 시점에 이미 자신의 스코프를 벗어났기 때문입니다.
에러 메시지는 다음과 같습니다:

```console
{{#include ../listings/ch10-generic-types-traits-and-lifetimes/listing-10-16/output.txt}}
```

변수 `x`가 ‘충분히 오래 살지 못했습니다 (does not live long enough).’
`x`는 안쪽 스코프가 끝나는 7번째 줄에서 스코프를 벗어나지만 `r`은
바깥쪽 스코프에서 유효하기 때문입니다. 스코프가 더 클수록
‘더 오래 산다 (lives longer)’고 표현합니다. 만약 러스트가 이 코드의 작동을 허용하면
`r`은 `x`가 스코프를 벗어날 때 할당 해제된 메모리를 참조할 테고,
`r`을 이용하는 모든 작업은 제대로 작동하지 않을 것입니다.
그렇다면 러스트는 이 코드가 유효한지를 어떻게 검사할까요? 정답은 대여 검사기입니다.

### 대여 검사기

러스트 컴파일러는 *대여 검사기 (borrow checker)* 로 스코프를 비교하여
대여의 유효성을 판단합니다. 예제 10-17은 예제 10-16 코드의
변수 라이프타임을 주석으로 표시한 모습입니다:

```rust,ignore,does_not_compile
{{#rustdoc_include ../listings/ch10-generic-types-traits-and-lifetimes/listing-10-17/src/main.rs}}
```

<span class="caption">예제 10-17: `r`, `x`의 라이프타임을 각각
`'a`, `'b`로 표현한 주석</span>

`r`의 라이프타임은 `'a`, `x`의 라이프타임은 `'b`로 표현했습니다.
보시다시피 안쪽 `'b` 블록은 바깥쪽 `'a` 라이프타임 블록보다 작습니다.
러스트는 컴파일 타임에 두 라이프타임의 크기를 비교하고,
`'a` 라이프타임을 갖는 `r`이 `'b` 라이프타임을 갖는 메모리를 참조하고 있음을 인지합니다.
하지만 `'b`가 `'a`보다 짧으니, 즉 참조 대상이 참조자보다
오래 살지 못하니 러스트 컴파일러는 이 프로그램을 컴파일하지 않습니다.

예제 10-18은 댕글링 참조를 만들지 않고 정상적으로
컴파일되도록 수정한 코드입니다.

```rust
{{#rustdoc_include ../listings/ch10-generic-types-traits-and-lifetimes/listing-10-18/src/main.rs}}
```

<span class="caption">예제 10-18: 데이터의 라이프타임이
참조자의 라이프타임보다 길어서 문제없는 코드</span>

여기서 `x`의 라이프타임 `'b`는 `'a`보다 더 깁니다.
러스트는 참조자 `r`이 유효한 동안에는 `x` 도 유효하다는 것을 알고 있으므로,
`r`은 `x`를 참조할 수 있습니다.

참조자의 라이프타임이 무엇인지, 러스트가 어떻게 라이프타임을 분석하여
참조자의 유효성을 보장하는지 알아보았습니다.
이제 함수 매개변수와 반환 값에 대한 제네릭 라이프타임을 알아봅시다.

### 함수에서의 제네릭 라이프타임

두 문자열 슬라이스 중 긴 쪽을 반환하는 함수를 작성해 보겠습니다.
이 함수는 두 문자열 슬라이스를 전달받고 하나의 문자열 슬라이스를 반환합니다.
`longest` 함수를 구현하고 나면 예제 10-19 코드로
`The longest string is abcd`가 출력되어야 합니다.

<span class="filename">파일명: src/main.rs</span>

```rust,ignore
{{#rustdoc_include ../listings/ch10-generic-types-traits-and-lifetimes/listing-10-19/src/main.rs}}
```

<span class="caption">예제 10-19: 두 문자열 슬라이스 중 긴 쪽을 찾기 위해
`longest` 함수를 호출하는 `main` 함수</span>

`longest` 함수가 매개변수의 소유권을 얻지 않도록, 문자열 대신
참조자인 문자열 슬라이스를 전달한다는 점을 주목하세요.
어째서 예제 10-19처럼 문자열을
매개변수로 전달하는지는 4장의
[‘문자열 슬라이스를 매개변수로 사용하기’][string-slices-as-parameters]<!-- ignore -->
절을 참고해 주세요.

예제 10-20처럼 `longest` 함수를 구현할 경우,
컴파일 에러가 발생합니다.

<span class="filename">파일명: src/main.rs</span>

```rust,ignore,does_not_compile
{{#rustdoc_include ../listings/ch10-generic-types-traits-and-lifetimes/listing-10-20/src/main.rs:here}}
```

<span class="caption">예제 10-20: 두 문자열 슬라이스 중
긴 쪽을 반환하는 `longest` 함수
(컴파일되지 않음)</span>

나타나는 에러는 라이프타임과 관련되어 있습니다:

```console
{{#include ../listings/ch10-generic-types-traits-and-lifetimes/listing-10-20/output.txt}}
```

이 도움말은 반환 타입에 제네릭 라이프타임 매개변수가
필요하다는 내용입니다. 반환할 참조자가 `x`인지, `y`인지
러스트가 알 수 없기 때문입니다. 사실 우리도 알 수 없죠.
`if` 블록에서는 `x` 참조자를 반환하고
`else` 블록에서는 `y` 참조자를 반환하니까요.

이 함수를 정의하는 시점에서는 함수가 전달받을
구체적인 값을 알 수 없으니, `if`의 경우가 실행될지
`else`의 경우가 실행될지 알 수 없습니다.
전달받은 참조자의 구체적인 라이프타임도 알 수 없습니다.
그러니 예제 10-17, 예제 10-18에서처럼 스코프를 살펴보는
것만으로는 반환할 참조자의 유효성을 보장할 수 없습니다.
대여 검사기도 `x`, `y` 라이프타임이 반환 값의 라이프타임과
어떤 연관이 있는지 알지 못하니 마찬가지입니다.
따라서, 참조자 간의 관계를 제네릭 라이프타임 매개변수로
정의하여 대여 검사기가 분석할 수 있도록 해야 합니다.

### 라이프타임 명시 문법

라이프타임을 명시한다고 해서 참조자의 수명이 바뀌진 않습니다. 그보다는
여러 참조자에 대한 수명에 영향을 주지 않으면서 서로 간 수명의 관계가
어떻게 되는지에 대해 기술하는 것입니다. 함수 시그니처에 제네릭 타입 매개변수를
작성하면 어떤 타입이든 전달할 수 있는 것처럼, 함수에 제네릭 라이프타임
매개변수를 명시하면 어떠한 라이프타임을 갖는 참조자라도 전달할 수 있습니다.

라이프타임 명시 문법은 약간 독특합니다. 라이프타임 매개변수의
이름은 어퍼스트로피(`'`)로 시작해야 하며, 보통은 제네릭 타입처럼
매우 짧은 소문자로 정합니다. 대부분의 사람들은 첫 번째 라이프타임을
명시할 때 `'a`를 사용합니다. 라이프타임 매개변수는 참조자의 `&` 뒤에
위치하며, 공백을 한 칸 입력하여 참조자의 타입과 분리합니다.

다음은 순서대로 라이프타임 매개변수가 없는 `i32` 참조자,
라이프타임 매개변수 `'a`가 있는 `i32` 참조자, 마찬가지로
라이프타임 매개변수 `'a`가 있는 가변 참조자에 대한 예시입니다.

```rust,ignore
&i32        // 참조자
&'a i32     // 명시적인 라이프타임이 있는 참조자
&'a mut i32 // 명시적인 라이프타임이 있는 가변 참조자
```

자신의 라이프타임 명시 하나만 있는 것으로는 큰 의미가 없습니다. 라이프타임
명시는 러스트에게 여러 참조자의 제네릭 라이프타임 매개변수가 서로 어떻게
연관되어 있는지 알려주는 용도이기 때문입니다. `longest` 함수의 컨텍스트에서
라이프타임 명시가 서로에게 어떤 식으로 연관 짓는지 실험해 봅시다.

### 함수 시그니처에서 라이프타임 명시하기

라이프타임 명시를 함수 시그니처에서 사용하기 위해서는 제네릭
*타입* 매개변수를 사용할 때처럼 함수명과 매개변수 목록 사이의
꺾쇠괄호 안에 제네릭 *라이프타임* 매개변수를 선언할 필요가 있습니다.

시그니처에서는 다음과 같은 제약사항을 표현하려고 합니다: 두 매개변수의
참조자 모두가 유효한 동안에는 반환된 참조자도 유효할 것이라는 점이지요.
이는 매개변수들과 반환 값 간의 라이프타임 관계입니다.
예제 10-21과 같이 이 라이프타임에 `'a`라는 이름을 붙여
각 참조자에 추가하겠습니다.

<span class="filename">파일명: src/main.rs</span>

```rust
{{#rustdoc_include ../listings/ch10-generic-types-traits-and-lifetimes/listing-10-21/src/main.rs:here}}
```

<span class="caption">예제 10-21: 시그니처 내 모든 참조자가
동일한 라이프타임 `'a`를 가져야 함을 나타낸 `longest` 함수
정의</span>

이 코드는 정상적으로 컴파일되며, 예제 10-19의 `main` 코드로
실행하면 우리가 원했던 결과가 나옵니다.

이 함수 시그니처는 러스트에게, 함수는 두 매개변수를 갖고 
둘 다 적어도 라이프타임 `'a`만큼 살아있는 문자열 슬라이스이며,
반환하는 문자열 슬라이스도 라이프타임 `'a`만큼 살아있다는
정보를 알려줍니다. 이것의 실제 의미는, `longest` 함수가
반환하는 참조자의 라이프타임은 함수 인수로서 참조된 값들의
라이프타임 중 작은 것과 동일하다는 의미입니다. 이러한 관계가
바로 러스트로 하여금 이 코드를 분석할 때 사용하도록 만들고
싶었던 것입니다.

함수 시그니처에 라이프타임 매개변수를 지정한다고 해서, 전달되는 값이나
반환 값의 라이프타임이 변경되는 건 아니라는 점을 기억해 두세요.
어떤 값이 제약 조건을 지키지 않았을 때 대여 검사기가 불합격
판정을 내릴 수 있도록 명시할 뿐입니다. `longest` 함수는 `x`와 `y`가
얼마나 오래 살지 정확히 알 필요는 없고, 이 시그니처를 만족하는
어떤 스코프를 `'a`로 대체할 수 있다는 점만 알면 됩니다.

라이프타임을 함수에 명시할 때는 함수 본문이 아닌, 함수 시그니처에
적습니다. 라이프타임 명시는 함수 시그니처의 타입들과 마찬가지로 함수에
대한 계약서의 일부가 됩니다. 함수 시그니처가 라이프타임 계약을 가지고
있다는 것은 러스트 컴파일러가 수행하는 분석이 좀 더 단순해질 수 있음을
의미합니다. 만일 함수가 명시된 방법이나 함수가 호출된 방법에 문제가
있다면, 컴파일러 에러가 해당 코드의 지점과 제약사항을 좀 더 정밀하게
짚어낼 수 있습니다. 그렇게 하는 대신 러스트 컴파일러가 라이프타임 간의
관계에 대해 개발자가 의도한 바를 더 많이 추론했다면, 컴파일러는 문제의
원인에서 몇 단계 떨어진 코드의 사용만을 짚어내는 것밖에는 할 수
없을지도 모릅니다.

`longest` 함수에 구체적인 참조자들이 넘겨질 때 `'a`에 대응되는
구체적인 라이프타임은 `x` 스코프와 `y` 스코프가 겹치는 부분입니다.
바꿔 말하면, `x` 라이프타임과 `y` 라이프타임 중 더 작은 쪽이
제네릭 라이프타임 `'a`의 구체적인 라이프타임이 됩니다.
반환하는 참조자도 동일한 라이프타임 매개변수 `'a`를 명시했으므로,
`x`, `y` 중 더 작은 라이프타임 내에서는 `longest`가 반환한
참조자의 유효함을 보장할 수 있습니다.

서로 다른 구체적인 라이프타임을 가진 참조자를 `longest` 함수에
넘겨보면서, 라이프타임 명시가 어떤 효과를 내는지 알아봅시다.
예제 10-22에 간단한 예제가 있습니다.

<span class="filename">파일명: src/main.rs</span>

```rust
{{#rustdoc_include ../listings/ch10-generic-types-traits-and-lifetimes/listing-10-22/src/main.rs:here}}
```

<span class="caption">예제 10-22: 서로 다른 구체적인 라이프타임을 가진
`String` 값의 참조자로 `longest` 함수 호출하기</span>

`string1`은  바깥쪽 스코프가 끝나기 전까지,
`string2`는 안쪽 스코프가 끝나기 전까지 유효합니다.
`result`는  안쪽 스코프가 끝나기 전까지 유효한 무언가를 참조합니다.
대여 검사기는 이 코드를 문제 삼지 않습니다.
실행하면 `The longest string is long string is long`이 출력됩니다.

다음은 두 인수 중 하나의 라이프타임이
`result` 참조자의 라이프타임보다 작을 경우입니다.
`result` 변수의 선언을 안쪽 스코프에서 밖으로 옮기고,
값의 대입은 `string2`가 있는 안쪽 스코프에 남겨보겠습니다.
그리고 `result`를 사용하는 `println!` 구문을 안쪽 스코프가
끝나고 난 이후의 바깥쪽 스코프로 옮겨보겠습니다.
이렇게 수정한 예제 10-23 코드는 컴파일할 수 없습니다.

<span class="filename">파일명: src/main.rs</span>

```rust,ignore,does_not_compile
{{#rustdoc_include ../listings/ch10-generic-types-traits-and-lifetimes/listing-10-23/src/main.rs:here}}
```

<span class="caption">예제 10-24: `string2`가 스코프 밖으로 벗어나고 나서
`result` 사용해 보기</span>

컴파일하면 다음과 같은 에러가 발생합니다:

```console
{{#include ../listings/ch10-generic-types-traits-and-lifetimes/listing-10-23/output.txt}}
```

이 에러는 `println!` 구문에서 `result`가 유효하려면 `string2` 가
바깥쪽 스코프가 끝나기 전까지 유효해야 한다는 내용입니다.
함수 매개변수와 반환 값에 모두 동일한 라이프타임 매개변수 `'a`를 명시했으므로,
러스트는 문제를 정확히 파악할 수 있습니다.

사실 우리 눈으로 보기에는 코드에 문제가 없어 보입니다.
`string1`의 문자열이 `string2` 보다 더 기니까 `result`는
`string1`을 참조하게 될 테고, `println!` 구문을 사용하는 시점에
`string1`의 참조자는 유효하니까요. 하지만 컴파일러는 이 점을 알아챌 수 없습니다.
러스트가 전달받은 것은 ‘`longest` 함수가 반환할 참조자의 라이프타임은
매개변수의 라이프타임 중 작은 것과 동일하다’라는 내용이었으니,
대여 검사기는 예제 10-23 코드가 잠재적으로 유효하지 않은 참조자를
가질 수도 있다고 판단합니다.

`longest` 함수에 다양한 값, 다양한 라이프타임의 참조자를 넘겨보고,
반환한 참조자를 여러 방식으로 사용해 보세요. 컴파일하기 전에
코드가 대여 검사기를 통과할 수 있을지 혹은 없을지 예상해 보고,
여러분의 생각이 맞았는지 확인해 보세요!

### 라이프타임의 측면에서 생각하기

라이프타임 매개변수 명시의 필요성은 함수가 어떻게 동작하는지에 따라서 달라집니다.
예를 들어, `longest` 함수를 제일 긴 문자열 슬라이스를 반환하는 게 아니라,
항상 첫 번째 매개변수를 반환하도록 바꾸었다고 가정해 봅시다.
그러면 이제 `y` 매개변수에는 라이프타임을 지정할 필요가 없습니다.
다음 코드는 정상적으로 컴파일됩니다:

<span class="filename">파일명: src/main.rs</span>

```rust
{{#rustdoc_include ../listings/ch10-generic-types-traits-and-lifetimes/no-listing-08-only-one-reference-with-lifetime/src/main.rs:here}}
```

매개변수 `x`와 반환 타입에만 라이프타임 매개변수 `'a`가 지정되어 있습니다.
`y`의 라이프타임은 `x`나 반환 값의 라이프타임과 전혀 관계없으므로,
매개변수 `y`에는 `'a`를 지정하지 않았습니다.

참조자를 반환하는 함수를 작성할 때는 반환 타입의 라이프타임 매개변수가
함수 매개변수 중 하나와 일치해야 합니다.
반환할 참조자가 함수 매개변수중 하나를 참조하지 *않을* 유일한 가능성은
함수 내부에서 만들어진 값의 참조자를 반환하는 경우입니다.
하지만 이 값은 함수가 끝나는 시점에 스코프를 벗어나므로 댕글링 참조가
될 것입니다. 다음과 같이 `longest` 함수를 구현하면 컴파일할 수
없습니다:

<span class="filename">파일명: src/main.rs</span>

```rust,ignore,does_not_compile
{{#rustdoc_include ../listings/ch10-generic-types-traits-and-lifetimes/no-listing-09-unrelated-lifetime/src/main.rs:here}}
```

반환 타입에 `'a`를 지정했지만,
반환 값의 라이프타임이 그 어떤 매개변수와도
관련 없으므로 컴파일할 수 없습니다.
나타나는 에러 메시지는 다음과 같습니다:

```console
{{#include ../listings/ch10-generic-types-traits-and-lifetimes/no-listing-09-unrelated-lifetime/output.txt}}
```

`result`는  `longest` 함수가 끝나면서 스코프를 벗어나 정리되는데,
함수에서 `result`의 참조자를 반환하려고 하니 문제가 발생합니다.
여기서 댕글링 참조가 발생하지 않도록 라이프타임 매개변수를 지정할 방법은 없습니다.
그리고 러스트는 댕글링 참조를 생성하는 코드를 눈감아주지 않죠.
이런 상황을 해결하는 가장 좋은 방법은 참조자 대신 값의 소유권을 갖는
데이터 타입을 반환하여 함수를 호출한 함수 측에서
값을 정리하도록 하는 것입니다.

라이프타임 문법의 근본적인 역할은 함수의 다양한 매개변수와
반환 값의 라이프타임을 연결하는 데에 있습니다.
한번 라이프타임을 연결해 주고 나면, 러스트는 해당 정보를 이용해
댕글링 포인터 생성을 방지하고, 메모리 안전 규칙을 위배하는 연산을 배제합니다.

### 구조체 정의에서 라이프타임 명시하기

여태껏 정의해 본 구조체들은 모두 소유권이 있는 타입을 들고 있었습니다.
구조체가 참조자를 들고 있도록 할 수도 있지만, 이 경우 구조체 정의 내 모든
참조자에 라이프타임을 명시해야합니다. 예제 10-24는 문자열 슬라이스를
보유하는 `ImportantExcerpt` 구조체를 나타냅니다:

<span class="filename">파일명: src/main.rs</span>

```rust
{{#rustdoc_include ../listings/ch10-generic-types-traits-and-lifetimes/listing-10-24/src/main.rs}}
```

<span class="caption">예제 10-24: 참조자를 보유하여
라이프타임 명시가 필요한 구조체</span>

이 구조체에는 문자열 슬라이스를 보관하는 `part` 참조자 필드가 하나 있습니다.
구조체의 제네릭 라이프타임 매개변수의 선언 방법은 제네릭 데이터 타입과 마찬가지로,
제네릭 라이프타임 매개변수의 이름을 구조체 이름 뒤 꺾쇠괄호 내에 선언하고
구조체 정의 본문에서 라이프타임 매개변수를 이용합니다. 예제 10-25의
라이프타임 명시는 ‘`ImportantExcerpt` 인스턴스는 `part` 필드가 보관하는
참조자의 라이프타임보다 오래 살 수 없다’라는 의미입니다.

`main` 함수에서는 `novel` 변수가 소유하는 `String`의
첫 문장에 대한 참조자로 `ImportantExcerpt` 구조체를 생성합니다.
`novel` 데이터는 `ImportantExcerpt` 인스턴스가 생성되기 전부터 존재하며,
`ImportantExcerpt` 인스턴스가 스코프를 벗어나기 전에는
`novel`이 스코프를 벗어나지도 않으니,
`ImportantExcerpt` 인스턴스는 유효합니다.

### 라이프타임 생략

모든 참조자는 라이프타임을 가지며, 참조자를 사용하는 함수나
구조체는 라이프타임 매개변수를 명시해야 함을 배웠습니다. 하지만
4장에서 본 예제 4-9의 함수는, 예제 10-25에서 다시 보여 드리겠지만,
라이프타임 명시가 없었는데도 컴파일 할 수 있었습니다.

<span class="filename">파일명: src/lib.rs</span>

```rust
{{#rustdoc_include ../listings/ch10-generic-types-traits-and-lifetimes/listing-10-25/src/main.rs:here}}
```

<span class="caption">예제 10-25: 4장에서 정의했던,
매개변수, 반환 타입이 참조자인데도 라이프타임 명시 없이
컴파일 가능한 함수</span>

이 함수에 라이프타임을 명시하지 않아도 컴파일 할 수 있는 이유는 러스트의 역사에서
찾아볼 수 있습니다. 초기 버전(1.0 이전) 러스트에서는 이 코드를 컴파일할 수
없었습니다. 모든 참조자는 명시적인 라이프타임이 필요했었죠.
그 당시 함수 시그니처는 다음과 같이 작성했습니다:

```rust,ignore
fn first_word<'a>(s: &'a str) -> &'a str {
```

수많은 러스트 코드를 작성하고 난 후, 러스트 팀은 러스트 프로그래머들이
특정한 상황에서 똑같은 라이프타임 명시를 계속 똑같이 작성하고 있다는 걸 알아냈습니다.
이 상황들은 예측 가능한 상황들이었으며, 몇 가지 결정론적인 (deterministic) 패턴을
따르고 있었습니다. 따라서 러스트 팀은 컴파일러 내에 이 패턴들을 프로그래밍하여,
이러한 상황들에서는 라이프타임을 명시하지 않아도
대여 검사기가 추론할 수 있도록 하였습니다.

앞으로 더 많은 결정론적 패턴이 컴파일러에 추가될 가능성이 있다는 사실은
이러한 러스트의 역사와 관련되어 있습니다. 나중에는 라이프타임 명시가 필요한
상황이 더욱 적어질지도 모르지요.

러스트의 참조자 분석 기능에 프로그래밍 된 이 패턴들을
*라이프타임 생략 규칙 (lifetime elision rules)* 이라고 부릅니다. 이 규칙은
프로그래머가 따라야 하는 규칙이 아닙니다. 그저 컴파일러가 고려하는 특정한 사례의
모음이며, 여러분의 코드가 이에 해당할 경우 라이프타임을 명시하지 않아도 될 따름입니다.

생략 규칙이 완전한 추론 기능을 제공하는 것은 아닙니다. 만약 러스트가
이 규칙들을 적용했는데도 라이프타임이 모호한 참조자가 있다면,
컴파일러는 이 참조자의 라이프타임을 추측하지 않습니다.
컴파일러는 추측 대신 에러를 발생시켜서, 여러분이 라이프타임
명시를 추가하여 문제를 해결하도록 할 것입니다.

먼저 몇 가지를 정의하겠습니다. 함수나 메서드 매개변수의 라이프타임은 *입력 라이프타임 (input lifetime)* 이라 하며,
반환 값의 라이프타임은 *출력 라이프타임 (output lifetime)* 이라 합니다.

라이프타임 명시가 없을 때 컴파일러가 참조자의 라이프타임을
알아내는 데 사용하는 규칙은 세 개입니다. 첫 번째 규칙은
입력 라이프타임에 적용되고, 두 번째 및 세 번째 규칙은 출력 라이프타임에 적용됩니다.
세 가지 규칙을 모두 적용했음에도 라이프타임을 알 수 없는 참조자가 있다면
컴파일러는 에러와 함께 작동을 멈춥니다.
이 규칙은 `fn` 정의는 물론 `impl` 블록에도 적용됩니다.

첫 번째 규칙은, 컴파일러가 참조자인 매개변수 각각에게 라이프타임 매개변수를
할당한다는 것입니다. `fn foo<'a>(x: &'a i32)`처럼 매개변수가 하나인 함수는
하나의 라이프타임 매개변수를 갖고, `fn foo<'a, 'b>(x: &'a i32, y: &'b i32)`처럼
매개변수가 두 개인 함수는 두 개의 개별 라이프타임 매개변수를
갖는 식입니다.

두 번째 규칙은, 만약 입력 라이프타임 매개변수가 딱 하나라면,
해당 라이프타임이 모든 출력 라이프타임에 대입된다는 것입니다:
`fn foo<'a>(x: &'a i32) -> &'a i32`처럼 말이지요.

세 번째 규칙은, 입력 라이프타임 매개변수가 여러 개인데,
그중 하나가 `&self`나 `&mut self`라면, 즉 메서드라면
`self`의 라이프타임이 모든 출력 라이프타임 매개변수에 대입됩니다.
이 규칙은 메서드 코드를 깔끔하게 만드는 데 기여합니다.

한번 우리가 컴파일러라고 가정해 보고, 예제 10-25의
`first_word` 함수 시그니처 속 참조자의 라이프타임을
이 규칙들로 알아내 봅시다. 시그니처는 참조자에 관련된
어떤 라이프타임 명시도 없이 시작됩니다:

```rust,ignore
fn first_word(s: &str) -> &str {
```

첫 번째 규칙을 적용해, 각각의 매개변수에 라이프타임을 지정해 봅시다.
평범하게 `'a`라고 해보죠.
시그니처는 이제 다음과 같습니다:

```rust,ignore
fn first_word<'a>(s: &'a str) -> &str {
```

입력 라이프타임이 딱 하나밖에 없으니 두 번째 규칙을 적용합니다.
두 번째 규칙대로 출력 라이프타임에 입력 매개변수의 라이프타임을 대입하고 나면,
시그니처는 다음과 같습니다:

```rust,ignore
fn first_word<'a>(s: &'a str) -> &'a str {
```

함수 시그니처의 모든 참조자가 라이프타임을 갖게 됐으니,
컴파일러는 프로그래머에게 이 함수의 라이프타임 명시를 요구하지 않고도
계속 코드를 분석할 수 있습니다.

이번엔 다른 예제로 해보죠. 예제 10-20에서의 아무런 라이프타임
매개변수 없는 `longest` 함수를 이용해 보겠습니다:

```rust,ignore
fn longest(x: &str, y: &str) -> &str {
```

첫 번째 규칙을 적용해, 각각의 매개변수에 라이프타임을 지정해 봅시다.
이번에는 매개변수가 두 개니, 두 개의 라이프타임이 생깁니다.

```rust,ignore
fn longest<'a, 'b>(x: &'a str, y: &'b str) -> &str {
```

입력 라이프타임이 하나가 아니므로 두 번째 규칙은 적용하지 않습니다.
`longest` 함수는 메서드가 아니니, 세 번째 규칙도 적용할 수 없습니다.
세 가지 규칙을 모두 적용했는데도 반환 타입의 라이프타임을 알아내지 못했습니다.
예제 10-20의 코드를 컴파일하면 에러가 발생하는 이유가 바로 이 때문입니다.
컴파일러가 라이프타임 생략 규칙을 적용해 보았지만,
이 시그니처 안에 있는 모든 참조자의 라이프타임을
알아내지 못했습니다.

세 번째 규칙은 메서드 시그니처에만 적용되니,
메서드에서의 라이프타임을 살펴보고, 왜 세 번째 규칙 덕분에
메서드 시그니처의 라이프타임을 자주 생략할 수 있는지 알아봅시다.

### 메서드 정의에서 라이프타임 명시하기

라이프타임을 갖는 메서드를 구조체에 구현하는 문법은
예제 10-11에서 본 제네릭 타입 매개변수 문법과 같습니다.
라이프타임 매개변수의 선언 및 사용 위치는 구조체 필드나
메서드 매개변수 및 반환 값과 연관이 있느냐 없느냐에 따라 달라집니다.

라이프타임이 구조체 타입의 일부가 되기 때문에,
구조체 필드의 라이프타임 이름은 `impl` 키워드 뒤에
선언한 다음 구조체 이름 뒤에 사용해야 합니다.

`impl` 블록 안에 있는 메서드 시그니처의 참조자들은 구조체 필드에 있는
참조자의 라이프타임과 관련되어 있을 수도 있고, 독립적일 수도 있습니다.
또한 라이프타임 생략 규칙으로 인해 메서드 시그니처에
라이프타임을 명시하지 않아도 되는 경우도 있습니다.
예제 10-24의 `ImportantExcerpt` 구조체로 예시를 들어보겠습니다.

먼저 `level`이라는 메서드가 있습니다.
이 메서드의 매개변수는 `self` 참조자 하나뿐이며, 반환 값은 참조자가 아닌 그냥 `i32` 값입니다.

```rust
{{#rustdoc_include ../listings/ch10-generic-types-traits-and-lifetimes/no-listing-10-lifetimes-on-methods/src/main.rs:1st}}
```

`impl` 뒤에서 라이프타임 매개변수를 선언하고
타입명 뒤에서 사용하는 과정은 필수적이지만,
첫 번째 생략 규칙으로 인해 `self` 참조자의 라이프타임을 명시할 필요는 없습니다.

다음은 세 번째 라이프타임 생략 규칙이 적용되는 예시입니다:

```rust
{{#rustdoc_include ../listings/ch10-generic-types-traits-and-lifetimes/no-listing-10-lifetimes-on-methods/src/main.rs:3rd}}
```

두 개의 입력 라이프타임이 있으니, 러스트는 첫 번째 라이프타임 생략 규칙대로
`&self`, `announcement`에 각각의 라이프타임을 부여합니다.
그다음, 매개변수 중 하나가 `&self`이니 반환 타입에 `&self`의 라이프타임을 부여합니다.
이제 모든 라이프타임이 추론되었네요.

### 정적 라이프타임

정적 라이프타임 (static lifetime), 즉 `'static`이라는 특별한 라이프타임을 다뤄봅시다.
`'static` 라이프타임은 해당 참조자가 프로그램의 전체 생애주기 동안 살아있음을 의미합니다.
모든 문자열 리터럴은 `'static` 라이프타임을 가지며, 다음과 같이 명시할 수 있습니다.

```rust
let s: &'static str = "I have a static lifetime.";
```

이 문자열의 텍스트는 프로그램의 바이너리 내에
직접 저장되기 때문에 언제나 이용할 수 있습니다.
따라서 모든 문자열 리터럴의 라이프타임은 `'static`입니다.

`'static` 라이프타임을 이용하라는 제안이 담긴 에러 메시지를 보시게 될 수도 있습니다.
하지만 어떤 참조자를 `'static`으로 지정하기 전에 해당 참조자가 반드시
프로그램의 전체 라이프타임동안 유지되어야만 하는 참조자인지, 그리고 그것이
진정 원하는 것인지 고민해 보라고 당부하고 싶습니다. `'static` 라이프타임을
제안하는 에러 메시지는 대부분의 경우 댕글링 참조를 만들다가 발생하거나, 사용
가능한 라이프타임이 잘못 짝지어져서 발생합니다. 이러한 경우 바람직한 해결책은
그런 문제를 고치는 것이지, `'static` 라이프타임이 아닙니다.

## 제네릭 타입 매개변수, 트레이트 바운드, 라이프타임을 한 곳에 사용해 보기

제네릭 타입 매개변수, 트레이트 바운드, 라이프타임이 문법이 함수 하나에
전부 들어간 모습을 살펴봅시다!

```rust
{{#rustdoc_include ../listings/ch10-generic-types-traits-and-lifetimes/no-listing-11-generics-traits-and-lifetimes/src/main.rs:here}}
```

예제 10-21에서 본 두 개의 문자열 슬라이스 중 긴 쪽을 반환하는
`longest` 함수입니다. 하지만 이번에는 `where` 구문에 명시한 바와 같이
`Display` 트레이트를 구현하는 제네릭 타입 `T`에 해당하는 `ann` 매개변수를
추가했습니다. 이 추가 매개변수는 `{}`를 사용하여 출력될 것인데,
이 때문에 `Display` 트레이트 바운드가 필요합니다. 라이프타임은
제네릭의 일종이므로, 함수명 뒤의 꺾쇠괄호 안에는 라이프타임
매개변수 `'a` 선언과 제네릭 타입 매개변수 `T`가 함께
나열되어 있습니다.

## 정리

이번 장에서는 정말 많은 내용을 배웠네요!
여러분은 제네릭 타입 매개변수, 트레이트, 트레이트 바운드, 제네릭 라이프타임 매개변수를 배웠습니다.
이제 다양한 상황에 맞게 작동하는 코드를 중복 없이 작성할 수 있겠군요.
제네릭 타입 매개변수로는 다양한 타입으로 작동하는 코드를 작성할 수 있고,
트레이트와 트레이트 바운드로는 제네릭 타입을 다루면서도 코드에서 필요한 특정 동작을 보장할 수 있습니다.
라이프타임을 명시하면 이런 유연한 코드를 작성하면서도 댕글링 참조가 발생할 일이 없습니다.
그리고, 이 모든 것들은 컴파일 타임에 분석되어
런타임 성능에 전혀 영향을 주지 않습니다!

이번 장에서 다룬 주제들에서 더 배울 내용이 남았다고 하면 믿어지시나요?
17장에서는 트레이트를 사용하는 또 다른 방법인 트레이트 객체 (trait object) 를 다룰 예정입니다.
매우 고급 시나리오 상에서만 필요하게 될, 라이프타임 명시에 관한 더 복잡한 시나리오도
있습니다. 이와 관련해서는 [러스트 참고 자료 문서][reference]를 읽으셔야 합니다.
하지만 일단 다음 장에서는 러스트에서 여러분의 코드가 원하는 대로
작동함을 보장할 수 있도록 해주는 코드 테스트 작성 방법을 배워보도록 하죠.

[references-and-borrowing]:
ch04-02-references-and-borrowing.html#references-and-borrowing
[string-slices-as-parameters]:
ch04-03-slices.html#string-slices-as-parameters
[reference]: https://doc.rust-lang.org/reference/index.html

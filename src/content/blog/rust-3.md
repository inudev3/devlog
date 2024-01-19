---
author: Inu Jung
pubDatetime: 2022-09-23T15:22:00Z
modDatetime: 2023-12-21T09:12:47.400Z
title: Rust study #3
slug: rust-study
featured: true
draft: false
tags:
  - rust phantomdata trait
description:
---

### PhantomData

`PhantomData<T>`는 struct의 drop에 있어, T타입이 struct에 없어도 있는 것처럼 취급하게 해준다. PhantomData는 사이즈가 zero인 zero-sized-type이다.

이는 drop의 메커니즘을 이해해야 하는데,
drop은 어떤 struct에 대해서, 내부 필드에 있는 타입 T가 있을 때 T가 Drop trait을 구현했는지 찾아서 실행한다. 즉, 어떤 복합타입이 있다면 내부 타입에 대해서 전부 Drop Trait을 찾아 이를 같이 실행한다. 그런데 만약 해당 타입이 포인터, 혹은 박스나 스마트 포인터 등으로 T를 가리키고 있다면(즉 소유하지 않는다면) T가 Drop trait을 구현했는지 여와 상관없이 T의 drop을 실행하지 않는다. 따라서 T의 drop을 실행하기 위해서 struct에 별도의 필드를 할당하는데, 이 때 `PhantomData<T>`를 할당한다. 컴파일러는 `PhantomData<T>`를 보고, T의 Drop trait을 찾아 compount struct를 드랍할 시 drop을 실행한다.
혹은, `PhantomData<T>`를 변성을 변경하기 위한 용도로도 사용한다.
예를 들어 `PhantomData<fn()->T`>는 공변,
`PhantomData<fn(T)->()>`는 반공변,
`PhantomData<fn(T)->T>`는 무공변이다.
다만 이경우 T타입을 소유하지 못해, drop 체크의 대상이 되지 못한다.

### Nonnull

`*mut T` raw pointer와 동일하지만 무공변인 `*mut T` 와 달리 공변적이다.

### Atomic type

AtomicUsize 등의 Atomic 타입이 존재하며, 이 타입들의 메서드는 가변메서드도 전부 &mut self가 아닌 &self 공유 참조를 인자로 한다. 그 이유는 해당 타입이 멀티 쓰레드에서 공유해도 연산에 한번에 하나의 스레드만 접근하게끔 CPU 명령어 수준에서 구현되기 때문이다.
값을 가져오고 저장하는 load, store 부터 이러한 일련의 과정을 하나의 atomic operation으로 구현한 fetch*시리즈, compare* 시리즈 등의 api를 제공한다.
Atomic은 임계 영역 없이 단일 메모리 접근에 대한 상호 배제 `Mutual Exclusion`을 제공한다는 점에서 mutex보다 효율적이다.
Atomic 구현은 기본적으로 unsafecell을 통한 어셈블리 수준의 조작으로 이루어져 있다.

compare_and_exchange의 경우, 메모리 위치에 대한 배타적인 소유권을 필요로하기 때문에(메모리 쓰기), 쓰레드 간에 이를 동기화 하기 위해 많은 비용이 든다. spin lock의 구현은 대부분 compare_and_exchange를 수행하는 루프 하나와, 그 내부에서(실패할경우) load를 수행하는 루프로 이루어져 있다. 즉 `exclusive`한 메모리 ownership을 얻지 못하면, 계속해서 내부의 메모리를 `load`로 읽게 된다. load는 읽기 연산으로, 여러 쓰레드 간에 동시에 접근하는 shared 상태가 가능하다.

    compare_and_exchange의 경우 current value가 동일한 경우 항상 성공한다. 하지만 compare_and_exchange_weak의 경우 current value가 같아도 실패할 수 가 있다.

Compare and Swap 연산의 경우 x86과 AMD 아키텍쳐에서 구현이 다르다.
x86의 경우 실제로 CAS(Compare And Swap)을 원자적 명령어로 구현한다. 그러나 AMD의 경우, 메모리에 대한 배타적 소유권을 필요로하는 LDREX, STREX라는 연산을 사용하는데 , 이 경우 메모리 값을 변경하기 전에 다른 쓰레드가 메모리를 읽기만 하더라도 연산이 실패할 수 있다. 즉 원자적 연산이 아니다. 따라서 ARM에서는 , LDREX와 STREX의 중첩 루프를 사용해서 구현되어 있고, 따라서 성능이 저하된다. 따라서 LDREX와 STREX의 중첩 루프 대신 단순한 2개의 원자적 연산으로 구현하는 대신, `spurious` 가짜로 실패하는 것을 허용하는 `compare_and_exchange_weak `연산을 사용하며, 이 경우 STREX에서 메모리 소유권이 없는 경우 실패하지만 실제 연산은 수행된다.

### Ordering

Atomic 연산에는 Ordering Enum 속성이 있다.
Relaxed Ordering의 경우, 쓰레드 간의 동기화 포인트가 없는 한, 어떤 메모리에서 읽어오는(`load`) 값은 해당 메모리에 저장되었던 어떠한 값이라도 가능하다. 어떤 메모리에 값이 저장되는 일련의 순서를 `modification order`라고 하는데, Relaxed Ordering의 경우, 이 순서가 어떤 순서라도 모두 가능하다.
쓰레드 내에서도, 다른 쓰레드 간에서도 atomic 명령어의 순서가 어떤 순서로든 재배치 될 수 있다. CPU는 이러한 순서 재배치를 통해 실행시간 및 메모리 활용에서 매우 큰 이점을 얻기 때문이다.
따라서 relaxed를 사용하면 뮤텍스 등에서 임계영역 내에서 실행되어야 하는 연산이 락의 범위를 벗어날 수도 있는 위험이 있다.

`Ordering::Released` 오더링은 store 연산과 같이 사용되며, 해당 store 연산 이후에 연산도 쓰레드에서 발생할 수 없고, `load`연산과 같이 `Ordering::Acquired` 오더링이 사용된 다음 연산은 해당 store 연산 이전에 발생한 모든 연산을 볼 수 있어야 한다. 따라서 released와 acquired는 짝지어서 사용된다. `Acquired`는 load연산과 같이 사용되며, 어떤 연산도 해당 load 연산 이전으로 재배치될 수 없다.

load와 store가 한번에 일어나는 연산의 경우 `load`에는 Acquire을, store에는 Released를 자동으로 적용시키는 AcqRel도 있으나, 이 경우 fetch_add 등 하나의 연산만을 동기화해야할 때 사용한다. 넓은 임계영역에 사용할 때는 별도의 Acquire과 Released/ load store 연산을 가지는 것이 낫다.

acquire과 release의 sync는 한번에 하나의 메모리에 관해서만 일어난다. 즉, 하나의 쓰레드에서 acquire로 load하면 해당 쓰레드는 하나의 Release load 쓰레드와만 동기화된다.

복수의 메모리에 대한 연산 순서를 동기화 하려면 SeqCst 오더링을 사용해야 한다. `SeqCst` 는 Sequentially Consistent의 줄임말로, 다른 SeqCst 오더링을 가진 연산과 순서관계를 형성하고 이를 일관성있게 적용한다. 즉, acquire와 release가 하나의 메모리 위치에 대한 순서를 보장한다면, SeqCst는 다른 메모리 위치의 경우에도 순서를 보장한다. SeqCst는 Acquire나 Release와도 상호작용하며 이들보다 한단계 더 강한 보장이다.
즉 다음과 같은 쓰레드가 있다고 하면

```rust
let t1 = spawn(move||{
	while !y.load(Ordering::SeqCst){}
	if x.load(Ordering::SeqCst){z.fetch_add(1,Ordering::Releaxed)}
})
```

x가 true일 때 y가 false일 수 없다.

### fetch

fetch_add, fetch_sub 등의 atomic 연산이 존재하는데, compare_and_exchange와 다른 점은, currentvalue나 newvalue를 주지 않고 주어진 연산을 수행하게 하기 때문에 항상 성공한다는 점이다. 다만 `fetch_update`나 `fetch_or` 등은 조금 이질적으로 클로저를 인자로 받는데, 아키텍쳐/플랫폼에 따라 `compare_and_exchange` 루프로 이루어어져 있다. 한번에 주어진 연산을 수행하지 않고 클로저를 받아서 조건에 따라 수행하기 때문이다.

### Monomorphization

제네릭과 제네릭과 동일한 역할을 하는 impl 키워드를 사용하면 컴파일 타입에 가능한 타입들을 모두 생성한다. 즉 런타임 타입소거와는 정 반대로 컴파일 타임을 희생해서 가능한 `non-generic`타입들을 모두 생성해버린다. 즉 예를 들어 HashMap<T,R>의 인스턴스를 HashMap<String, Int>로 생성하면 해쉬맵의 메서드를 String,Int에 맞게 생성하고 각 타입에 맞게 최적화할 수 있다.
물론 실제로 사용하는 메서드만 생성하는 최적화를 수행하긴 하지만 러스트 코드의 바이너리 파일의 사이즈가 조금 커지게 하는 원인이다.
생성하는 메커니즘은 컴파일 타임에 제네릭 타입을 실제 타입으로 바꾸어 메서드를 생성한다. 이 때 타입을 알고 이를 인라인화하여 추가로 최적화할 수도 있다.

어떠한 구체적인 타입을 찾아 실행하는 것을 `dispatch`라고 하고, 정적인 `static dispatch`와 동적 디스패치 `dynamic dispatch`두 가지로 나눈다.
정적 디스패치는 impl 및 제네릭 등의 키워드로 컴파일 타입에 구상타입을 찾아서 제네릭 메서드를 단형화 monomorphize하여 생성하고 실행한다.

하지만 정적 디스패치는 코드를 작성할 때에도 구상 타입이 정해져야 한다. 만약 우리가 코드를 작성할 때도 실행시에만 동적으로 결정되게끔 타입을 작성할 수는 없을까?

dyn trait를 사용할 수 있다 이 때 Sized 바운드를 가져야 하는데, 이유는 코드를 실행할 때 스택에 할당되는 사이즈를 알아야하기 때문이다. 정적 디스패치를 사용하는 경우, 당연히 구상 타입을 컴파일 타입에 알게 되므로 컴파일러가 스택에 할당될 타입의 크기를 알 수 있다. 그러나 동적 디스패치를 사용하는 경우 런타임에 타입을 결정되는, 타입에 Sized 바운드가 없다면 컴파일러가 스택에 할당할 사이즈를 알 방법이 없으므로 컴파일 할 수 없다.
모든 trait에는 Sized 바운드가 암묵적으로 있다. 심지어 제네릭 타입도 Sized 바운드를 가진다. 그러나 dyn trait 또는 슬라이스 같이 크기가 동적으로 결정되는 타입은 Sized 바운드가 없다. 이를 동적 사이즈 타입 , `DST`라고 한다.

이러한 제한을 우회하는 방법은 고정된 사이즈를 가진 타입으로 한번 더 래핑하는 방법이 있다. 예를 들어, 동적 타입 대해 Box 또는 포인터, Arc로 래핑하면 컴파일 타임에 고정된 크기를 가질 수 있다.

Box의 제네릭 T는 `?Sized`로 옵트 아웃되어 있기 때문에 `Box<dyn trait>`같은 동적 타입을 사용할 수 있다.

이러한 동적 타입에 대한 포인터 및 스마트포인터는 정적 `Sized` 타입에 대한 포인터에 비해 2배의 크기를 가진다.
슬라이스의 경우, 슬리이스의 `len()`에 대한 정보를,
trait 객체의 경우 `vtable`로의 참조를 추가적으로 가진다.
vtable 이란 trait에 각 메서드에 대한 포인터가 모여있는 테이블이다. vtable이란 각 타입에 대해서 `dyn trait` 오브젝트의 메서드를 참조하는 포인터가 모인 테이블로, 컴파일 타임에 생성된다. 따라서 동적 trait 객체를 가리키는 `Box, Arc, &` 등은 일반적인 참조보다 vtable에 대한 참조를 하나 더 포함하며 따라서 `wide pointer 혹은  fat pointer` 라고 불린다. `Box나 Arc`의 경우 `*mut T`의 raw pointer가 vtable 참조를 포함하는 fat pointer가 된다. 즉, `&(T as trait)::method)` 와 같은 메서드에 대한 참조를 전부 모아논 테이블이라고 보면된다.
trait 오브젝트에 대한 bound를 추가할 수도 있다. 이 경우 multiple trait에 대해서 각각 vtable이 생성되어 trait 개수만큼 vtable 참조가 추가된다.
또한 trait에 associated type이 있는 경우 `dyn` 오브젝트는 이를 명시해줘야 한다. `dyn Iterator<Item=String>`와 같은 식이다.타입 파라미터에 associated type을 명시할 수 있는 것은 dyn trait object만으로 한정되며, 다른 곳에서는 제네릭 타입만이 허용된다. associated type은 vtable 상에서 참조되지 않는 `타입`이므로 (타입은 주소가 없다) dyn trait를 사용할 때는 명시해줘야 하는 것이다.
그런데 만약 trait 메서드 중, &self 를 인자로 받는 메서드가 아니라 associated function이 있다고 해보자. associated function은 타입을 명시하여 호출해야 하고, 타입의 impl 마다 오버라이딩 할 수 있으므로 우리는 `(dyn Hei)::fn()`등으로 호출할 수 없다. 또한, &self 인자가 없으므로 `vtable`에 해당 associated function은 존재하지 않는다. 물론 `static dispatch`에서는 제네릭 타입을 사용하여 호출할 수 있다(impl 문법으로는 불가능하다)
그 외에 &self 참조를 받지만 trait object에서 제외하고 싶은 메서드가 있다면 Sized bound를 추가하여 vtable에서 opt-out 한다. `vtable`은 Sized bound가 있는 메서드는모두 제외해버린다. 컴파일 타임에 스택 사이즈를 알 수 있기 때문에 포인터 참조가 필요없기 때문이다. 따라서 trait object로 호출할 수 없다. 또한 어떤 trait이 trait object로 사용되는 것을 막으려면 trait 자체에 Sized 바운드를 걸 수도 있다.

그런데 Extend trait 의 경우를 살펴보면 조금 재미난 사실을 알 수 있다. Extend trait은 다음과 같다.

```rust
pub trait Extend<A>{
	fn extend<I>(&mut self, iter:I)
	where
		I:IntoIterator<Item=A>
}
```

즉 A는 자기 자신의 Item, I는 인자가 될 iterator로 trait의 제네릭에 더해 메서드의 제네릭이 추가적으로 존재한다. 이터레이터와 아이템 조합마다 전부 다른 구상 타입이 생성된다. 만약 dyn trait object가 생성된다면, extend는 여전히 Iterator인 I타입에 대해서 제네릭하다. 이를 타입 파리미터로 명시할 수 있는 방법은 없다. 모든 가능한 구상타입을 전부 가지는 vtable을 만드는 방법은 크기가 너무 커져서 사용할 수 없다. 따라서 이 경우(메서드 자체가 제네릭한 경우) vtable은 생성되지 않는다.

또한 trait safe object는 Self를 리턴하는 메서드를 전부 제외한다. 예를 들어 Self를 리턴하는 `Clone` trait 의 clone 메서드를 살펴보면

```rust
pub fn clone(v:&dyn Clone){
	let x = v.clone() // not sized, 에러
}
```

Clone trait object를 사용할 경우 Self를 리턴하는 clone() 메서드는 컴파일 타임에 당연히 사이즈를 알 수 없다. Size를 알 수 없는 코드는 컴파일 되지 않고 작성할 수 없다. 따라서 trait object의 vtable에서 제외된다. 또한, receiver가 포인터가 아닌 `self` 밸류 타입인 경우 receiver를 consume하고 소유권을 가지며, 따라서 함수 인자의 self가 `Sized` 되지 않는 dyn trait object에서 제외된다. 즉 dyn trait object는

1. receiver가 self에 대한 포인터이며
2. 반환하는 타입이 self 배률 타입이 아니고
3. Sized 바운드가 없고
4. 다른 제네릭 파라미터가 없어야 한다.
   등의 제약을 가지고 있다. 위 조건에 해당되는 메서드는 모두 제외된다.

함수의 인자로 dyn trait object에 대한 포인터를 받는다고 하자, 함수가 종료될 때 스마트 포인터가 drop되면, dyn trait object가 Drop을 구현했다는 바운드가 없는 상황에서 어떻게 처리될까? 정답은 vtable은 자동으로 drop된다이다. 모든 vtable은 암묵적으로 해당 구상 타입에 대하 drop function을 가리키고 구상 타입이 drop될 수 있게 한다.

dyn trait , `[]`슬라이스, str 객체를 포인터없이 사용할 수 없는 이유는 Sized가 아니기 때문이다. 포인터를 통해서만 컴파일 타임에 스택에 할당될 사이즈를 알 수 있다.

또한 이러한 unsized 타입을 구조체 필드로 갖는 경우, 항상 마지막 필드에 위치해야 한다. 중간에 위치할 경우 컴파일 타임에 그 다음 필드를 접근할 수가 없기 때문이다. 또한 하나라도 Unsized 필드를 포함하는 구조체는 Unsized 타입이다.

impl Fn()과 &dyn Fn() 으로 클로져를 사용하는 경우를 구분할 수 있어야 한다. 전자는 제네릭하므로 다른 dyn trait object의 인자로 사용될 수없고 후자는 가능하다.

### Executor

async, await의 Future를 프로그램의 탑레벨 main 함수에서 실행할 수 있게 해주는 crate을 Executor crate이라고 한다. executor는 리소스들을 관리하는데 각 async 채널마다 state를 가진다. 이 때 이 리소스들의 state에 변화가 발생하면 운영체제에게 async가 다시 깨어나서 진행될 수 있도록 알려달라고 한다. 즉 yield 시마다 반환하는 state를 관리하면서 운영체제에게 이를 관리 감독할 수 있게 한다. 대표적인 executor crate는 tokio가 있다.
대략 이러한 코드를 사용한다.

```rust
let runtime = tokio::runtime::Runtime::new();
let mut network = some_future();
let mut terminal = some_future_2();
let mut copy = tokio::io::copy(&mut f1, &mut f2);
runtime.block_on(async{
	select!{
		stream<- network.await()=>{},
		line<- terminal.await()=>{},
		_ <- copy.await()=>{}
	}
})
```

select는 go의 select와 동일하다 여러 async 중에서 가장 먼저 yield하는 브랜치를 먼저 실행한다. Future의 경우 한번 await으로 yield하면 소유권이 move되어버리므로 borrow하여 사용하는 것이 좋다.

async의 오버헤드는 주로 이런 executor의 리소스를 준비하는 데에서 온다. 하지만 일반적인 멀티쓸드 실행에 들어가는 컨텍스트 스위칭에 비교했을 때, 동일 쓰레드 내에서 작업을 yield하고 다시 재개할 수 있는 async가 효율적인 부분이 있다.

### join

다음 코드는 순차적으로 실행된다.

```rust
let file1 = files[0].await;
let file2 = files[1].await;
let file3 = files[2].await;
```

중간에 몇번 yield하던지 상관없지만, file1, file2, file3가 순차적으로 실행된다는 사실은 변함이 없다.
그런데 join! 을 사용하면

```rust
let (file1,file2,file3) = join!(files[0],files[1], files[2]);
```

셋을 concurrent하게 수행하고, 어떤 future가 먼저 awaiy되는지 알 수 없다. 물론 중간에 yield는 얼마든지 할 수 있다.

try_join_all은 이터레이터를 입력받아서 동일한 순서로 결과를 반환한다. 이 때 Future의 순서가 상관없다면 `FuturesUnordered` 타입을(이터레이터) 사용하면 된다. 조금 더 효율적이고, future가 종료되는 순서대로 이터레이터를 반환한다.

그런데, tokio에서 runtime.block_on()으로 하나의 async future을 실행하면 하나의 쓰레드가 이를 실행하는데, 멀티쓰레드 환경에서 실행할 수 있는 방법은 없을까?

tokio::spawn을 통해 parent async 와 다른 별도의 future을 넘겨주면 다른 쓰레드가 이를 실행할 수 있다. 이 때, future의 라이프타임은 부모/자식 중 누가 먼저 끝날 지 알 수 없으므로 `'static` 이여야 한다.

즉 async future 자체는 concurrent하지만 (쓰레드에 yield 할 수 있으므로), top level future 하나만 있다면 결과적으로 이를 실행하는 것은 하나의 쓰레드이다. 멀티쓰레드를 활용하여 parellelism을 활용하기 위해서는 spawn을 사용해 런타임 executor에 여러개의 future를 , 마치 큐에 넣듯이 등록하고, 쓰레드 풀에서 이를 병행적으로 수행할 수 있게 하는 것이다.

Future는 CPS의 StateMachine으로 await 포인트 사이에서 공유되는 상태를 가지는 일종의 struct이다. 이 때 future의 state는 스택변수가 아니다. 만약 state이 스택에 할당된다면 여러번 중단되고 resume될 때 사라져버리기 때문이다. 따라서 heap에 할당되며, 대신, 이러한 Statemachine의 상태가 다른 future 상태머신을 포함하는 등 계속해서 커질 수 있어, 힙에 재할당 되는 mem copy가 빈번하게 일어난다.
따라서 Box::pin 등으로 스마트 포인터를 사용하거나, tokio::spawn을 사용하면 포인터를 사용해 future를 가리키도록 한다. 한번 await 하기 시작한 future는 메모리를 move할 수 없다.

async 함수는 trait에 있지 못한다. 그 이유는 앞서 말했듯 statemachine의 크기는 힙에 할당되고 얼마나 커질지 알 수 없기 때문이다. dyn trait 오브젝트를 사용하던, impl /제네릭을 사용하던, 컴파일 타임에 어떠한 async 호출의 결과가 스택에 차지하는 사이즈를 알 방법이 (아예 없지는 않으나) 없다.
따라서 `#[async trait]` 이라는 매크로를 사용해서 마킹하면, `Pin<Box<dyn Future<Output=Response>>`의 형태로 async 호출의 결과를 바꿔줌으로써 컴파일 타임에 스택 상에 고정된 크기를 갖도록 해준다. 그러나 여전히, 힙에 모든 future가 할당되고 포인터 래핑에 따른 오버헤드가 있다.

async에서 스탠다드 라이브러리의 Mutex를 사용할 수 없다. 그 이유는 mutex를 서로 다른 future가 공유하는 경우(혹은 `Arc<Mutex<T>>`)를 사용하는 경우 락이 걸린 채로 다른 future에 쓰레드를 yield해 버리면, 다른 future 에서 해당 락을 걸려 시도하는 경우 바로 데드락이 발생하고 쓰레드가 블럭되기 때문이다.
Tokio 자체에서 제공하는 mutex를 사용하면 lock이 걸린 상태에서 lock을 시도하는 경우 바로 yield한다. 물론 일반 mutex에 비교했을 때 statemachine을 구현해야 해서 훨씬 느리다. 따라서, 표준 라이브러리 Mutex를 사용하되, 해당 Mutex의 임계영역에 await point가 포함되지 않는 경우에만 사용하는 것이 좋다.

### Function/Closure

러스트의 함수 식별자는 Function Pointer가 아니라 ZST인(사이즈가 0인) function item이라는 타입이다. function item은 고유한 함수를 가리켜야 하므로 제네릭 파라미터가 있다면 구상 타입이 있어야 한다. function item은 포인터가 아니므로 구상화된 구체적 타입을 가진다. 그러나 식별자가 없는 함수 타입의 인자나 리턴값등은 전부 함수 `포인터`로 8바이트 사이즈를 가진다. function item은 function 포인터로 형변환이 가능하다.

function 포인터 `fn()`타입은 세가지 타입의 클로져로 전부 형변환이 가능한데 (`Fn, FnMut, FnOnce`) 이는 function 포인터가 실제로 라이프타임도 없고, 상태도 없는, 가장 단순한 포인터이기 때문이다. 따라서 가장 공변적이라고 생각해도 좋다. 또한 세 클로져는 Fn<Fnmut<FnOnce 순서대로 계층이 존재하는데 그 이유는 owned self 타입이 있으면 shared reference, mutable reference를 모두 만들어 낼 수 있고, mutable reference가 있으면 shared reference를 만들어낼 수 있기 때문이다. 따라서 FnOnce가 가장 공변적이다.

반대로 아무것도 캡쳐하지 않는 non-capturing closure는 가장 단순한 function pointer로 형변환할 수 있다.

Closure trait 또한 동적디스패치로 호출할 수 있지만 이 때 클로저타입의 borrow/ownership requirement를 충족해야 한다. 즉 `&dyn Fn()`은 가능하지만 `&dyn FnOnce, &dyn FnMut` 는 불가능하다.
dyn FnOnce 오브젝트는 소유권을 가질 수 있는 스마트 포인터 타입 `Box<dyn FnOnce>`등으로만 사용할 수 있다. dyn FnOnce는 자기자신 만으로는 사용할 수 없는(사이즈를 모르는) 오브젝트 이기 때문이다.

나이틀리에서 사용할 수 있는 피쳐로 const Fn trait bound가 있다. 제네릭한 함수는 const 함수(컴파일 타임에 값을 알 수 있는 함수) 가 될 수 없고 컴파일 되지 않는다. 하지만 제네릭의 trait bound로 ~const Fn / FnOnce 를 가지면 해당 제네릭이 const일 때만 이 함수가 const 라고 힌트를 줌으로써 const 함수로써 컴파일 된다.

tokio::spawn에 넘겨지는 future 클로져의 경우 스택프레임 보다 더 오래 유지되기도 하는(바로 await을 하면 그럴 필요가 없음) 경우 'static 라이프타임을 요구함. 'static 라이프타임을 요구하는 클로져의 경우 move (FnOnce)로 구현하여 해당 요구사항을 충족할 수 있음(캡쳐링하는 변수도 모두 'static으로 바꾼다)

### Send /Sync

Send/Sync는 다른 행위를 별도로 정의하지 않는 `marker` trait 이다. struct, enum , vecotr 등 compount type의 경우 내부 타입이 모두 send/sync면 자동으로 send/sync를 갖는다. `auto-trait`이다. auto-trait은 marker trait라고 보면 된다. (반대는 성립하지 않는다).
Send는 해당 값을 다른 쓰레드에 전달할 수 있다(소유권을 넘길 수 있다)는 뜻이다. 거의 모든 타입은 Send trait을 가진다. 그러나 다른 쓰레드간에 공유가 허용되지 않는 `RC, MutexGuard`등의 타입이 예외에 해당된다. Mutex 자체가 아니라 MutexGuard임에 주의하자. 이는 하나의 쓰레드에서 건 lock을 다른 쓰레드에서 해제할 수 없다는 의미이다. Mutex 자체는 Send이다. 또한 `ThreadLocal state`과 관련된 drop 구현을 가지는 타입 또한 Send의 예외이다. (다른 쓰레드에서 drop될 수 없으므로)

```toc

```

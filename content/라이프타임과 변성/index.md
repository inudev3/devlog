---
emoji:
title: 라이프타임과 변성
date: '2023-03-28 03:00:00'
author: inu
tags: Rust lifetime variance unsafe
categories: Rust 
---

### Variance

더 긴 라이프타임을 가지는 변수를 해당 타입의 더 짧은 라이프타임에 할당할 수 있다. 이는
`더 짧은 라이프타임 타입이 더 긴 라이프타임 타입의 상위 타입이기 때문이다. 즉 긴 라이프타임 , 예를 들어 &'static 은 더 짧은 라이프타입 &'a의 하위 타입 혹은 서브타입이다.`
서브타입은 적어도 부모타입 만큼의 효용이 있을 때, 공변적이라고 여겨진다. 반면 부모타임이 적어도 서브타입 만큼의 효용이 있을 때 반공변적이라고 여겨진다.

함수는 인자타입에 대해 반공변적이다. 즉 인자에 구상적인 타입을 정의한 함수는 그보다 더 추상적인 타입을 인자로 받는 함수로 대체할 수 있다.

불변 참조 `&'a T` 타입은 라이프타임과 T 타입에 대해 모두 공변적이다. 그러나 가변 참조는
`&'a mut T` 라이프타임 a에 대해서는 공변적이지만 T에 대해서는 무공변이다.

즉 가변참조 &'a mut T는  라이프타임 'a는 보다 구체적인(보다 긴) 라이프타임으로 대체할 수 있어 공변적이지만, T 타입은 T의 서브타입으로 대체할 수 없고 정확하게 T 타입만을 소비한다. 그 이유는, 가변참조는 참조를 변경할 수 있고, 이 참조는 공변적으로 참조를 변경할 수 있기 때문에 공변성을 알 수 없다.
```rust
let x :&mut T = T::new();
let x = U::new(); //안됨
```
즉 `&'static mut T`는 `&'a mut T` 에는 여전히 할당할 수 있다. 라이프타임에 있어서는 여전히 공변적이기 때문이다. 그러나 서브타입 U가 있을 때 `&'a mut U`는  `&'a mut T`에 할당할 수 없다. 물론 러스트에서 trait이 없이 구체적인 서브타입이 존재하지는 않기 때문에, 이는 실제로 사용되지 않는 예시이다.

다음과 같은 제네릭 함수가 있다고 하자
```rust
pub fn strtok<'a>(s: &'a mut &'a str, delim:char) -> &'a str
```
strtok은 의 인자 s는 포인터 &mut &str에서, 타입 &str과 포인터 타입 &mut의 라이프타임을 모두 동일학 제네릭을 가지고 있다. 이 때 타입인 &str에 대해서 무공변이므로, 인자로 주어지는 &str 타입의 라이프타입은 해당 &str에 대한 가변 포인터 &mut의 라이프타임과 동일해야만 하낟.


&mut T는 라이프타임에 대해서는 여전히 공변적이라는 사실을 기억하자. 즉 다음과 같이 적을 수 있다.
```rust
#[test]  
fn it_works() {  
	fn check_is_static(_:&'static str){}  
	let mut x = "hello world";  
	//strtok <'a,'b> (&'a mut &'b str)-> &'b str  
	let z = &mut x;  
	let hello = strtok(&mut x, ' ');  
	assert_eq!(hello, "hello");  
	assert_eq!(x, "world");  
}
```
x에 대한 가변 참조를 2번 생성했지만 컴파일 된다. 그 이유는 라이프타임이 공변적이기 때문에, 첫번째 &mut로 선선한 `z` 변수의 라이프타임이 그 다음 `&mut` 참조를 빌리기 전에 끝나는 것으로 더 짧다고 선언할 수 있기 때문이다.

`PhantomData<T>`는 러스트에서 유일하게 T에 대해 제네릭하지만 T를 소유하고 있지 않는 것이 허용되는 타입이다. 따라서 동일한 상황의 다른 구조체가 있을 경우 `PhantomData<T>`를 통해 소유하고 있지 않는 T타입에 대해 제네릭함이 허용됨을 마킹할 수 있다.
라이프타임이 공변적이기 때문에 사용되지 않는 포인터가 있다면 포인터가 가리키는 타입을 `drop` 해도 `dangling`하지 않고 컴파일된다.
```rust
fn main(){
	let x = String::new();
	let y = vec![&x]; //여기서 라이프타임이 종료됨., 더 짧은 라이프타임에 할당되도록 컴파일러가 암묵적으로 처리
	drop(x); 
}
```
y의 타입은 컴파일러에 의해 drop이 실행되기 전에 끝나는 라이프타임을 가지는 것으로 추론된다. 따라서 x에 대한 포인터가 이미 없는 것으로 가정되어 vec이 drop된다.

만약 어떤 타입이 T라는 타입에 대해서 제네릭하다고 하자, 이 타입이 `Drop` trait을 구현했다면 drop될 때, 실제 drop impl 구현에서 T타입에 대해서 접근하던/접근하지 않던, 컴파일러는 실제로 T타입을 '사용'한 것으로 간주한다. 사용했다는 것은 소유권을 넘겼다는 것이고, `drop` 과 가깝지만 엄격한 의미로는 동일하지 않다. '사용'함으로써 drop하는 예시로는 `let _ = T()` 가 있겠다. 이 경우 사용하면서 drop된다. 이는 drop 메서드가 가변참조인 `&mut self`를 받기 때문이며, 컴파일러는 이를 보수적으로 해석하여 drop 메서드 내에서 제네릭 타입 T를 사용하여 소유권이 이미 이전되었음을
`Drop` trait을 구현하지 않았다면 제네릭 T타입은 아무 일도 일어나지 않는다. 그러나 다음과 같이 unsafe와 `#[may_dangle]` 매크로를 사용하면, Drop 구현이 T타입에 대해서 아무 일도 하지 않는 다는 힌트를 컴파일러에게 주고, 컴파일러는 T타입에 대해서 drop 구현동안 접근하지 않는다는 것을 알게 된다. 다만 이것이 T타입을 drop하지 않는다는 보장은 아니다.
따라서 다음의 경우를 살펴보자.
```rust
pub struct Boks<T>{  
	p: *mut T,  
}  
impl<T> Boks<T>{  
	pub fn ny(t:T)->Self{  
		Boks{  
			p:Box::into_raw(Box::new(t))  
		}  
	}  
}  
impl <T> Drop for Boks<T>{  
	fn drop(&mut self) {  
		//SAFETY  
		unsafe { Box::from_raw(self.p)}; //creates a new Box, deallocates the previous Box and pointer  
	}  
}
```
제네릭한 T타입으로 raw 포인터를 가지는 Boks라는 스마트 포인터 타입을 정의했다. Drop을 구현했지만, drop 내부에서 T타입에 대해서 어떤 접근도 하지 않는다. 물론 T타입을 drop하고 있기는 하다.

다음과 같은 코드를 살펴보자.
```rust
fn main() {  
	let x = 42;  
	let b = Boks::ny(x);  
	println!("{:?}", *b);  
	let mut y = 42;  
	let b = Boks::ny( &mut y); //b is now a raw pointer to a mut ref  
	println!("{:?}", y);  
	// drop(b);  
}
```

b 변수는 y에 대한 가변참조를 가지는 Boks 스마트 포인터를 가지고 있다. 이 때 스마트 포인터는 raw pointer를 사용하기 때문에, 라이프타임이 없는 raw pointer는 reference 타입 처럼 컴파일러가 자동으로 `lifetime` 변성을 사용해 포인터를 drop할 수 없다. 따라서 명시적으로 drop이 이루어져야 하는데, drop이 일어나지 않는 `&mut T`가 존재하고, 컴파일러는 drop 이 구현될 경우 제네릭 &mut T가 기본적으로 *사용된다*고 가정하기 때문에, &mut T의 라이프타임을 변성으로 단축시킬 수가 없는 것이다. 즉 라이프타임 변성이 drop 구현시에는 적용될 수 없어 가변참조와 불변참조가 동시에 접근하려고 할 때 충돌이 일어나는 것이다. 반면 Drop 구현이 없다면 컴파일러는 제네릭 &mut T가 사용되지 않는다고 가정하고, 라이프타임 변성을 사용해 불변참조가 사용되기 전에 포인터를 drop해버린다.
drop 구현을 하면서 제네릭 타입을 사용하지 않는 다는 힌트를 컴파일러에게 준다면 컴파일러가 알아서 T타입에 대해서 라이프타임 변성을 추론하도록 할 수 있다. `#[may_dangle]` 등이 해당 된다.

```toc
```
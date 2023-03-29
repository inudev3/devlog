---
emoji:
title: Rust를 알아보자 - 2부
date: '2023-03-29 03:00:00'
author: inu
tags: Rust
categories: Rust
---

### 제네릭

러스트에서 제네릭 함수는 `컴파일 타임` 에 타입을 구체화 한다. 런타임에 타입을 구체화하는 타입 소거와는 다르다.

제네릭의 upperbound로써 Trait을 사용하는 것은 매우 자주 있는 일이다. 이는 해당 타입이 해당 Trait을 구현했다는 것을 말하며, 제네릭 함수 내에서 Trait의 메서드를 사용할 수 있게 해준다.
```rust
use std::fmt::Display; 
fn print_number<T:Display>(number:T){
	println!("{}", number); //{}로 포맷하기 위해서 Display Trait을 impl 해야 함
}
```
모든 제네릭 타입 T가 Display를 구현했다는 제약 upperbound가 있기 때문에 함수 내에서 `{}`로 포매팅이 가능하다.
하나의 upperBound에 여러 조건을 명시할 수도 있다. 이 경우 + syntax로 Trait들을 연결한다.
```rust
fn compare_and_display<T:Display+Debug>(stmt:T)
```
where 문법을 사용하면 타입 파라미터의 제약조건을 분리할 수 있다.
```rust
fn compare_and_display<T,U>(stmt:T, num:U) where T:Display, U:Display+PartialOrd
```

rust의 panick은 런타임 익셉션이다.
`Option`은 `enum`으로써 제네릭하며 자바의 `Optional`과 동일하다.
```rust
/*
pub enum Option<T>{
	None,
	Some(T)
}예시, 실제 구현은 다름
*/
fn take_fifth(value:Vec<i32>)->Option<i32>{
	if value.len()<5{
		None
	}else{
		Some(value[4])
	}
}
fn main(){
	let new_vec = vec![1,2];
	let bigger_vec = vec![1,2,3,4,5];
	println!("{:?}, {:?}", take_fifth(new_vec), take_fifth(bigger_vec));
	//None,Some(5)
}
```
Option의 값을 꺼내는 메서드는 `unwrap()`이다. Optional의 get과 동일하다고 보면 된다. unwrap할 option이 None이면 러스트는 런타임익셉션(패닉이라고 부름)을 발생시킨다. orElse, orElseGet과 동일하게 unwrap_or_else, unwrap_or_default 등의 메서드가 있다. expect()는 unwrap() 대신에 사용할 수 있는 메서드로 런타임에 패닉하게 되면 expect에 인자로 전달한 문자열을 출력한다.
`isSome()`, `isNone()` 을 사용해 unwrap 되지 않은 상태에서 어떤 enum인지 확인해볼 수 있다.
Vec에서 `[]` 대신 `get(index)`를 사용해 인덱스 원소를 가져올 수 있는데 이 때 Option에 이를 래핑해서 반환한다.

### Result
Option과 비슷하게 제네릭한 enum으로 다음과 같다
```rust
pub enum Result<T,E>{
	Ok(T),
	Err(E)
}
```
성공시 T, 에러시 E타입을 래핑하며 반환한다. T와 E는 전부 Unit이 될 수 있기 때문에, 비어있는 Ok와 Err를 반환할 수도 있다. Option과 마찬가지로 unwrap()으로 타입을 꺼낼 수 있으며, Err를 unwrap()하는 경우 런타임 패닉한다. 마찬가지로 `isOk(), isErorr()` 함수가 존재한다.

### turbofish

제네릭함수를 호출할 때 타입 파라미터와 같이 호출하는 syntax를 `turbofish`라고 한다. 물고기를 닮았다고 해서 붙여진 이름이다.

```rust
use std::num::ParseIntError;
fn return_number(input:&str)->Result<i32,ParseIntError>{
	input.parse::<i32>()
}
fn main(){
	let my_vec = vec!["1", "two", "3ree", "4"];
	for num in my_vec{
		match return_number(num){
			Ok(num)=> println!("{}", num),
			Err(err)=> println!("{}", err),
		} //1, parseinterror, parseinterror, 4
	}
}
```

```rust
fn main(){
	let my_vec = vec![1,2,3,4];
	let get_one = my_vec.get(0);
	let get_ten = my_vec.get(10);//Option을 반환한다.
	for index in 0..10{
		match my_vec.get(index){
			Some(number)=>println!("{}", number),
			None => {}
		}
	}
}
```
이렇게 match에서 다른 가지는 신경쓰지 않을 경우가 있다.
이럴 때는 `if let` syntax를 사용하자
```rust
fn main(){
	let my_vec = vec![1,2,3,4];
	let get_one = my_vec.get(0);
	let get_ten = my_vec.get(10);//Option을 반환한다.
	for index in 0..10{
		if let Some(number) = my_vec.get(index){
			println!("The number is:{}", number);
		}
	}
}
```
if let은 조금은 어색하지만, 구조 분해할당과 비슷하다. match 구문에서 가지에 변수를 정의한 것과 동일하다고 볼 수 있는데, match와 다른 점은 하나의 가지만 적용하고 나머지는 무시된다는 점이다. if let을 통해 변수를 할당해 만들어낸다고 보면 된다.
이것이 가능한 이유는 let 부터가 변수 선언만을 위해 사용하는 것이 아니라
`let 패턴 : 타입= 표현식` 의 문법이 항상 동작하는 패턴이기 때문이다. 이를 `irrefutable pattern` 이라고 한다. 그말인 즉슨 let 자체적으로 항상 분해할당이 가능하기 때문인데, if let의 경우 패턴의 타입이 표현식과 일치하는지 일치하지 않는지의 여부에 따라 let이 `boolean`을 반환하기 때문에 가능하다
따라서 `if let` 뿐만 아니라 `while let` 또한 가능하다.
```rust
fn main(){
	let weather_vec = vec![
		vec!["Berlin", "cloudy", "5","-7", "77"],
		vec!["Athens", "sunny","not humid","20", "10", "50"]
	]
	for mut city in weather_vec{
		println!("for the city of :{}", city[0]);
		while let Some(information) = city.pop(){
			if let Ok(number) = information.parse::<i32>(){
				println("the temp is:{}", number)
			}else{
				println!("{}", information)
			}
		}
	}

}
```

### HashMap

std::collections에 컬렉션 자료구조가 있다. 맵에는 HashMap과 BTreeMap이 대표적이다.
Hashmap은 순서가 유지되지 않으며  이터레이터가 (k,v) 튜플 형태를 반환한다.
hashmap의  값을 키로 get할 때는 항상 borrowd 타입을 사용해야 한다. 즉 포인터를 인자로 넘긴다.
```rust
let mut book_map = HashMap::new();
book_map.insert(1, "이름");
book_map.get(&1)
```
일반적인 문자열 리터럴은 &str 타입이므로 그대로 전달해도 된다.

hashmap은 `entry(key)` 메서드를 가지는데 entry가 반환하는 `Entry`는 enum으로, `OccupiedEntry`와 `VacantEntry` 두 개의 변수를 갖는다.  Entry는 매우 편리한 `or_insert()` 메서드를 제공하는데 VacantEntry의 경우에 value를 삽입하고 `&mut`  참조를 반환한다. or_insert 외에도 or_default 등이 있다.
```rust
let mut letters = HashMap::new();  
for ch in "a letter in this sentence".chars(){  
    let counter = letters.entry(ch).or_insert(0);  
    *counter+=1;  
    println!("{}, {}", ch,counter);  
}
```
mut 참조를 반환하므로 counter를 직접 증가시켜야 한다.
mut 참조를 사용하는 점을 사용하면 다음과 같이 일종의 그룹핑을 할 수 있다.
```rust
let data = vec![  
    ("male", 0),  
    ("female", 1),  
    ("male", 10),  
    ("female", 7),  
    ("female",9),  
    ("male",5)
];  
let mut survey_hash = HashMap::new();  
for item in data{  
    survey_hash.entry(item.0).or_insert(Vec::new()).push(item.1);
    //&mut를 반환하므로 벡터에 값을 푸쉬하여 리스트를 만든다.  
}
```
or_insert는 VacantEntry일 때만 값을 삽입하지만 항상 값에 대한 &mut 참조를 반환한다는 사실을 기억하자.

### HashSet
HashSet의 구현은 HashMap 의 value가 `()` 타입인 것으로 구현된다.
```rust
use std::collections::HashSet;

fn main() {
    let many_numbers = vec![
        94, 42, 59, 64, 32, 22, 38, 5, 59, 49, 15, 89, 74, 29, 14, 68, 82, 80, 56, 41, 36, 81, 66,
        51, 58, 34, 59, 44, 19, 93, 28, 33, 18, 46, 61, 76, 14, 87, 84, 73, 71, 29, 94, 10, 35, 20,
        35, 80, 8, 43, 79, 25, 60, 26, 11, 37, 94, 32, 90, 51, 11, 28, 76, 16, 63, 95, 13, 60, 59,
        96, 95, 55, 92, 28, 3, 17, 91, 36, 20, 24, 0, 86, 82, 58, 93, 68, 54, 80, 56, 22, 67, 82,
        58, 64, 80, 16, 61, 57, 14, 11];

    let mut number_hashset = HashSet::new();

    for number in many_numbers {
        number_hashset.insert(number);
    }

    let hashset_length = number_hashset.len(); // The length tells us how many numbers are in it
    println!("There are {} unique numbers, so we are missing {}.", hashset_length, 100 - hashset_length);

    // Let's see what numbers we are missing
    let mut missing_vec = vec![];
    for number in 0..100 {
        if number_hashset.get(&number).is_none() { // If .get() returns None,
            missing_vec.push(number);
        }
    }

    print!("It does not contain: ");
    for number in missing_vec {
        print!("{} ", number);
    }
}

```

### BTreeSet

BTreeSet은 HashSet과 거의 동일하지만 삽입 시 원소가 정렬된다.

### BTreeMap

BTreeMap은 HashMap과 거의 동일하지만 삽입 시에 원소가 정렬된다.

### BinaryHeap

heap 구조를 가지는 우선순위 큐이다.  러스트의 BinaryHeap은 최대힙으로 크기가 큰 원소부터 pop된다.
```rust
//789 100 59,33, 2, -7  
}  
fn show_remainder(input:&BinaryHeap<i32>)->Vec<i32>{  
    let mut remainder_vec = vec![];  
    for number in input{  
        remainder_vec.push(*number);  
    }  
    return remainder_vec  
}
fn main(){
	let mut binary_heap = BinaryHeap::new();  
	binary_heap.push(100);  
	binary_heap.push(790);  
	binary_heap.push(2);  
	binary_heap.push(33);  
	binary_heap.push(59);  
	binary_heap.push(-7);  
	while let Some(number) = binary_heap.pop(){  
	    println!("{}", number);  
    }
	 let my_numbers = vec![1,2,3,4,5,6];  
	let mut my_heap = BinaryHeap::new();  
	for number in my_numbers{  
	    my_heap.push(number);  
	}  
	while let Some(number) = my_heap.pop(){  
	    println!("popped:{}, remaining:{:?}", number, show_remainder(&my_heap))  
	}
}

```
BinaryHeap 또한 이터레이터를 사용해 반복문을 돌 수 있다.

### VecDeque

Vec의 경우 뒤가 아니라 앞에서 `remove`를 하면 모든 원소를 한 칸씩 앞으로 옮기므로 O(n)이 소요된다.
VecDeque은 ring buffer를 사용하는 큐 구조로 `pop_front, push_back`을 O(1)에 수행한다.
vec으로 부터 VecDeque을 만들 수 있는 from `associated function`이 있다.
```rust
let mut my_deque = VecDeque::from(vec![0;600_000]);
for(i in 600_000){
	my_deque.pop_front();
}
```


### `?` 연산자

기존의 `try!` 매크로였던 것이 rust의 버전이 업데이트 되면서 ? 연산자로 변경되었다. ? 연산자는 Result와 Option에 대해서 동작하는 연산자로 kotlin의 `?:` (엘비스 연산자)와 약간 비슷하다.
Option과 Result를 unwrap하고, 만약 None 또는 Err 라면 루틴을 종료시키고, Ok또는 Some이라면 값을 정상적으로 반환한다.
```rust
use std::num::ParseIntError;

fn parse_str(input: &str) -> Result<i32, ParseIntError> {
    let parsed_number = input.parse::<i32>()?; // Here is the question mark
    Ok(parsed_number)
}
```
Err라면 그대로 parse_str이 err를 반환하고, 아니라면 parsed_number에 값을 unwrap한다. Err가 아니여야 Ok가 리턴된다.
?를 체이닝하는 것도  허용된다. 하지만 Err의 에러타입이 다를 경우에 함수 시그니쳐에 주의하자
```rust
use std::num::ParseIntError;

fn parse_str(input: &str) -> Result<i32, ParseIntError> {
    let parsed_number = input.parse::<u16>()?.to_string().parse::<u32>()?.to_string().parse::<i32>()?; // Add a ? each time to check and pass it on
    Ok(parsed_number)
}

fn main() {
    let str_vec = vec!["Seven", "8", "9.0", "nice", "6060"];
    for item in str_vec {
        let parsed = parse_str(item);
        println!("{:?}", parsed);
    }
}

```

### panic!, assert!

런타임에 패닉을 일으킬 수 있는 매크로들로, 도메인의 데이터나 함수에 대한 불변식 `invariant`를 검사하는 데에 유용하다.

assert는 boolean 조건식을 검증하며 이름그대로의 조건식을 2개의 인자의 적용하는 `assert_eq, assert_ne` 등의 버전도 있다.

### Trait

Trait은 Interface와 같다. `trait` 키워드로 정의한다. 이전에 Trait을 구현하기 위해 추가했던
`#[derive(Trait)]` 은 Trait의 구현을 자동화해주는 `attribute` 이다. 하지만, `attribute` 로 구현을 자동화할 수 있는 경우는 매우 공통적으로 사용되는 Display, Clone  등 몇가지 Trait에 한해서 attribute를 통해 자동으로`impl`이 구현되며 대부분은 이를 직접 impl해야 한다.
```rust
struct Animal { // A simple struct - an Animal only has a name
    name: String,
}

trait Dog { // The dog trait gives some functionality
    fn bark(&self) { // It can bark
        println!("Woof woof!");
    }
    fn run(&self) { // and it can run
        println!("The dog is running!");
    }
}

impl Dog for Animal {} // Now Animal has the trait Dog

fn main() {
    let rover = Animal {
        name: "Rover".to_string(),
    };

    rover.bark(); // Now Animal can use bark()
    rover.run();  // and it can use run()
}

```
struct에 trait을 추가하려면 `impl {Trait} for {struct}` 문법을 사용해야 한다. Trait이 디폴트로 메서드를 구현한 경우 구현하지 않아도 상관없지만, Trait이 시그니쳐만 명시한 경우 이를 반드시 오버라이드 해야 한다.
Trait의 이름은 동사/형용사가 적합하다.


```toc
```
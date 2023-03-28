---
emoji:
title: Rust를 알아보자 - 1부
date: '2023-03-28 03:00:00'
author: inu
tags: Rust
categories: Rust
---

## Rust를 알아보자 -1부
rust는 유니코드를 지원 (다국어 가능)


### 원시자료형

정수형
1. signed
   i8,i16,i32,i64,i128, isize
   isize는 유저 컴퓨터 운영체제의 비트 수를 의미(64비트 or32비트)
   변수 타입을 지정하지 않은 정수는 기본적으로 i32로 할당
2. unsigned
   u8,u16,u32,u64,u128,usize

Rust에서 char 타입으로 변환될 수 있는 정수는 u8 이 유일함.

String::len() 함수는 문자열 길이가 아니라 *바이트 크기*를 반환함
char는 항상 4바이트이지만 문자열은 그 이하일 수도 있음(최소 1바이트)
문자열의 길이를 알려면 String.chars().count()를 호출해야 함.

정수형의 타입을 명시 할 때 다음과 같이 적을수도있음
```rust
let mynum = 10u8;
let mynum2 = 120u32;
let mynum3 = 120_u32; //underscore가 몇개든 상관없음
let bignum = 120_930_330_u128;
```
크기가 다른 정수형(u8, u128) 끼리는 서로 연산할 수 없음

다음의 경우에 rust는 타입을 추론해줌
```rust
let my_float = 5.0;
let my_other_float: f32 = 5.0;
let third = my_float+my_other_float;
```
my_float과 my_other_float의 Add연산에서 매개변수가 f32이므로 my_float도 f32으로 타입이 추론됨.

러스트에서 코드블록은 함수의 의미를 가지지만 아무 키워드 없이 블록만 있어도 컴파일이 된다. 블록은 리턴값으로 평가되며 다음과 같은 코드도 가능하다.
```rust
fn main(){
	let my_number = {
		let second_number =9;
		second_number //리턴값 9
	};
	println!("{}", my_number); //9
}
```
변수의 스코프는 선언된 블록까지이다.
아무것도 리턴하지 않는 함수/블록의 값은 `unit` (러스트에서는 `()` 로 표현)으로 평가된다.

println, format 매크로에서 `"{}"`  로 포맷될 수 있는 변수는 `std::fmt::Display`라는 `trait` 을 구현해야하며 (인터페이스) 만약 이 인터페이스(`trait`)를 구현하지 않은 경우, `"{:?}"`  로 Debug printing을 하거나 `"{:#?}"` 로 pretty printing을 해야 한다.

### 가변과 불변

rust의 변수 선언인 `let` 은  불변 변수이다. 가변 변수를 선언하고 싶다면 `let mut`으로 선언해야 한다.
```rust
let mut my_number=8;
my_number=10;
```
동일한 변수를 중복해서 선언하더라도 에러는 발생하지 않으며 단지 변수가 섀도잉된다.
```rust
let mut my_number=8;
println!("{}", my_number);//8
let my_number = "hi";
println!("{}", my_number);//hi
```

따라서 rust에서는 변수명을 새로 지을 필요 없이 새로 선언하기만하면 된다
```rust
let x = 8;
let x = x+8; //16
let x = x/8; //2
```

### 스택, 힙, 포인터
스택이 힙보다 *항상* 빠르다.
러스트의 변수는 항상 컴파일 타임에 변수 크기를 알아야 스택에 변수 메모리를 적재한다. 타입을 명시하지 않더라도 대부분은 타입을 추론할 수가 있다. 그런데 타입을 컴파일 타임에 추론할 수 없는 경우가 존재한다. 이 경우 스택에는 포인터만을 적재한 뒤 힙 메모리에 해당 변수를 추가한다. 힙은 동적이므로 크기와 상관없이 추가할 수 있다. 포인터의 사이즈는 항상 고정되어있기 때문에 스택에 적재할 때 문제가 없다.
포인터는 일종의 인덱스이다. 포인터의 레퍼런스(참조)는 몇번이든 중첩될 수 있다.
포인터는 몇번 중첩됐는지에 따라 모두 다른 타입이다. 1번 참조한 포인터와 5번 참조한 포인터는 다른 타입이다.

```rust
fn main(){
	let my_number = 8;
	let my_reference = &my_number;
	println!("{}", *my_reference==my_number);
}
```

### Printing
println은 새로운 줄을 띄어쓸 수는 있지만 들여쓰기는 전부 빈칸으로 취급한다. \ 를 통해 이스케이프 기능을 사용하지만, 다음과 같이 사용할 경우 \ 를 없이 문자를 이스케이프할 수 있다.
```rust
fn main(){
	println!("He said, \"You can find it at c:\\files\\my_documents.\". I said, \"Thank you\". ")
	println!(r#"He said, "You can find it at c:\files\my_documents.". I said, "Thank you". "#)
}
```
r#은 이스케이프처리 없이도 특수문자들을 출력할 수 있게 해준다.
그러나 \#문자에 대해서는  r# 또한 이스케이프가 불가능하다. 문자열에 \#이 들어가는 경우에 한해서 문자열에 포함되는 \#한개당 r#으로 시작하는 \#을 하나 추가해줘야 이를 이스케이프할 수 있다.
문자열 앞에 b를 붙이면 문자열이 바이트로 출력된다. 이경우 `{:?}` 또는 `{:#?}` 로 출력해야 한다.
`{:X}` 는 16진수, `{:p}` 는 포인터 주소 출력, `{:b}` 는 바이너리이다. c printf 형식 지정자와 동일하다.

println으로 포맷을 출력할 때는 인자로 받는 문자열의 순서를 원하는대로 조정할 수 있다.
```rust
fun main(){
	let father_name = "Vlad";
	let son_name = "Adrian Fahrenheit";
	let family_name = "Tepes";
	println!("This is {1} {2}, son of {0} {2}", father_name, son_name, family_name); 
}
//This is Adrian Fahrenheit Tepes, son of Vlad Tepes
```
혹은 named argument도 가능하다
```rust
fun main(){
	println!("{city1} is in {country} and {city2} is also in {country}, but {city3} is not in {country}", city1="seoul", city2="Busan", city3="Tokyo", city4="Korea")
}
```
매우 복잡한 포맷팅도 가능하다
`{variable:padding alignment minimum.maximum}` 의 형태로 포맷한다.
`println!("{:-^11}", "hello")`의 결과는 `---hello---`이다.

rust에는 String과 &str이 있으며
다음과 같이 리터럴로 선언시 타입은 &str이다.
```rust
fn main(){
	let my_variable = "Hello, world!"; //&str
}
```

&str은 컴파일 타임에 문자열 데이터의 크기를 알 수가 없어 스택에서 사이즈를 알 수 있는 포인터로 참조하는 것이다.
String을 만들기 위해서는 문자열 리터럴이 아닌 몇가지 함수를 사용한다.
```rust
fn main(){
	let my_string = String::from("Hello, world!");
	let my_string = "Hello, world!".to_string();
}
```
String은 추가적인 함수를 지원하며 조금 더 느리고 복잡한 자료형이다. String은 항상 24바이트 고정크기를 갖는다. &str은 항상 가변크기를 가지고 크기가 고정되지 않는다.(데이터가 힙에 있으므로)
&str은 문자열 힙데이터에 대한 소유권을 *빌려* 온 것이며, String은 문자열 힙 데이터에 대한 완전한 소유권을 가지고 잇다.

`format!` 매크로는 println과 인자가 동일하지만 출력하는 대신 포맷한 문자열을 반환한다.
```rust
let my_string = format!("{}, {}!", "hello", "world") //hello, world!
```

또다른 문자열을 만드는 방법으로는 `.into()`가 있다. into는 `not-owned` 데이터를 `owned` 타입으로 변환한다. 다만, &str의 owned 타입은 `String` 말고도 몇가지가 더 있으므로 타입추론이 불가능하다. 타입을 명시하자
```rust
let my_string:String = "hello, world!".into();
```


### Const와 Static

기본적으로 `let` 변수는 불변이지만 더 확실하게 하기 위해서는 `const` 와 `static` 을 사용할 수 있다.

const와 static은 타입 추론도 불가능하며 타입을 명시해야 한다.
```rust
const MY_NUMBER:i32 = 8;
```
또한 const는 소문자의 경우 컴파일러가 경고한다(대문자가 컨벤션이다) 섀도잉이 불가능하다.

배열 타입의 경우 크기가 다르면 다른 타입이다.
```rust
static SEASONS:[&str; 4] = ["Spring", "Summer", "Fall", "Winter"];
```
`Vec`  타입은 배열이면서 크기가 동적으로 결정된다.

### Ownership

rust는 항상 하나의 변수에 하나의 owner만 있을 것을 권장한다.
```rust
fn return_str()->&'static str{
	let country = String::from("hello, world!");
	let country_ref = &country
	return country_ref
}
fn main(){
	let country_name = return_str();//에러
}
```
위 코드는 에러가 발생하는데, 이유는 이렇다.
country 변수 소유권은 함수 `return_str`에게 있다. 그런데, return_str함수는 country의 포인터를 반환하는데 country변수는 함수가 종료되면 스택에서 삭제되고, 따라서 이를 참조를 다시 변수에 대입하여 사용하려고 하면 에러가 발생한다.
정상적으로 소유권을 넘겨주려면 소유한 변수를 참조하는 것이 아니라 그대로 리턴해야 한다.
```rust
fn return_str()->String{
	let country = String::from("hello, world!");
	country
}
fn main(){
	let country_name = return_str();
}
```
함수가 소유한 변수는 함수가 리턴되면서 country_name 변수로 소유권이 이전되었다.

### 가변 참조
&mut로 표현된다. 가변 참조는 가변 변수의 주소값만 참조할 수 있다.
```rust
fn main(){
	let mut my_number =8;
	let num_ref = &mut my_number;
}
```
즉 &은 불변변수에, &mut는 가변변수에만 사용할 수 있다.
똑같이 역참조 기능을 사용한 뒤에 값을 변경할 수 있다.

```rust
*num_ref+=10;
println!("{}", num_ref); //18
```
메모리 값을 변경할 권한을 노출하기 때문에 동시성 문제가 발생한다.
따라서 반드시 하나의 가변 변수에 대한 가변 참조는 **하나만 존재해야 한다**
불변 변수에 대한 참조는 몇십개, 몇만개가 존재해도 상관없다. 값을 변경할 수 없기 때문에 동시성 문제가 발생하지도 않기 때문이다. 당연한 얘기로, 불변참조와 가변참조가 동시에 존재할 수도 없다. 가변참조가 값을 변경하면 불변참조가 어느 시점의 값을 읽는지 알 수가 없다. 다행히 rust 컴파일러는 가변참조가 값을 변경하기 전에 또다른 불변참조가 선언되는 것을 막아준다. 하지만 이를
가변 참조가 하나만 존재하도록 관리하는 것은 중요한 문제이다.

### 함수와 소유권

rust에서 모든 `값` 소유자(`owner`)가 필요하다. 함수에 인자로 전달되는 값은 해당 함수가 소유권을 가지게 되고, 함수가 종료되면서 소유권을 또다시 이전하지 않는다면 (반환하여 다른 변수에 ownership 이전) 해당 값은 소유자가 없어지게 된다. 소유자가 없어진 변수는 곧 가비지 컬렉션이 대상이 되고, 참조할 수 없다.

```rust
fn print_country(country_name:String){
	println!("{}", country_name);
}
fn main(){
	let country = String::from("Austria");
	print_country(country);
	print_country(country);//에러
}
```
위 코드에서, 첫번째 print_country함수가 종료되면 country 변수는 이미 가비지 컬렉션의 대상이 되어 다시 참조할 수 없다. 소유권을 이전하지 않았기 때문이다.  두번째 함수호출은 에러를 발생시킨다.
**함수는 매개변수 값의 소유권을 가진다**  (참조 아님)
이를 해결하려면 값 대신 참조를 함수에 전달하면 된다. 참조는 소유권을 변수로부터 *빌려*(`borrow`)온다. 참조 자체에는 소유권이 없다.
```rust
fn print_country(country_name:&String){
	println!("{}", country_name);
}
fn main(){
	let country = String::from("Austria");
	print_country(&country);
	print_country(&country);
}
```
소유권을 *빌려*오기 때문에 , 함수에게 소유권이 이전되지 않는다.

함수에게 가변 참조를 인자로 전달하면 변수는 변경된다.
```rust
fn add_and_print_hungary(country_name:&mut String){
	country_name.push_str("-Hungary");
	println!("{}", country_name);
}
fn main(){
	let mut country = String::from("Austria");
	add_and_print_hungary(&mut country); //Austria-Hungary
	println!("{}", country); //Austria-Hungary
}
```

문제는 다음과 같이 인자로 가변 인자를 받는 것도 가능하다는 점이다.
```rust
fn adds_hungary(mut country:String){
	country.push_str("-Hungary");
	println!("{}", country);
}
fn main(){
	let country =String::from("Austria"); //불변 변수이다.
	adds_hungary(country); //Austria-Hungary	
}
```
main 함수는 정상적으로 실행된다. country 변수의 소유권은 adds_hungary 함수에게 이전되었다. adds_hungary 함수는 자신이 소유권을 가진 매개변수를 가변/불변을 원하는대로 선언할 수 있다. 즉 함수가 호출된 스코프에서의 가변/불변과 관계없이 함수가 매개변수 인자를 다시 가변/불변성을 원하는대로 부여할 수 있다는 점이다. 매개변수 인자의 소유권은 100% 함수에게 있기 때문이다.
다만 이 경우 참조가 아니라 소유권이 이전됐으므로 해당 변수는 메모리가 해제된다.

지금까지 살펴본 소유권이 전달되는 변수들, 예를 들어 String 타입의 변수들은 데이터가 힙에 데이터가 존재하고 해당 메모리에 대한 소유권을 함수에 같이 이전한다.
하지만 만약, 데이터가 스택에서 할당될만큼 충분히 작은 크기라면 `Copy` Trait (일종의 인터페이스)을 구현하여 함수에 전달될 때 값이 복사되어 전달된다.  이 경우 변수의 소유권은 함수와는 전혀 무관하다.
```rust
fn prints_number(num:i32){
	println!("{}", num)
}
fn main(){
	let num = 8;
	prints_number(num); //8
	prints_number(num); //8
}
```
num 변수의 소유권은 함수 호출에 전혀 영향받지 않는다.
다만, Copy Trait을 구현하지 않았더라도 Clone Trait을 구현한 크기가 큰 타입들의 경우 clone() 메서드를 사용해 소유권 이전을 막을 수 있다.
```rust
fn print_country(country_name:String){
	println!("{}", country_name);
}
fn main(){
	let country = String::from("Austria");
	print_country(country.clone());
	print_country(country.clone());
}
```

### 변수 선언과 초기화

초기화 하지 않고 선언만 할 수도 있다.
그러나 초기화하지 않은 변수를 사용하려고 하면 에러가 발생한다.

### 배열
배열의 타입은 다음과 같이 기술한다.
```rust
fn main(){
	let my_array:[&str;2] = ["Mon", "Tue"];
}
```
배열의 타입은 `[원소타입;크기]` 이며, 크기가 다른 모든 배열은 전부 타입이 다르다.

다음과 같은 리터럴을 허용한다.
```rust
let my_array = ["a";40]; //크기 40의 원소가 전부 "a"
```

배열을 슬라이싱 할 수 있는 리터럴은 다음과 같다.
```rust
let three_to_five = &my_array[2..5];
let start_at_two = &my_array[1..];
let end_at_five = &my_array[..5];
let everything = &my_array[..];
```
배열에 대한 sliced view가 제공된다.

### Vec
크기가 동적으로 결정되는 `Vec` 타입을 사용하면 많은 편의함수가 제공된다. 배열보다는 살짝 느리지만 여전히 빠르다.
```rust
fn main(){
	let name1 = String::from("no.1");
	let name2 = String::from("no.2");
	let name3 = String::from("no.3");
	let mut my_vec = Vec::new();
	my_vec.push(name1);
	my_vec.push(name2);
	my_vec.push(name3);
	println!("{:?}", my_vec);
}
```
주의할 점은 , Vec 타입은 가변으로 선언되어야 push, pop 등의 배열 크기를 변경하는 연산이 가능하다는 점이다. 불변 Vec 타입은 초기화 시 선언한 원소 외에는 추가하거나 삭제할 수 없다.
Vec은 초기화를 편하게 해주는 매크로도 제공한다.
```rust
fn main(){
	let my_vec:Vec<String> = Vec::new();
	let my_vec = vec![7,8,9,10];
	let my_vec_sli = &my_vec[1..];
}
```
배열과 동일하게 slicing 할 수 있다.

vec에는 최대 길이인 용량(`capacity`) 가 있다.
```rust
fn main(){
	let mut my_vec:Vec<Char> = Vec::new();
	println("{}", my_vec.capacity());//0
	my_vec.push('a');
	println("{}", my_vec.capacity());//4
	my_vec.push('a');
	my_vec.push('a');
	my_vec.push('a');
	my_vec.push('a');
	println("{}", my_vec.capacity());//8
}
```
capacity는 동적으로 변경되며 2배씩 증가한다.

배열 리터럴 선언, explicit type과 `into()` 소유권 부여 메소드를 통해 Vec을 선언할 수도 있다. Vec은 힙에 할당되며 따라서 소유권을 필요로 한다.  타입추론을 사용할 때는 Vec 제네릭에 `_` 를 사용하면 된다
```rust
fn main(){
	let my_vec:Vec<u8> = [1,2,3].into();
	let my_vec:Vec<_> = [2,3,4].into();
}
```

### 튜플
튜플의 특징은 여러 개의 타입을 가질 수 있다는 점이다. 그리고 개수와 타입이 다른 모든 튜플은 전부 다른 타입이다.
튜플에서 인덱스로 아이템을 꺼내오기 위한 리터럴은 배열처럼 `[]`가 아니라 `.`을 찍어서 접근한다.
```rust
fn main(){
	let random_tuple = (1,2, "hi there", [2,3], !vec[1,5,9]);
	println!("{}",random_tuple.0); //1
}
```
튜플은 구조 분해할 수 있다
```rust
fn main(){
	let (a,b,c,d) = (1,2,3,[1,2,3]);
}
```

### match
match의 else는 `_`  이며 함수의 리턴 화살표와는 달리 match는 => (double arrow)를 사용한다. match는 식으로 평가되기 때문에 변수에 대입할 수 있다.
```rust
fn main(){
	let my_number: u8 = 5;
	let something = match my_number{
		8=>10,
		9=>11,
		_=>0
	}
}
```
튜플도 매치할 수 있는데, 가지는 튜플만 가능하다.
```rust
fn main(){
	let sky = "cloudy";  
	let temp = "warm";  
	let score = match (sky,temp){  
	    ("cloudy", "warm")=> "good",  
	    ("sunny", "cold")=>"bad",  
	    _=>"soso"  
	};
}
```
참고로 이런 문법을 사용할 수 있다.
```rust

fn main(){
	let sky = "cloudy";  
	let temp = "warm";  
	let score = match (sky,temp){  
	    (sky,temp) if temp=="warm" => "good",
	    (sky, temp) if sky=="cloud" && temp=="cold"=>"bad",  
	    _=>"soso"  
	};
}
```

튜플의 구조 분해 중 무시하고 싶은 원소는 _ 를 사용한다.
```rust
fn match_colours(rgb:(i32,i32,i32))->String{
	return match rgb{
		(r,_,_)if r<10 => "Not much red",
		(_,g,_) if g<10 => "Not much green",
		(_,_, b) if b<10 => "Not much blue",
		_=>"Each color at least ten"
	}
}
```

if else 문 또한  식으로 평가 되며, 변수에 할당하거나 반환할 수 있다. 식으로 평가되는 경우에는 리턴문처럼 세미콜론을 생략해야 한다.
`match` 및 `if else` 의 가지의 타입이 일치하지 않는다면 컴파일 에러가 발생한다.
```rust
let my_num = 10;
let my_var = if(my_num==10) {8} else {7} 
```
`match` 문 내부에서 변수를 선언하여 사용하려면 `@` 리터럴을 사용한다.
```rust
fn match_number(input:i32){
	match input{
		number @ 4=> println!("{}, {}*2={}", number,number,number*2),
		_=> (),
	}
}
```
브랜치를 무시하려면 `Unit`을 의미하는 `()`를 사용한다.

### Struct

struct 선언은 `struct` 키워드로 시작한다. 이름을 제외하고는 선언에 필요한 별도의 리터럴은 없다. 즉 아무것도 없는 껍데기도 선언할 수 있다.
```rust
struct MyType;
```
`()` 리터럴을 사용하면 별도의 필드가 없고 튜플처럼 원소를 가지는 `Tuple Struct` 를 선언할 수 있다

```rust
struct Colour(u8,u8,u8); //타입만 명시한다
```
`named struct` 는 이름이 붙는 필드를 가지는 타입으로 `{}` 리터럴을 사용한다. 코드 블럭은 세미콜론이 필요없다.
```rust
struct SizeAndColor{
	size:u32,
	color:Colour
}
fn main(){
	let my_color = Colour(0,50,0);
	let size_and_color = SizeAndColor{
		size:150,
		color:my_color
	}
}
```

rust에서 범위는 다음과 같이 표현한다.
```rust
fn create_skystate(time:i32)->ThingsInTheSky{
	match time{
		6..=8 => ThingsInTheSky::Sun, //6<=time<=8
		2..6=>ThingsInTheSky::Starts //2<= time <6, 
		
	}
}
```
`..`은 exclusive, `..=`은 inclusive 이다.


### Enum

enum은 enum 키워드로 정의하며, enum 변수들 자체가 하나의 struct이므로 named struct 밑 Tuple struct를 선언할 수 잇다.
```rust
enum Things{
	Sun(String,String),  
	Stars{name:String,age:i32}
}
```
`enum`을 match하면 모든 변수를 포함하는 이상 `_` 가지를 필요로 하지 않는다.
```rust
fn match_things(things:&Things) ->i32{
	let mythings =match things{
		Things::Sun-> 10,
		Things:Stars-> 11
	};
	return mythings
}
```
enum 자체를 import 하면 참조 구문 `::`를 생략할 수 있다.
```rust
```rust
fn match_things(things:&Things) ->i32{
	use Things::*;
	match things{
		Sun-> 10,
		Stars-> 11
	}
}
```
return은 생략 가능하다.

enum 변수 자체를 문자열로 취급하는 언어도 있지만, rust에서는 그냥 struct일 뿐이므로 `Display` trait을 구현하지 않는이상 출력할 수 없다.

하지만 enum은 정수로 캐스팅된다. 다른 언어의 Ordinal 처럼 정의한 순서대로 부여된 숫자가 있으며 정수로 캐스팅할 시 이 숫자로 변환된다.

직접 숫자를 부여할 수도 있다. 이 경우에도 정수로 변환해야 이를 사용할 수있다.
```rust
enum Star{
	Brown =10,
	Mars=100,
	Uranos = 1000,
	Mercury=500,
	Saturn
}
fn main(){
	use Star::*;
	let starvec = vec![Brown,Mars,Uranos,Mercury,Saturn];
	for star in starvec{
		println!("{}", star);
		//10, 100, 1000, 500, 501
	}
}
```
숫자를 명시하지 않은 Saturn의 경우 바로 이전 enum의 숫자 바로 다음 정수가 할당된다는 점에 주의하자.


### 반복문

반복문은 `loop` 키워드를 사용해 작성할 수 있다. 중첩된 반복문에서의 break문은 가장 내포된 반복문만을 탈출한다. 탈출하고 싶은 반복문을 지정하기 위해서 라벨을 붙일 수 있는데 다음과 같다.
```rust
let counter = 0;
let second_counter=0;
`first_loop:loop{
	counter+=1;
	if counter>9{
		loop{
			second_counter+=1;
			if second_counter>9{
				break `first_loop; //labeled break
			}
		}
	}
}
```

반복문도 식으로 평가되기 때문에 값을 리턴할 수 있다. 반복문에서 값을 리턴하기 위해서는 `break 식;` 구문을 사용한다.
```rust
let counter=0;
let my_number = loop{
	counter+=1;
	if counter%10==3{
		break counter;
	}
}
```
my_number의 값은 counter의 값으로 결정된다.

rust에서는 변수가 데이터의 소유권을 가지기 때문에, 사용되지 않는 변수는 모두 컴파일러가 경고한다. 이를 방지하기 위해서는 `_`를 변수명 앞에 붙이면, 아직 사용중이지 않으나 곧 사용할 예정이라는 의미를 컴파일러에게 알려준다. 아예 사용하지 않을 변수는 \_ 를 쓰면 무시된다.

### Impl
struct는 데이터를 정의하고, impl은 메서드를 정의한다.
impl의 첫번째 인자로는 항상 `self`가 오는데, 만약 self 인자를 받지 않는 함수가 있다면 `static` 함수, 또는 `associated function` 이라고 하며 정적인 함수가 된다.
또한, 다른 모든 함수와 마찬가지로 &self로 소유권을 빌려오지 않은 경우, 매개변수로 들어온 self를 반환하지 않으면 self에 대한 소유권이 원래 인스턴스에게서 박탈된다. 또한 참조를 인자로 받을 경우 매개변수 사이즈가 작아지기 때문에, self는 참조로 인자를 받는 것이 좋다.
`&self`  로는 struct의 필드값을 변경할 수 없으며 &mut self 가변 참조로만 필드값을 변경할 수 있다. 이로부터 알 수 있는 것은 모든 `rust` 의 `struct`는 가변객체로 선언하지 않는 이상 필드값 또한 불변이라는 것을 알 수 있다.
```rust
struct Book{
	number:u32
}
impl Book{
	fn get_number(&self)->u32{
		self.number
	}
	fn change_number(&self, new_num:u32){
		self.number= new_num;
	}
	fn new(number:u32)->Self{
		Self{number}
	}

}
```
반환 타입으로 자기자신을 반환할 때는 struct 타입 이름을 써도 되고, `Self`를 사용해도 된다. `Self`는 키워드로, 자기자신의 타입을 말한다. 대문자가 아닌 `self`는 인스턴스를 말한다. 러스트에서 new는 키워드가 아니다.

associated method, 또는 static method는 인스턴스를 통해 호출할 수 없으므로 `::` 문법을 사용해 호출해야 한다.
```rust
fn main(){
	let my_book = Book::new(32);
}
```

Trait을 구현했다는 것을 알릴 때는
`#![derive(Debug)]`, 컴파일러 경고를 무시할 때는 `#![allow(unused_variables)]` 등을 사용하는데, 이러한 문법은 `attribute`라고 한다.

### 구조 분해 할당
```rust
#[derive(Debug)]
struct Person{
	name:String,
	age: u32,
	height: u32,
	happy:bool
}
fn main(){
	let person = Person{
		name: "clarence".to_string(),
		age: 32,
		height:170,
		happy:false
	};
	let Person{
		name,age,height,happy
	} = person;
	println!("{},{},{},{}", name,age,height,happy);
	//clarence, 32, 170, false
	//필드명 그대로 분해할당 가능
	let Person{
		name:person_name,
		age:person_age,
		height:person_height,
		happy:is_happy
	} = person;
	println!("{},{},{},{}", person_name,person_age,person_height,is_happy);
	//변수명을 정의하여 분해할당 가능
	let Person{
		name:my_name, 
		..
	} = person;
}
```
분해할당은 위와 같다.
필드명과 동일한 변수명을 사용하여 분해할당하거나, 별도의 변수명을 필드명에 매핑하여 분해할당 한다. struct의 이름을 정확하게 명시해야 한다. 원하는 필드만 분해할당 하기 위해서는 `..` 문법을 사용한다. 자바스크립트와 거의 동일하다. (자바스크립트는 `...`)

```toc
```
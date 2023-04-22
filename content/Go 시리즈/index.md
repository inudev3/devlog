---
emoji:
title: go 시리즈 #1
date: '2023-04-17 03:00:00'
author: inu
tags: golang goroutine
categories: golang
---
go는  포인터의 라이프타임이 데이터보다 더 긴 경우, 해당 데이터를 자동으로 힙에 할당해준다. 이를 클로져라고 한다.
예를 들어
```go
func fib() func() int{
	a,b := 0,1
	return func() int{
		a,b = b, a+b
		return b
	}
}
```
이 함수는 클로져를 반환하고, 이 때 클로져의 라이프타임은 fib() 함수의 두 지역변수를 포함하기 때문에 go 컴파일러는 이 두 변수를 힙에 할당한다.

모든 클로저가 포함하는 (`close over`) 것은 변수의 값이 아니라 변수의 참조/포인터 이다. 따라서 c 언어 등에서는 클로져를 반환할 때 dangling pointer같은 문제가 발생하는 것이다.

슬라이스를 초기화 하지 않으면 nil이다. 그러나 초기화 했지만 아무 원소도 없는 empty slice일 수도 있다.
중요한 건 두 경우 모두 len이 0이라는 점이다.

make로 슬라이스를 만들 때 cap을 지정하지 않으면 언제까지고 append할 수 있다.
그러나 cap을 지정한 경우에는 cap 이상으로는 append하더라도 변하지 않는다.

가장 헷갈리는점 : nil map에 무언가를 insert하면 에러가 나지만 nil slice에 무언가를 append하면 추가된다.
주의하자.

```go
a := [3]int{1,2,3}
b := a[:1]
c := b[:2]
```
얼핏보면 말도안된다고 생각할 수 있다.
배열을 slicing하면 슬라이스의 cap은 원본 배열의 크기이다.
그리고 슬라이스를 또 슬라이스 할 수 있다. 이 경우에 2차 슬라이스의 cap은 결국 원본 배열의 크기이다. 따라서 c의 cap은 a의 길이이다.

slicing 문법은 다음과 같다.
```go
a := []int{1,2,3,4,5}
b := [i:j:k]
```
b의 len은 j-i, cap은 k-i이다.

map의 제한점은 map의 엔트리는 주소로 참조할 수 없다. 또한 내부 엔트리를 포인터가 아닌 값 타입으로 가지고 있다면 엔트리를 변경할 수 없다.
서로 이름만 다르고 구조는 같은 두 타입이 있다고 하자.
서로 다른 타입의 변수는 절대 할당할 수 없다.
그러나 서로 동일한 구조를 가진 타입은 항상 변환할 수 있다.
```go
type T1 struct{
	X int
}
type T2 struct{
	X int
}
func main(){
	v1 := T1{1}
	v2 := T2{2}
	//v1 = v2 //error
	v1 = T1(v2) //항상 타입 변환 됨
}
```
struct 타입 또한, 포인터로 넘기지 않는 이상 복사하여 전달된다.
go는 메모리를 스택에 할당하는 것을 우선시하지만 메모리의 라이프타임이 그보다 더 길어질 때는 힙에 할당하며 이를 `escape analysis`라고 한다.
따라서 go에서는 new로 무언가를 생성하더라도 조건만 충족한다면 스택에 메모리를 할당한다.
`range`로 반환되는 이터레이터의 `value`는 항상 복사본이다. 따라서 for loop 내에서 value를 아무리 수정해도 블럭 밖에서 반영되지 않으며, 수정하려면 인덱스로 접근해야 한다.
```go
for i, value :=range things{
	value = whichever
	//반명안됨
}
for i := range things{
	things[i] = whichever //반영됨
}
```
슬라이스에 append를 사용할 때는 항상 원래 슬라이스 변수에 이를 재할당해야한다. 왜냐하면, 슬라이스를 변경하는 연산은 슬라이스가 가리키는 포인터를 재할당할 수 있기 때문이다. 즉 슬라이스가 다른 배열을 가리키게될 수가 있으므로,
```go
things := []int
append(things, 5) //다른 슬라이스가 될 수 있음
things := append(things,5)
```
이는 기타 슬라이스를 변경하는 함수도 마찬가지이다. 슬라이스를 반환하는 것이 안전하다. 변경시에 슬라이스가 가리키는 포인터가 변경될 수 있다.

슬라이스는 언제든지 cap을 초과할 시 재할당 될 수 있으므로 슬라이스의 원소에 대해 포인터는 안전하지 않다.

```go
func main(){
	items := [][2]byte{{1,2},{3,4},{5,6}}
	a := [][]byte{}
	for _, item := range items{
		a = append(a, item[:])
	}
	fmt.Println(items)
	fmt.Println(a)
}
```
range 함수의 반환 value은 copy라고 했었다. 매 iteration마다, a에 item에 대한 슬라이스(포인터) 를 추가한다. 그런데, 매 iteration은 item을 overwrite 하고 있다. 따라서 가장 마지막에 a에 출력되는 결과물은
`[[5,6], [5,6], [5,6]]` 이다.  따라서 이 경우에는 item을 복사한 변수를 하나 더 루프 안에 만든 후 그 변수에 대한 슬라이스를 추가해야한다.
```go
func main(){
	items := [][2]byte{{1,2},{3,4},{5,6}}
	a := [][]byte{}
	for _, item := range items{
		i := make([]byte, len(item))
		copy(i, item[:])
		a = append(a, i)
	}
	fmt.Println(items)
	fmt.Println(a)
}
```
copy 하려면 len이 copy할 배열/슬라이스와 일치해야 한다. cap

메서드의 수신자 타입이 포인터가 아닌 값이 라면, 해당 메서드는 수신자가 복사된다.

어떤 두 인터페이스가 포함관계에 있다고 하면, 더 일반적인 인터페이스에 더 제한적인 인터페이스를 할당할 수는 없지만 반대는 가능하다.

어떤 포인터에 수신자 메서드를 정의했다고 하면, 수신자 메서드는 `변수`에 의해 호출되어야 한다. 생성 리터럴은 항상 값을 리턴한다.

```go
type IntSet struct{...}
func (*IntSet) String() string //fmt.Stringer 인터페이스
var _ = IntSet{}.String() //에러, 포인터가 아니라 값임
var s IntSet
var _ = s.String() //포인터
var _ fmt.Stringer = s //에러, 값으로 할당할 수 없음
var _ fmt.Stringer = &s // 포인터로 정의했으므로 포인터로 할당해야 함
```

두 인터페이스를 합성한 인터페이스를 만들 수 있다.
```go
type Reader{}
type Writer{}
type ReaderWriter{
	Reader
	Writer
}
```

하나의 타입에 대한 메서드는 동일한 패키지 내에서만 작성되어야 하며 다른 패키지의 타입에 대한 수신자 메서드를 만들 수 없다. 그러나 다른 패키지의 타입을 합성하고 확장할 수 있다.

### Channel

chan 키워드로 사용한다.
채널에 보내진 데이터보다 받으려고 시도하는 횟수가 더 많아지면 쓰레드를 블록한다. (버퍼가 없는 이상). 또한, 채널에 receiver가 없으면 데이터를 send할 수도 없다. 즉 reader가 있어야 데이터를 보낼 수 있고, 채널에 데이터가 있어야 채널로부터 데이터를 받을 수 있다. 한쪽만 있으면 어느 경우든 쓰레드는 블록된다.

### select
select는 복수의 채널에서 데이터를 수신하면서 그 중 가장 먼저 전송된 데이터를 수신한다.
수신하지 않는 나머지 case 채널은 데이터가 전부 그대로 남아있다. 복수의 채널에서 수신 포인트를 하나로 가질 수 있어 타이밍 등을 동기화할 때 유용하다.
select 문에서 break를 사용할 수 있다. switch와 마찬가지이다. default문도 쓸 수 있어 모든 채널 case가 fall throught 될 경우 실행되게 할 수 있지만, for 문 또는 무한 루프에서 select를 실행중이라면 default가 계속해서 실행되므로 사용하지 말아야 한다.

### time.After
time 라이브러리 중, 특정 시간 후에 시그널을 보내는 채널 타입의 함수이다. chan Time
log.Fatal은 런타임에 프로그램을 패닉시킨다.
time.After가 수신 후 특정 시간 뒤에 한번만 송신한다면, time.NewTicker는 지정한 시간 간격마다 송신한다.
단 After와는 다른 인터페이스를 사용하여 NewTicker 자체는 Ticker 오브젝트이며 NewTicker().C가 채널이다.

### Context
컨텍스트는 취소를 위해 사용하는  불변 컨텍스트로 이루어진 트리 자료구조이다. 빈 컨텍스트인 context.Backgroun()를 루트 컨텍스트로 하는 자식 컨텍스트를 만들고, timeout 혹은 어떤 값 등의 조건을 사용해 다른 routine에 자식 컨텍스트를 넘기고
`defer cancel()` 를 호출하면, 해당 자식 컨텍스트의 조건 등에 따라 cancel()이 실행된다.함수의 첫번째 인자로 context를 넘기는 것이 컨벤션이며, http Request 등의 라이브러리에 context에 따라 동작할 수 있는 편의함수등의 구비되어 있어 contextTimeout의 경우 ContextTimeoutExceeded 등의 에러를 라이브러리에서 반환하기도 한다.
context에는 Done, Err(), Deadline() 등의 채널이 존재하여 부모 컨텍스트에서 에러 / 타임아웃 등으로 취소되면 해당 채널로 시그널이 전송된다. 따라서 다음과 같은 select 문을 사용하는 것이 일반적이다.
 ```go
 func get(ctx context.Context, url string, ch chan<- result) {  
	start := time.Now()  
	req, _ := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)  
	if resp, err := http.DefaultClient.Do(req); err != nil {  
		ch <- result{  
			url, err, 0,  
		}  
	} else {  
		t := time.Since(start)  
		ch <- result{url, nil, t}  
		resp.Body.Close()  
	}  
}
func first(ctx context.Context, urls []string) (*result, error) {  
	results := make(chan result)  
	ctx, cancel := context.WithCancel(ctx)  
	defer cancel()  
	for _, url := range urls {  
		go get(ctx, url, results)  
	}  
	select {  
	case r := <-results:  
		return &r, nil //return first  
	case <-ctx.Done():  
		return nil, ctx.Err()  
	}  
}
```
아래 first함수를 잘 살펴보면, 두가지 케이스가 있는데 첫번째는 자식 컨텍스트와 함께 실행한 채널의 결과를 수신하고, 두번째는 자신이 부모로부터 받은 컨텍스트가 종료되는 경우다. 따라서 자식 컨텍스트가 에러/타임아웃이 나는 경우는 부모에서 핸들링 되지 않고 자식 컨텍스트에서 (여기서는 `get` 함수) 핸들링 됨을 알 수 있다.

컨텍스트는 키-밸류의 맵 형태이므로, 서로 다른 라이브러리 간의 컨텍스트 키 충돌이 일어나지 않게 하기 위해 패키지마다 별도의 프라이빗 컨텍스트 키를 만들어 사용한다.





### 채널 leak

channel leak은 채널에 데이터를 보내거나 받을 때 반대편에 수신자/송신자가 없어서 무한하게 대기하며 쓰레드를 블럭하는 것을 말한다. 채널이 GC의 대상이 되지 못한채로 메모리가 leak된다. 이를 방지하기 위해서는 채널에 버퍼를 주면 주어진 버퍼 이하에서는 수신자/송신자가 없어도 데이터를 보내거나 받고 쓰레드를 블럭하지 않는다. 쓰레드를 블럭하지만 않는다면 닫히지 않는 채널은 GC의 대상이 된다.

Go는 내장된 데드락 디텍터가 있다.
데드락은 언제발생하는가? 데드락은 일종의 부수효과이다. 동시성에서 해결하고자 하는 것은 race condition, 경쟁 상태이다. 이를 해결하기 위한 동기화 도구들이 채널 등인데, 이러한 채널 등이 작동할 때 발생하는 현상이 데드락이다.

채널 변수를 make()를 통해 생성하지 않으면 nil 값으로 초기화된다. nil 채널은 read/write 모두 할 수없으며 고루틴을 블럭한다. 하지만 단 한가지, `select` 구문에서 nil 채널이 있다면 이 채널은 무시된다. 없는 case와 마찬가지이다.


채널을 Close 닫는 것은 채널 leak을 막는 가장 간단한 방법이다. 채널을 누가 닫느냐가 중요한 문제이며, 이를 해결하는 것이 structured goroutine을 만드는 방법이다. 닫힌 채널을 또 닫으려하거나, 혹은 닫힌 채널에 데이터를 보내려고 하면 패닉을 일으킨다. 다만, 닫힌 채널로부터 데이터를 수신한다면 이 때는 `zero-value`를 수신한다.

채널은 디폴트로 버퍼가 없으며 이를 `rendez-vous`라고 한다. 데이터를 send할 경우 다른 쪽에서(read-end) 데이터를 receive 할 때까지 sender의 고루틴은 블럭된다. 수신자가 데이터를 수신한뒤 return하면  송신자가 return한다. 수신자가 항상 송신자보다 먼저 return한다. 즉 송신자는 채널에 데이터를 보낸 뒤 return 될 때 수신자가 수신했다는 것을 알게 된다. rendez-vous 채널에서 수신자가 받은 데이터는 송신자가 보낸 데이터임이 보장되며, 이를 통해 송신자와 수신자는 동기화 된다.
하지만 버퍼가 있는 채널에서는 이런 일이 일어나지 않는다. 송신자는 데이터를 보내고 바로 종료되며, 수신자 또한 버퍼가 들어있는 한 바로 종료된다. 송신자가 데이터를 보낸 뒤 수신자가 데이터를 채널로 부터 받았다고 해도 두 데이터가 동일하지 않을 수도 있다. 즉 동기화가 일어나지 않는다.

랑데부의 동기화를 알아보기 위해 다음 예제를 살펴보자.

```go
type T struct{
	i byte
	b bool
}
func send(i int, ch chan<- *T){
	t := &T{i:byte(i)}
	ch <-i
	t.b = true //채널에 데이터를 보낸 뒤에 데이터를 수정하면 어떻게 될까?
}
```
send는 채널에 T 스트럭트에 대한 포인터를 보낸 뒤 이를 수정하고 있다.

```go
func main(){
	vs := make([]T, 5)
	ch := make(chan, *T)
	for i:= range vs{
		go send(i, ch)
	}
	time.Sleep(1*time.Second) //make sure all goroutines started
	for i:= range vs{
		vs[i]= *<- ch //역참조한 값을 복사함
	}
	for _, v := range vs{
		fmt.Pritnln(v)
	}
}
```
이제 time.Sleep으로 1초의 슬립을 통해 모든 고루틴이 채널에 데이터를 보내도록 한 뒤 채널에서 T 스트럭트의 포인터를 수신하고 역참조한다. 역참조한 값을 vs 배열에 복사함으로써, 수신할 때의 상태를 확인하려고 한다. 보다 자세하게는 send 채널이 T 포인터를 보낸 후 이를  `true` 로 수정한 것이 반영이 됐는지 안됐는지를 확인할 것이다. 반영이 됐다면 true, 되지 않았다면 (수신 후에 포인터를 수정했다면) 디폴트값인 fale가 나올 것이다.
출력 결과는 어떨까? 다섯 번 모두 false가 나온다. 그말인 즉슨, 채널의 데이터가 수신되고 나서야 포인터를 통한 수정이 이루어졌다는 얘기다. 이는 앞서 언급한, 수신자의 데이터 수신이 끝나야 송신자의 송신이 리턴되는 - 랑데부 채널의 동기화 특성이다. 즉 앞서 `ch<- i` 코드는 `vs[i] = *<- ch` 로 수신하기 전에는 고루틴을 블럭하고 있었다. 


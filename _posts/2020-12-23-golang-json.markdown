---
layout	: posts
title	: Golang 학습 일기 #1 (Json)
summary	: Golang
data	: 2020-12-23 19:35:14 +0900
updated	: 2020-12-23 19:35:14 +0900
comment	: true
categories: CI/CD
tags:
  - Golang
  - Json
---

* TOC
{:toc}

## Study with Golang(1)


### JSON 읽기 쓰기
JSON을 읽고 쓰는 라이브러리가 여러가지 있겠지만 기본적인 내장 표준 라이버리리인 encoding/json 패키지를 이용하여 Encoding/Decoding 하는 방법을 습득 하였다.

해당 정리는 `Building Microservice with Go` 책을 학습 하면서 정리 한 내용이다.  

#### 개인적으로 정리 한 내용 이므로 추후 자세히 알게 되는 부분을 지속적으로 수정할 예정이다.


### Masharing Json with Golang

```go
func Marshal(v interface{}) ([]byte, error)
```

encoding/json은 위와 같은 marshar 함수를 제공 해주며 해당 함수는 Interface 타입의 매개 변수를 하나 입력받고 ([]byte, error)의 튜플 리턴한다. Go lang은(리턴 타입, 에러) 튜플 리턴하는 방식을 사용하며, 리턴에 성공하면 에러는 nil값을 반환한다.

Golang에 대해 전문가 처럼 아는게 아니지만 함수를 처리시 문제가 발생되면 `Panic` 함수가 프로그램의 정상 실행을 중지 시키고 Go 루틴의 모든 지연(`defer`)된 함수를 싱핼하고 나면 로그 메시지를 출력하고 애플리케이션을 종료 하며, 이런 방식ㄱ을 코드상 버그나 unexpected error를 처리하는대 사용 된다고 한다.

위와 같은 패턴으로 Marshal 함수로 구현 되어있어 JSON으로 인코딩된 바이트 배열을 생성 할 수 없는 경우 에러 객체를 호출한 함수로 리턴된다.    


```go
type helloWorldResponse struct {
    Message string
}

func helloWorldHandler(w http.ResponseWriter, r *http.Request) {

	response := helloWorldResponse{Message: "Hello World"}
	data, err := json.Marshal(response)
	if err != nil {
		panic("Error")
	}

	fmt.Fprint(w, string(data))
}
```

위 코드는 Error가 발생되면 error 메시지를 축력 해주고 정상적인 응답시에는 `{"Message":"Hello World"}` 메시지를 출력해준다.

JSON 구조체 필드 이름을 가져와서 Marshal 하는 방법이 기본 방식인데 만약 `"message"`로 표시 되게 하고 싶은 경우  helloWorldResponse Struct에서 필드 이름을 바꿔야 할까?

몰랐던 사실이지만(GoLang 시작한지 얼마 되지 않음) Go lang에서는 소문자로된 properties는 내보낼수 없어서 Marshal 함수는 이러한 properties를 무시하여 Print시 Properties를 포함 하지 않는다고 한다.

즉 Encodig/json 패키지는 프로퍼티의 출력을 프로그래머가 선택한 방식으로 변경 할 수 있도록 Struct 필드 속성을 구현하고 있기 때문에 값 자체가 손실 되는 것이 아니다.


```go
type helloWorldResponse struct {
    // 출력 필드를 "message"로 변경한다"
    Message string `json:"message"`
    // 해당 필드는 출력하지 않는다.
    Author  string `json:"-"`
    // 값이 비어 있으면 필드를 출력하지 않는다.
    Data    string `json:",omitempty"`
    // 출력을 문자열로 변환하고 이름을 "id"로 변경한다.
    Id    int   `json:"id,string"`
}
```

이런식으로 Struct 필드에서 태그를 사용하면 출력을 어떻게 표시 할지 자세히 제어 할 수 있다. 

그러면 웹 요청을 응답 받아서 처리 할 경우는 어떻게 처리 할까? encoding/json에서는 인코더 및 디코더를 제공하며 `ResponseWriter`를 사용 하여 응답 스트림을 처리 하면 된다.

책에서 자세히 설명이 되어있지만 `ResponseWriter`는 아래와 같이 3가지 메서드를 정의 하는 인터페이스이다.

```go
// WriteHeader Method를 통해서 전송될 해더들의 Map을 리턴한다.
Header()

// 연결(connection)에 데이터를 쓴다. WriteHeader Method가 아직 호출되지 않았다면
// WriteHeader(http.StatusOK)를 호출한다.
Writer([]byte) (int, error)

// 상태 코드(status code)를 포함한 HTTP 응답 헤더를 전송한다.
WriteHeader(int)
```

사실 위에 인터페이스 정의만 보았을때 이해하기 힘들었고 실제로 사용해서 어떻게 리턴하나 확인 해보았다.

실제로 Handler에 `NewEncoder`를 사용을 해보았다.

```go
func helloWorldHandler(w http.ResponseWriter, r *http.Request) {
	response := helloWorldResponse{Message: "HelloWorld"}

	encoder := json.NewEncoder(w)
	encoder.Encode(response)
}
```

처음에는 위와 같이 코드를 작성하고 결과 값 확인시 처음에 encoder를 사용하지 않고 출력한 결과 값과 동일해서 무엇이 다르나 확인 해보았는대 차이점은 바이트 배열로 Masaring이 아닌 Encoder에서 응답을 하는 방식이라고 한다. 개인 적으로 이부분은 성능관련에서 차이가 날 듯한대 나중에 한번 다시 정리가 필요 할 것같다. 하지만 성능적인 부분에서 보자면 코드를 더 실행 할 수록 그만큼 Latency가 발생 되기때문에 적은 코드를 실행 하는 encoder가 더 빠르지 않을까 추측해본다.

추후 테스트 코드를 짜서 확인 해보도록 하자.


### Unmasharing Json with Golang

```go
func Unmarshal(data []byte, v interface{}) error
```

encoding/json을 사용하여 Json을 Client로 전송하는 방법(Marshal)을 학습 했는대 반대로 출력 값을 읽어야 할 경우는 어떻게 처리 해야할까?  
Unmashal Method는 Masharl Method와 반대 방식으로 작동하며 필요에 따라 맵, 슬라이스 및 포인터를 할당 한다. 입력받은 객체의 키가 Struct 필드의 이름이나 태그와 일차하는 필드를 찾아서 처리 하기 때문에 대소문자 구분을 정확히 일치 하는것이 좋다.

```go
type helloWorldRequest struct {
		Name string `json:"name"`
}
```
위와 같이 `{ "name": "---" }`과 동일한 구조체로 Marshalling 하도록 작성 하였다.

그리고 Request와 함께 전송된 JSON에 접근하려면 Handler에 전달된 http.requet를 이해 할 필요가 있다.
책에서 간편히 http.Request 객체에 대해 설명이 들어있어서 명시를 했다 자세한 정보는 아래의 링크에서 확인 하는게 좋다.  

link: https://godoc.org/net/http#Request

해당 example은 테스트한 method만 작성을 하였다.

```go
type Requests struct {
    ...
        // Method는 HTTP  요청 방식을 지정한다. ex) GET, POST, PUT and so on.
        Method string
        
        // Header는 서버가 수신한 요청 Header 필드를 가지고 있다.
        // Header 타입은 map[string] []string에 대한 연결이다.
        Header Header

        // Body 요청의 본문이다.
        Body io.ReadCloser
    ...    
}
```

Http Request를 호출하는 예제를 작성 하였다. 만약 `ioutil.ReadALL`을 호출 했다면 클라이언트가 자동적으로 closed되지 않기 때문에 Close()를 반드시 호출 해줘야한다. 하지만 ServeHTTP 핸들러에서 사용 될때는 서버가 자동으로 요청 된 스트림을 Closed 처리 한다.

```go
func helloWorldHandler(w http.ResponseWriter, r *http.Request) {

		body, err := ioutil.ReadAll(r.Body)
		if err != nil {
			http.Error(w, "Bad request", http.StatusBadRequest)
			return
		}

		var request helloWorldRequest
		err = json.Unmarshal(body, &request)
		if err != nil {
			http.Error(w, "Bad request", http.StatusBadRequest)
			return
		}

		response := helloWorldResponse{Message: "Hello " + request.Name}

		encoder := json.NewEncoder(w)
		encoder.Encode(response)
}
```

위와 같이 코드를 작성하여 Curl 명령으로 호출 하면 아래와 같은 결과 값을 볼 수 있다. 

```bash
$ curl localhost:8080/helloworld -d '{"name":"TRQ1"}'

{"message":"Hello TRQ1"}
```

만약 위와 같이 body를 포함 시키지 않으면 error가 발생 하여 `Bad request` 메시지 발생 시킨다.  


### 결론
Go lang에서 Marshalling/Unmarshalling이 어떻게 되는지 학습 해보았으며, encoding/json 패키지를 사용하면 json을 인코딩 디코딩 하는 방법을 알게 되었다. 사실 기본적인 사용법 학습이고 추후 Golang으로 bot을 만들시 좀더 유익하게 사용해볼 예정이다.
추후 자세하 확인이 필요하면 link: https://golang.org/pkg/encoding/json 을 살펴볼 예정이다.
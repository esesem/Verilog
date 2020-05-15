Verilog and SystemVerilog Gotchas
=================================
101 Common Coding Erros and How to Avoid Them
---------------------------------------------

## Gotcha 1: Case Sensitivity

> *Gotcha*: VHDL 모델의 코드에는 문제가 없고 정상적으로 동작하는 것처럼 보이는데, Verilog/SystemVerilog에서는 `undeclared identifier` 에러가 발생한다.
>
> *Synopsys*: Verilog와 SystemVerilog는 **대소문자를 구분하는 언어**지만, VHDL은 **대소문자를 구분하지 않는**다.


### Gotcha!

```SystemVerilog
module FSM (...);

    enum logic [1:0] {WAIT, LOAD, READY} State, nState;
    ...
    always_comb begin
        case (state)                    // GOTCHA!
            WAIT:  nState = LOAD;       // GOTCHA!
            LOAD:  nState = READY;
            READY: nState = wait;       // GOTCHA!
        end case
    end
endmodule: FSM
```


### How To Avoid This Gotcha

좋은 명명 규칙(naming convention)을 적용하고, 이 규칙 들의 사용을 강제 해야한다. 예를 들면 아래와 같이 사용한다.

* 변수와 넷은 소문자로 작성
  * 사용자 정의(user-defined) 타입은 `_t`로 끝나도록 작성
  * 열거형(enumerate) 변수 이름은 `_e`로 끝나도록 작성
  * Active-low signal 이름은 `_n`로 끝나도록 작성
* 상수 이름과 열거된 레이블(enumerated label)은 모두 대문자로 작성
* 클래스 선언 이름은 첫 글자는 대문자, 나머지는 소문자로 작성
* 키워드와 충돌되지 않는 이름을 선택


### Corrected Version

```SystemVerilog
module FSM (...);

    enum logic [1:0] {HOLD, LOAD, READY} state_e, nstate_e;
    ...
    always_comb begin
        case (state_e)                      // OK, names match declarations
            WAIT:  nstate_e = LOAD;         // OK, names match declarations
            LOAD:  nstate_e = READY;
            READY: nstate_e = HOLD;         // OK, names match declarations
        end case
    end
endmodule: FSM
```




## Gotcha 2: Implicit Net Declarations

> *Gotcha*: 설계에서 커넥션에 있는 오타(typo)가 컴파일러로 잡히지 않았고, 시뮬레이션에서 기능 문제(functional problem)로만 확인된다.
>
> *Synopsys*: 잘못 타이핑 된 식별자(identifier)에 대해 문법 오류(syntax error)가 나타나는 대신 암묵적 넷(implicit net)을 추론(infer)한다.

미선언 식별자(undeclared identifier)가 삽입되면 어떻게 사용되느냐에 따라 Verilog/SystemVerilog 툴이 다르게 동작한다.

* 미선언 식별자가 오른쪽 또는 절차적 할당문(procedure assignment statement)의 좌변에 사용되면 컴파일 에러가 발생한다.
    ```SystemVerilog
    logic [7:0] foo;
    initial foo = bar;      // ERROR: bar not declared
    ```

* 미선언 식별자가 연속적 할당문(continuous assignment statement)의 우변에 사용되면 컴파일 에러가 발생한다.
    ```SystemVerilog
    logic [7:0] foo;
    assign foo = bar        // ERROR: bar not declared
    ```

* 미선언 식별자가 연속적 할당문의 좌변에 사용되면 **암묵적 넷 선언이 추론되고, 에러나 경고가 리포트 되지 않는다**.
    ```SystemVerilog
    logic [7:0] foo;
    assign bar = foo;       // GOTCHA: bar not declared, but no error
    ```

* 미선언 식별자가 모듈의 인스턴스, 인터페이스(interface), 프로그램(program), 또는 프리미티브(primitive)의 커넥션에 사용되면 **암묵적 넷이 추론되고, 에러나 경고라 리포트 되지 않는다**.


### Gotcha!

```SystemVerilog
module adder (input  logic a, b, ci,
              output logic sum, co);
    ...
endmodule

module top;
    wire a, b, ci, s1, s2, c1;
    adder i1 (.a(a), .b(b), .ci(cl), .sum(s1), .co(c1) );   // GOTCHA!
    adder i2 (.a(a), .b(b), .ci(c),  .sum(s1), .co(c1) );   // GOTCHA!
endmodule
```


### How To Avoid This Gotcha Using Verilog

Emacs의 Verilog 모드와 같은 언어 인식형 에디터(language-aware editor)를 활용하여 Verilog의 넷리스트를 자동으로 넣어준다.


### How To Avoid This Gotcha Using SystemVerilog

SystemVerilog에는 *닷-네임(dot-name)*, *닷-스타(dot-star)* 단축어가 제공되어 포트 연결 이름의 반복을 줄여준다.

* 닷-네임 단축어는 연결된 포트를 명시하지만, 같은 이름의 넷이 연결되어 있음을 추론한다.
* 닷-스타 단축어는 자동으로 같은 이름으로 포트와 신호가 연결되어 있음을 추론한다.

타이핑 해야하는 신호 이름의 개수를 줄임으로써, 오타로 인한 에러의 가능성을 줄일 수 있다. 닷-네임과 닷-스타 단축어 역시 모든 넷을 명시적으로 선언해야 한다. 단축어를 사용하면 넷리스트나 넷 선언에 있는 오타는 명시적 와이어로 추론되지 않는다. 그러한 오타는 시뮬레이션에서의 기능적 문제(functional problem) 대신에 컴파일 에러를 발생시킨다.

```SystemVerilog
module adder (input  logic a, b, ci,
              output logic sum, co);
    ...
endmodule

module top;
    wire a, b, ci, s1, s2, c1;
    adder i1 (.a, .b, .ci, .sum(s1), .co(c1) );     // .name connections
    adder i2 (.sum(s2), .ci(c1), .* );              // .* connections
endmodule
```




## Gotcha 3: Default of 1-bit Internal Nets

> *Gotcha*: 벡터에서 넷리스트의 포트에 bit 0만 연결된다.
>
> *Synopsys*: 넷리스트에서 미선언된 내부 연결은 벡터에서 포트에 넷이 연결되어 있어도 1-bit 와이어로 추론된다.

Verilog와 SystemVerilog에는 넷리스트를 모델링하는데 편리한 단축어가 있다. 즉, 서로 연결되는 모든 넷들을 꼭 선언할 필요는 없다. 미선언된 식별자들(undeclared identifiers)은 포트 연결에 사용되며 기본적으로 와이어 넷 타입이 된다. 넷리스트의 수백 수천개의 연결이 암묵적인 와이어가 되어 소스 코드를 상당히 간결화 한다.

암묵적 넷들의 벡터 사이즈는 로컬 컨텍스트에 의해 결정된다. 만일 미선언된 신호들이 신호를 포함하는 모듈의 포트라면, 암묵적 넷이 포함된 모듈의 포트와 동일한 사이즈로 추론된다. 만일 미선언된 신호들이 포함된 모듈 부분에서만 내부적으로 사용된다면, 1-bit 넷이 추론된다. Verilog와 SystemVerilog는 암묵적 넷 타입의 사이즈를 결정하기 위해 연결된 신호의 포트 사이즈는 보지 않는다.


### Gotcha!

```SystemVerilog
module top_level
(output [7:0] out,                  // 8-bit port, no data type declared
 input  [7:0] a, b, c, d            // 8-bit ports, no data type declared
);

    mux4 m1 (.y(out),               // output infers an 8-bit wire type
             .a(a),
             .b(b),
             .c(c),
             .d(d),
             .sel(select) );        // GOTCHA! select infers 1-bit wire
    ...
endmodule

module mux4
(input  logic [1:0] sel,            // 2-bit input port
 input  logic [7:0] a, b, c, d,     // 8-bit input ports
 output logic [7:0] y               // 8-bit output port
);
    ...
endmodule
```


### How To Avoid This Gotcha

1-bit 보다 큰 모든 내부 넷과 변수는 반드시 명시적으로 선언해야 한다. SystemVerilog의 닷-네임, 닷-스타 포트 연결 단축어는 이 실수를 방지해준다. 이 단축어는 선언되지 않은 넷에 대해서는 추론하지 않는다. 또한, 사이즈가 맞지 않는 연결은 추론하지 않는다.

```SystemVerilog
module top_level
(output [7:0] out,                  // 8-bit port
 input  [7:0] a, b, c, d            // 8-bit ports
);

    mux4 m1 (.y(out), .* );         // ERROR: no net declared for sel port
endmodule

module mux4
(input  logic [1:0] sel,            // 2-bit input port
 input  logic [7:0] a, b, c, d,     // 8-bit input ports
 output logic [7:0] y               // 8-bit output port
);
    ...
endmodule
```




## Gotcha 4: Single File Versus Multi-File Compilation of `$unit` Declarations

> *Gotcha*: 내 모델도 컴파일 되고 다른 그룹의 모델도 컴파일 되는데, 같이 컴파일 하면 여러 선언과 관련된 에러가 발생한다.
>
> *Synopsys*: 분리된 파일 컴파일은 분리된 `$unit` 선언 네임 스페이스를 가진다. 멀티-파일 컴파일은 하나의 `$unit` 컴파일 네임 스페이스를 가진다.

`$unit`은 함께 컴파일 되는 모든 디자인 유닛에 보이는 선언 스페이스이다. `$unit`의 목적은 설계와 검증 엔지니어에게 공유된 정의(definition)과 선언(declaration)을 제공하는 것이다. 
모듈(module), 인터페이스(interface), 테스트 프로그램(program), 또는 패키지(package)에 존재하지 않는 어떤 사용자 정의 타입, 태스크(task) 정의, 펑션(function) 정의, 파라미터(parameter) 선언 또는 변수 선언들을 자동으로 `$unit`에 위치한다. 모든 실제적 목적을 위해서, `$unit`은 모든 모델링 블록에 자동으로 와일드카드로 임포트 된 미리 정의된 패키지 이름으로 생각할 수 있다. `$unit`에 있는 모든 선언은 특별한 참조 없이도 볼 수 있다. 또한, 패키지 scope resolution operator를 이용하여 명시적으로 참조할 수도 있다. 만약 여러 패키지에 식별자가 존재할 경우 필요하다. `$unit`에 명시적 참조의 예는 아래와 같다.

```SystemVerilog
// following declaration is in $unit
typedef enum logic [1:0] {RESET, HOLD, LOAD, READY} states_t;

module chip (...);
    ...
    $unit::states_t state_e, nstate_e   // OK: definition in $unit
```

`$unit`과 관련된 실수는, 공유된 정의와 선언이 여러 소스 코드 파일에 흩어질 수 있고, 파일의 시작이나 끝에 있을 때 발생할 수 있다. 잘해봐야 비구조적이거나 스파케티 코드 모델링 스타일이 설계와 검증 코드가 디버깅 하기 어렵게 할 수 있고, 유지하고 재사용하기 거의 불가능하게 할 수 있다. 나쁜 경우에는 `$unit` 정의와 선언이 여러 파일들 간에 흩어져 이름 resolution 충돌이 발생되는 결과를 낳을 수 있다. 예를 들면, 디자인이 `$unit`에 `RESET`이라는 레이블을 포함하는 열거형 타입의 정의를 가지고 있다고 하자. 이 자체만으로는 디자인의 컴파일이 잘 될 것이다. 하지만 그 때 `RESET`이라 부르는 레이블을 포함하는 열거형 타입의 정의를 가지는 `$unit`을 포함하는 IP 모델이 디자인에 추가 된다면 어떻게 될까? IP 모델도 그 자체만으로는 컴파일이 잘 되겠지만, 디자인의 `$unit` 정의와 같이 컴파일하게 되면 네임 충돌이 발생한다.


### How To Avoid This Gotcha

`$unit` 대신에 공유된 선언을 사용하려면 패키지(package)를 사용한다. 패키지는 의도치 않은 스파게티 코드를 방지하고 공유된 정의와 선언을 공유하는 역할을 한다. 또한, 패키지는 자신 고유의 네임 스페이스를 가지고, 다른 패키지에 있는 정의와 충돌하지 않는다. 만약 두 패키지가 같은 네임스페이스에 와일드카드로 임포트되면 네임 충돌 문제는 여전히 발생할 수 있다. 이것은 와일드카드 임포트 대신에 명시작 패키지 임포트와(또는) 명시적 패키지 참조로 방지할 수 있다.




## Gotcha 5: Local Variable Declarations

> *Gotcha*: 
>
> *Synopsys*: 
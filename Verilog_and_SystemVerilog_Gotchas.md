Verilog and SystemVerilog Gotchas
=================================
101 Common Coding Erros and How to Avoid Them
---------------------------------------------

# Gotcha 1: Case sensitivity

> *Gotcha*: VHDL 모델의 코드에는 문제가 없고 정상적으로 동작하는 것처럼 보이는데, Verilog/SystemVerilog에서는 `undeclared identifier` 에러가 발생한다.
>
> *Synopsys*: Verilog와 SystemVerilog는 **대소문자를 구분하는 언어**지만, VHDL은 **대소문자를 구분하지 않는**다.


## Gotcha!

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


## How To Avoid This Gotcha

좋은 명명 규칙(naming convention)을 적용하고, 이 규칙 들의 사용을 강제 해야한다. 예를 들면 아래와 같이 사용한다.

* 변수와 넷은 소문자로 작성
  * 사용자 정의(user-defined) 타입은 `_t`로 끝나도록 작성
  * 열거형(enumerate) 변수 이름은 `_e`로 끝나도록 작성
  * Active-low signal 이름은 `_n`로 끝나도록 작성
* 상수 이름과 열거된 레이블(enumerated label)은 모두 대문자로 작성
* 클래스 선언 이름은 첫 글자는 대문자, 나머지는 소문자로 작성
* 키워드와 충돌되지 않는 이름을 선택


## Corrected Version

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




## Gotcha 2: Implicit net declarations

> *Gotcha*: 설계에서 커넥션에 있는 오타(typo)가 컴파일러로 잡히지 않았고, 시뮬레이션에서 기능 문제로만 확인된다.
>
> *Synopsys*: 잘못 타이핑 된 식별자(identifier)에 대해 문법 오류(syntax error)가 나타나는 대신 암묵적 넷(implicit net) 이 삽입될 수 있다.

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

* 미선언 식별자가 연속적 할당문의 좌변에 사용되면 *암묵적 넷 선언이 삽입되고, 에러나 경고가 리포트 되지 않는다*.
    ```SystemVerilog
    logic [7:0] foo;
    assign bar = foo;       // GOTCHA: bar not declared, but no error
    ```

* 미선언 식별자가 모듈의 인스턴스, 인터페이스(interface), 프로그램(program), 또는 프리미티브(primitive)의 커넥션에 사용되면 *암묵적 넷이 삽입되고, 에러나 경고라 리포트 되지 않는다*.


## Gotcha!

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


## How To Avoid This Gotcha Using Verilog

* Emacs의 Verilog 모드와 같은 언어 인식형 에디터(language-aware editor)를 활용하여 Verilog의 넷리스트를 자동으로 넣어준다.


## How To Avoid This Gotcha Using SystemVerilog

* SystemVerilog에는 *닷-네임(dot-name)*과 *닷-스타(dot-star)* 단축어가 제공되어 포트 연결 이름의 반복을 줄여준다.

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
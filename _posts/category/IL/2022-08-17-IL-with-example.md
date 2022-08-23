---
layout: post
title: "IL 코드 알아보기"
date: 2022-08-19
categories: C# IL
tags: IL C#
---

[toc]

----------

# 예시를 통한 IL 코드 알아보기

아주 간단한 예시를 만들고 분석하는 것을 반복함으로, 
C# 코드가 실질적으로 어떻게 IL 코드로 변환되는지 살펴보려 한다.

IL 코드를 실제로 짤 일은 없겠지만, 
문제 코드 분석 또는 최적화 등의 영역에서 C# 코드가 실제로 어떤 IL 코드를 생성하고, 
또 어떻게 동작하는지 확인하는 법 정도는 알아야 될 것 같아, 시간을 들여 보려 한다.

----------

## 함수 실행 구조

몇 가지 예시를 통해 함수 실행 구조를 파악하고, 정리해보자.

우선, 아무 것도 하지 않는 빈 함수를 선언 하고, 생성된 IL 코드도 확인해본다.

**C# 코드**
```csharp
  public static void DoNothingFunction()
  {
  }
```

**IL 코드**
```csharp
// Methods
.method public hidebysig static 
  void DoNothingFunction () cil managed 
{
  // Method begins at RVA 0x2050
  // Header size: 1
  // Code size: 2 (0x2)
  .maxstack 8

  IL_0000: nop
  IL_0001: ret
} // end of method Tester::DoNothingFunction
```

우선, `.maxstack` 구문이 보인다. 정확히 어떤 역할인지 아직 모르나, 우선은 스킵한다.
이후 실행 되는 `nop` 은 의미있는 연산을 수행하지 않는다. 우선은 단순히 함수의 시작점으로 이해해보자.
그 이후 수행되는 `ret` 은 반환 값이 있는 경우, *callee* 의 
스택에서 *caller* 의 스택으로 푸시한다. `ret` 은 문서 아래에서 다시 살펴본다.

----------

### 지역 변수

지역 변수를 할당하는 함수를 살펴보자

```csharp
public static void HasLocalVariable()
{
    int i = 1;
    long l = 1;
    double d = 1;
    float f = 1;
}
```

위 코드는 아래 IL 코드로 컴파일 된다.

```csharp
.method public hidebysig static 
  void HasLocalVariable () cil managed 
{
  // Method begins at RVA 0x2054
  // Header size: 12
  // Code size: 23 (0x17)
  .maxstack 1
  .locals init (
    [0] int32 i,
    [1] int64 l,
    [2] float64 d,
    [3] float32 f
  )

  IL_0000: nop
  IL_0001: ldc.i4.1
  IL_0002: stloc.0
  IL_0003: ldc.i4.1
  IL_0004: conv.i8
  IL_0005: stloc.1
  IL_0006: ldc.r8 1
  IL_000f: stloc.2
  IL_0010: ldc.r4 1
  IL_0015: stloc.3
  IL_0016: ret
} // end of method Tester::HasLocalVariable
```

`.maxstack` 의 값은 위 함수(`DoNothingFunction()`) 다르게 줄었다. 실제로 의미하는 바는 추후에 다시 확인해보자.

우선은 `.locals init {...}` 구문을 보자. 해당 구문 내에서
함수 내에서 사용되는 지역 변수에 대한 초기화를 진행하는 것으로 보인다.
여기서 중요시 봐야될 것은 **인덱스** 가 부여된다는 점이다. 
이후 IL 코드는 해당 인덱스를 기준으로 동작을 실행하고 있다.

그리고 `nop` 구문으로 함수가 시작되고 `ldc` 와 `stloc` 이라는 명령어가 보인다.

각 명령어는 다음 역할을 수행한다.
- `ldc` : 스택에 푸쉬한다.
- `stloc` : 스택의 맨 위를 Pop하고, 타겟 인덱스의 지역 변수 목록에 저장.

`HasLocalVariable()` 함수의 사실상 모든 부분을 설명한 것과 다름 없다.
`HasLocalVariable()` 를 라인 단위로 분석 해보자.

```csharp
.method public hidebysig static 
  void HasLocalVariable () cil managed 
{
  .maxstack 1

  -- 초기화 및 지역 변수 인덱싱 처리
  .locals init (
    [0] int32 i,
    [1] int64 l,
    [2] float64 d,
    [3] float32 f
  )

  IL_0000: nop          -- 함수 시작
  // int i = 1;
  IL_0001: ldc.i4.1     -- 정수 1 을 i4(int32) 로 스택에 푸쉬
  IL_0002: stloc.0      -- 스택을 Pop 하며 0번 인덱스(변수 i) 에 저장.

  // long l = 1;
  IL_0003: ldc.i4.1     -- 정수 1 을 i4(int32) 로 스택에 푸쉬
  IL_0004: conv.i8      -- 현재 스택의 Top 을 i8(int64) 로 변환
  IL_0005: stloc.1      -- 스택을 Pop 하며 1번 인덱스(변수 l) 에 저장.

  // double d = 1;
  IL_0006: ldc.r8 1     -- 실수 1 을 r8(float64) 로 스택에 푸쉬
  IL_000f: stloc.2      -- 스택을 Pop 하며 2번 인덱스(변수 d) 에 저장.

  // float f = 1;
  IL_0010: ldc.r4 1     -- 실수 1 을 r4(float32) 로 스택에 푸쉬
  IL_0015: stloc.3      -- 스택을 Pop 하며 3번 인덱스(변수 f) 에 저장.

  IL_0016: ret          -- 함수 종료
} // end of method Tester::HasLocalVariable
```

----------

### 매개 변수와 반환 값

매개 변수가 포함된 함수를 선언한다.

```csharp
public static void HasParam(int val, ref int val2)
{
    int fooVariable2 = 1;
}
```

이 코드의 IL 코드를 확인해보자.

```csharp
.method public hidebysig static 
  void HasParam (
    int32 val,
    int32& val2
  ) cil managed 
{
  // Method begins at RVA 0x2080
  // Header size: 12
  // Code size: 4 (0x4)
  .maxstack 1
  .locals init (
    [0] int32 fooVariable2
  )

  IL_0000: nop
  IL_0001: ldc.i4.1
  IL_0002: stloc.0
  IL_0003: ret
} // end of method Tester::HasParam
```

위에서 설명했던 `ldc`, `stloc` 외에 특별한 것은 보이지 않는다.
당연하겠지만, 함수에서 매개 변수를 받더라도 특별한 동작을 하지 않으면 
최소한 함수 내부에선 연관된 IL 코드를 만들지 않고 있다.

반환 값도 한번 확인해보자.

```csharp
public static int HasReturn()
{
    int value = 1;
    return ++value;
}
```

위 함수는 아래의 IL 코드를 생성했다.
```csharp
.method public hidebysig static 
  int32 HasReturn () cil managed 
{
  // Method begins at RVA 0x2090
  // Header size: 12
  // Code size: 13 (0xd)
  .maxstack 2
  .locals init (
    [0] int32 'value',
    [1] int32
  )

  IL_0000: nop
  IL_0001: ldc.i4.1
  IL_0002: stloc.0
  IL_0003: ldloc.0
  IL_0004: ldc.i4.1
  IL_0005: add
  IL_0006: dup
  IL_0007: stloc.0
  IL_0008: stloc.1
  IL_0009: br.s IL_000b

  IL_000b: ldloc.1
  IL_000c: ret
} // end of method Tester::HasReturn
```

아까 보지 못했던 명령어가 몇 개 보인다. 하나씩 살펴보자.

```csharp

  ... 생략
  IL_0001: ldc.i4.1     -- 정수 1 을 int32 로 스택에 푸쉬
  IL_0002: stloc.0      -- 스택을 팝하여 0번 인덱스(value 변수)에 저장.
  IL_0003: ldloc.0      -- 0번 인덱스의 변수를 스택에 푸쉬
  IL_0004: ldc.i4.1     -- 정수 1 을 int32 로 스택에 푸쉬
  IL_0005: add          -- 두 개의 값을 더해, 스택에 푸쉬: Stack 내 변수 {1, 1} 을 더한다.
  IL_0006: dup          -- 스택의 현재 Top을 복사
  IL_0007: stloc.0      -- 스택을 팝하여 0번 인덱스(value 변수)에 저장.
  IL_0008: stloc.1      -- 스택을 팝하여 1번 인덱스(이름 없는 변수)에 저장.
  IL_0009: br.s IL_000b -- 대상 명령(약식) 제어를 전송
  IL_000b: ldloc.1      -- 1번 인덱스의 변수를 스택에 푸쉬
  IL_000c: ret          -- 반환 및 종료
```

`add`, `dup` 은 처음 보지만, 위 주석 정도면 충분할 것 같다.


----------

### 호출자 살펴 보기

함수 내부를 살펴봤다면, 이제 호출자 쪽을 살펴보려 한다.

`Runner()` 함수는 위에서 살펴보았던 함수들을 호출한다.
```csharp
public static void Runner()
{
    int refval = 3;
    HasParam(1, ref refval);

    int localvar = 1;
    HasParam(localvar, ref refval);

    var retval = HasParamAndReturn(10, ref refval);
}
```

IL 코드는 다음과 같다.

```csharp
.method public hidebysig static 
  void Runner () cil managed 
{
  // Method begins at RVA 0x2100
  // Header size: 12
  // Code size: 34 (0x22)
  .maxstack 2
  .locals init (
    [0] int32 refval,
    [1] int32 localvar,
    [2] int32 'retval'
  )

  IL_0000: nop
  IL_0001: ldc.i4.3
  IL_0002: stloc.0
  IL_0003: ldc.i4.1
  IL_0004: ldloca.s 0
  IL_0006: call void Tester::HasParam(int32, int32&)
  IL_000b: nop
  IL_000c: ldc.i4.1
  IL_000d: stloc.1
  IL_000e: ldloc.1
  IL_000f: ldloca.s 0
  IL_0011: call void Tester::HasParam(int32, int32&)
  IL_0016: nop
  IL_0017: ldc.i4.s 10
  IL_0019: ldloca.s 0
  IL_001b: call int32 Tester::HasParamAndReturn(int32, int32&)
  IL_0020: stloc.2
  IL_0021: ret
} // end of method Tester::Runner
```

앞서 살펴보았던 명령어들(`ldc`, `stloc`) 가 보여서 마음이 좀 편안하게 코드를 읽을 수 있겠다.
못보았던 명령어를 포함하여 다시 정리해보자면 아래와 같다.

- `nop` : no operation. 아무것도 하지 않는다.
- `ldc` : 스택에 푸쉬한다.
- `stloc` : 스택의 맨 위를 Pop하고, 타겟 인덱스의 지역 변수 목록에 저장.

- `ldloca` : 특정 인덱스의 지역 변수의 주소를 스택에 로드
- `call <method>` :  함수 호출.

위 코드를 구간별로 살펴보자면 아래와 같다.

```csharp
  IL_0000: nop
  IL_0001: ldc.i4.3     -- 스택에 3(int32) 를 푸쉬
  IL_0002: stloc.0      -- 3을 refval(인덱스 0) 에 할당
  IL_0003: ldc.i4.1     -- 스택에 1(int32) 를 푸쉬
  IL_0004: ldloca.s 0   -- refval(지역 변수)의 주소 스택에 푸시
  IL_0006: call void Tester::HasParam(int32, int32&)
```

특이한 점은 `ref` 키워드의 경우 실제로도 지역 변수의 메모리를 스택에 푸쉬하는 것을 확인 할 수 있다는 점 정도로 보인다. 마지막 `HasParamAndReturn()` 함수를 호출하는 구간을 살펴보자면 다음과 같다.

```csharp
  IL_0016: nop
  IL_0017: ldc.i4.s 10     -- 스택에 10(int32) 를 푸쉬
  IL_0019: ldloca.s 0      -- refval(지역 변수)의 주소 스택에 푸시
  IL_001b: call int32 Tester::HasParamAndReturn(int32, int32&)  -- 함수 호출
  IL_0020: stloc.2         -- 스택의 top 을 retval(인덱스 2) 에 할당
  IL_0021: ret             -- 함수 종료
  
  // HasParamAndReturn 함수 내부 
  .method public hidebysig static 
    int32 HasParamAndReturn (
      int32 val,
      int32& val2
    ) cil managed 
  {
    // Method begins at RVA 0x20ac
    // Header size: 12
    // Code size: 18 (0x12)
    .maxstack 3
    .locals init (
      [0] int32 fooVariable2,
      [1] int32
    )

    IL_0000: nop
    // int fooVariable2 = 100;
    IL_0001: ldc.i4.s 100    -- 스택에 100(int32) 를 푸쉬
    IL_0003: stloc.0         -- 스택을 팝하여 0번 인덱스(fooVariable2) 에 저장.

    // val2 = fooVariable2 + val2;
    IL_0004: ldarg.1         -- argment1(val2)의 주소를 스택에 푸쉬
    IL_0005: ldloc.0         -- 0번 인덱스의 변수를 스택에 푸쉬 
    IL_0006: ldarg.1         -- argment0(val)의 값을 스택에 푸쉬
    IL_0007: ldind.i4        -- int32 값을 간접적으로 스택에 푸쉬
    IL_0008: add             -- 스택 상단의 두 수를 더하여 스택에 푸쉬
    IL_0009: stind.i4        -- 특정 주소에 int32 값을 저장.  

    // return val + num;
    IL_000a: ldarg.0         -- argument0(val)의 값을 스택에 푸쉬 
    IL_000b: ldloc.0         -- 0번 인덱스의 변수를 스택에 푸쉬 
    IL_000c: add             -- 스택 상단의 두 수를 더하여 스택에 푸쉬
    IL_000d: stloc.1         -- 스택을 팝하고 1번 인덱스에 저장.

    // (no C# code)
    IL_000e: br.s IL_0010    -- IL_0010 명령으로 제어 전송
    IL_0010: ldloc.1         -- 1번 변수를 스택에 푸쉬
    IL_0011: ret             -- 함수 종료
  } // end of method Tester::HasParamAndReturn

```

아직 모르는 것이 많이 있지만 (`.maxstack` 이나 `br.s` 등) 우선 IL 코드의 기초적인 부분을 읽어 보았다.
이제 이 다음엔 class, interface, struct 등을 사용한 코드를 분석해보자.
# Week 5

## 실행 환경

- 환경: Windows WSL (Ubuntu Linux)
- 컴파일러: GCC
- 디버거: GDB

## Challenge 1. 메모리 레이아웃 이해하기

### 실습 코드
```c
#include <stdio.h>
#include <stdlib.h>

int initialized_global = 100;
int uninitialized_global;

int main(void)
{
    int local_var = 200;
    int *heap_var = malloc(sizeof *heap_var);
    const char *literal = "Hello, memory!";

    if (heap_var == NULL) {
        perror("malloc");
        return 1;
    }

    *heap_var = 300;

    printf("Data   : %p\n", (void *)&initialized_global);
    printf("BSS    : %p\n", (void *)&uninitialized_global);
    printf("Stack  : %p\n", (void *)&local_var);
    printf("Heap   : %p\n", (void *)heap_var);
    printf("Literal: %p\n", (void *)literal);

    free(heap_var);
    return 0;
}
```

`<stdio.h>`는 `printf()`와 `perror()`, `<stdlib.h>`는 `malloc()`과 `free()`의 선언을 제공한다.

`%p`는 포인터 값을 출력하는 지정자이며, 이에 맞게 주소를 `void *`로 변환하여 전달한다.


### 실행 결과

> 각 변수의 주소를 출력한 실행 결과 캡처 추가

### 메모리 영역별 특징

| 영역 | 저장되는 내용 | 실습 코드의 예 |
|---|---|---|
| Text (`.text`) | 실행할 기계어 명령 | `main()`의 실행 코드 |
| Data (`.data`) | 일반적으로 0이 아닌 값으로 초기화한 전역변수·정적변수 | `initialized_global = 100` |
| BSS (`.bss`) | 초기화하지 않았거나 0으로 초기화한 전역변수·정적변수 | `uninitialized_global` |
| Heap | 동적으로 할당한 메모리 | `malloc()`으로 할당한 정수 공간 |
| Stack | 일반적으로 함수의 지역변수와 호출 관련 정보 | `local_var` |
| Read-only data (`.rodata`) | 문자열 리터럴 등의 읽기 전용 데이터 | `"Hello, memory!"` |

초기화하지 않은 전역변수 `uninitialized_global`은 자동으로 0으로 초기화된다.

문자열 리터럴은 과제에서 Text/Code로 분류했지만, 일반적인 GCC/Linux 실행 파일에서는 `.text`와 구분되는 `.rodata`에 저장된다.

### 포인터 변수와 할당된 공간의 차이

```c
int *heap_var = malloc(sizeof *heap_var);
```

`heap_var` 자체는 지역 포인터 변수이므로 이번 실습에서는 보통 Stack에 저장된다. `malloc()`이 할당한 정수 공간은 Heap에 있으며, 그 주소가 `heap_var`에 저장된다.

| 표현식 | 의미 |
|---|---|
| `heap_var` | 동적으로 할당한 정수 공간의 주소 |
| `&heap_var` | 포인터 변수 자체의 주소 |
| `*heap_var` | 할당된 공간에 저장한 정수 값인 300 |
| `literal` | 문자열 리터럴의 시작 주소 |
| `&literal` | 지역 포인터 변수 `literal` 자체의 주소 |

따라서 Heap 주소를 확인하려면 `&heap_var`가 아니라 `heap_var`를 출력해야 한다.


GDB에서 다음 명령을 실행한다.

```gdb
start
list
```

소스에서 `free(heap_var);`의 줄 번호를 확인하고, 해당 줄에 중단점을 설정한 다음 `continue`로 진행한다.

예를 들어 해당 줄이 26번이면 다음과 같다.

```gdb
break memory_layout.c:26
continue
```

```gdb
p &initialized_global
p &uninitialized_global
p &local_var
p heap_var
p literal
p &main
```


```gdb
info proc mappings
info files
```

`info proc mappings`는 실행 중인 프로세스의 메모리 매핑을 보여 준다. 각 주소가 시작 주소 이상, 끝 주소 미만인 범위에 속하는지 비교한다.

Stack은 `[stack]`, 이번 실습의 작은 동적 할당은 보통 `[heap]` 범위에서 확인할 수 있다. 큰 동적 할당 등은 별도의 매핑에 위치할 수도 있다.

`.data`와 `.bss`는 동일한 쓰기 가능한 매핑에 포함될 수 있으므로, 이 둘을 구분하려면 `info files`의 섹션 주소 범위를 함께 확인한다.

| 주소 | 비교할 범위 |
|---|---|
| `&initialized_global` | `.data` |
| `&uninitialized_global` | `.bss` |
| `&local_var` | `[stack]` |
| `heap_var` | 일반적으로 `[heap]` |
| `literal` | 일반적으로 `.rodata` |
| `&main` | `.text` |

주소는 실행 환경이나 ASLR 설정 등에 따라 달라질 수 있으므로, 같은 실행에서 확인한 주소와 메모리 맵을 비교해야 한다.

### GDB 확인 결과

> 변수 주소, `info proc mappings`, `info files`의 캡처 추가

| 확인 대상 | 실제 주소 | 확인한 영역 |
|---|---|---|
| `initialized_global` | 직접 기록 | 직접 기록 |
| `uninitialized_global` | 직접 기록 | 직접 기록 |
| `local_var` | 직접 기록 | 직접 기록 |
| `heap_var`가 가리키는 공간 | 직접 기록 | 직접 기록 |
| 문자열 리터럴 | 직접 기록 | 직접 기록 |
| `main()` | 직접 기록 | 직접 기록 |

## Challenge 2. 버퍼 오버플로우 실험

### 실습 코드

파일명: `bof.c`

```c
#include <stdio.h>

/* 실습용: 현대 C 표준에서 제거된 gets의 선언 */
extern char *gets(char *);

void secret(void)
{
    printf("secret function called!\n");
}

void vuln(void)
{
    char buf[16];
    gets(buf);
}

int main(void)
{
    vuln();
    return 0;
}
```

`gets()`는 입력 길이를 제한하지 않으며, 버퍼 크기를 인자로 받지 않는다. 현대 C 표준에서는 제거되었으므로 실습 코드에서 명시적으로 선언했다.

해당 함수를 제공하는 glibc 환경에서 실습할 수 있으며, 컴파일 시 위험한 함수라는 경고가 발생할 수 있다.

`secret()`은 정의되어 있지만 호출하지 않으므로 정상 실행에서는 메시지가 출력되지 않는다. 긴 입력만으로 이 함수가 자동으로 호출되는 것은 아니다.

### 컴파일 및 실행 방법

```bash
gcc -g -O0 -fno-stack-protector bof.c -o bof
./bof
```

`-fno-stack-protector`는 스택 손상을 탐지하는 Stack Canary 보호 코드의 삽입을 비활성화한다. 모든 메모리 보호 기능을 해제하는 옵션은 아니다.

### 정상 입력과 긴 입력 비교

정상 입력 예시:

```text
hello
```

16바이트를 초과하는 입력 예시:

```text
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
```

각 입력은 프로그램을 별도로 실행하여 입력한다. 실행 직후 터미널에서 다음 명령을 입력하면 종료 상태를 확인할 수 있다.

```bash
echo $?
```

| 구분 | 정상 입력 | 긴 입력 |
|---|---|---|
| 예시 | `hello` | 영문 `A` 40개 |
| 필요한 공간 | 종료 문자 포함 6바이트 | 종료 문자 포함 41바이트 |
| 버퍼 범위 | 16바이트 안에 저장 | 버퍼 범위를 넘어 기록 |
| 실행 결과 | 일반적으로 정상 종료 | 메모리 손상으로 비정상 종료할 수 있음 |

`gets()`는 개행을 저장하지 않고 문자열 끝에 널 문자 `\0`을 추가한다. 따라서 `char buf[16]`에 저장 가능한 문자열은 최대 15바이트이며, 16바이트의 입력도 종료 문자 때문에 이미 1바이트를 초과한다.

버퍼 범위를 벗어나는 쓰기는 정의되지 않은 동작이므로, 항상 같은 결과나 즉각적인 오류가 발생하는 것은 아니다.

### 실행 결과

정상 입력:

> 정상 입력 실행 결과 캡처 추가

긴 입력:

> 긴 입력 실행 결과 캡처 추가

### GDB로 스택 구조 확인하기

```bash
gdb ./bof
```

```gdb
break vuln
run
list
disassemble vuln
p sizeof(void *)
p &buf
info registers rsp rbp
info frame
```

`gets(buf);`가 실행되기 전에 멈춘 상태에서 확인한다.

아래 설명은 x86-64에서 `push rbp`, `mov rbp, rsp` 형태의 프롤로그를 사용하고, 프레임 설정이 끝난 경우를 기준으로 한다. AT&T 표기에서는 `mov %rsp, %rbp`로 표시된다.

| 위치 | 저장 내용 |
|---|---|
| `rbp`보다 낮은 주소 | `buf` 등의 지역변수 |
| `rbp` | 호출한 함수의 저장된 `rbp` |
| `rbp + 8` | 저장된 반환 주소 |

컴파일 결과에 따라 배치나 패딩이 달라질 수 있으므로 `buf`와 반환 주소 사이의 거리를 고정된 값으로 가정하지 않는다.

프레임 포인터를 사용하지 않는 경우에는 위 배치 대신 `info frame`과 디스어셈블 결과를 기준으로 확인한다.

`info frame`의 `saved rip`는 돌아갈 명령어 주소이며, `Saved registers` 아래의 `rip at ...`는 그 주소가 저장된 메모리 위치이다.

### 입력 전후의 반환 주소 비교

위 프레임 구조를 확인한 경우, 입력 전 주소를 저장하고 메모리를 확인한다.

```gdb
set $buf_start = (char *)&buf
set $ret_slot = (char *)$rbp + 8
p/d $ret_slot - $buf_start
x/48bx $buf_start
x/gx $ret_slot
```

`p/d` 결과는 버퍼 시작점과 반환 주소 저장 위치 사이의 바이트 거리이다.

`x/gx` 출력의 왼쪽은 메모리 위치, 오른쪽은 그곳에 저장된 값이다.

다음에 실행할 줄이 `gets(buf);`일 때 아래 명령을 입력한다.

```gdb
next
```

프로그램이 입력을 기다리면 `hello`를 입력한다. 입력 직후 다음 명령으로 버퍼와 반환 주소를 확인한다.

```gdb
x/48bx $buf_start
x/s $buf_start
x/gx $ret_slot
continue
```

정상 입력에서는 버퍼에 문자열이 저장되고 반환 주소는 유지된다.

이후 `run`으로 다시 실행하고, 중단점에서 `$buf_start`와 `$ret_slot`을 다시 설정한다. 같은 절차로 `A` 40개를 입력한 뒤 메모리를 비교한다.

영문 대문자 `A`의 ASCII 값은 `0x41`이다. 입력이 반환 주소까지 도달하여 8바이트 전체를 덮었다면 해당 위치에서 `0x4141414141414141`을 확인할 수 있다.

실제로 덮였는지는 주소 간 거리와 입력 후 메모리를 근거로 판단한다. 반환 주소가 손상되면 함수가 끝난 뒤 정상적인 호출 위치로 돌아가지 못해 비정상 종료할 수 있다.

### 스택 관찰 결과

> 입력 전후의 `buf`, 저장된 반환 주소, 프로그램 종료 상태 캡처 추가

| 확인 항목 | 실제 관찰 결과 |
|---|---|
| `buf` 시작 주소 | 직접 기록 |
| 반환 주소 저장 위치 | 직접 기록 |
| 두 위치 사이의 거리 | 직접 기록 |
| 정상 입력 전후 반환 주소 | 직접 기록 |
| 긴 입력 전후 반환 주소 | 직접 기록 |
| 긴 입력 후 종료 상태 | 직접 기록 |

### 버퍼 오버플로우가 발생하는 이유

버퍼 오버플로우(Buffer Overflow)는 할당된 버퍼 크기를 초과하여 데이터를 기록하는 현상이다.

이번 코드에서는 `gets()`가 입력 길이를 검사하지 않으므로 `buf` 뒤쪽의 스택 데이터까지 덮을 수 있다. 스택에 저장된 호출 정보나 반환 주소가 손상되면 프로그램의 정상적인 실행 흐름이 깨질 수 있다.

### `gets()` 이외에 주의해야 하는 함수

| 함수 | 위험해지는 경우 | 개선 방법 |
|---|---|---|
| `strcpy(dst, src)` | 복사할 문자열이 목적지보다 클 때 | 종료 문자를 포함한 길이와 목적지 크기를 검사한다. |
| `strcat(dst, src)` | 기존 문자열 뒤에 붙일 공간이 부족할 때 | 기존 길이, 추가 길이, 종료 문자 공간을 함께 확인한다. |
| `sprintf(buf, ...)` | 출력 결과가 버퍼보다 클 때 | 버퍼 크기를 지정하는 `snprintf()`를 사용한다. |
| `scanf("%s", buf)` | 입력 폭을 지정하지 않아 긴 문자열을 받을 때 | `char buf[16]`이면 `%15s`처럼 입력 폭을 제한한다. |
| `memcpy(dst, src, n)` | 복사 크기 `n`이 목적지 공간보다 클 때 | 읽을 수 있는 원본 범위와 쓸 수 있는 목적지 범위를 검사한다. |

`strncpy()`도 항상 널 종료를 보장하는 것은 아니므로 무조건 안전한 대체 함수로 볼 수 없다.

### 입력 크기를 제한한 코드

```c
void vuln(void)
{
    char buf[16];

    if (fgets(buf, sizeof buf, stdin) == NULL) {
        return;
    }
}
```

`fgets()`는 지정한 크기보다 하나 적은 수의 문자까지 읽고 끝에 널 문자를 붙인다. 위 코드에서는 최대 15개 문자를 저장하므로 버퍼 크기를 넘지 않는다.

다만 개행이 버퍼에 포함될 수 있고, 한 번에 읽지 못한 입력은 입력 스트림에 남을 수 있다.

## 실습에 사용한 GDB 명령어

| 명령어 | 기능 |
|---|---|
| `start` | 프로그램을 시작하고 `main()` 시작 부근에서 멈춘다. |
| `break` | 함수 또는 소스 줄에 중단점을 설정한다. |
| `run` | 프로그램을 처음부터 실행한다. |
| `next` | 소스 코드 한 줄을 실행하며, 함수 내부로 들어가지 않는다. |
| `print` 또는 `p` | 변수나 표현식의 값을 출력한다. |
| `info proc mappings` | 실행 중인 프로세스의 메모리 매핑을 확인한다. |
| `info files` | 실행 파일의 섹션 주소 범위 등을 확인한다. |
| `info registers` | CPU 레지스터 값을 확인한다. |
| `info frame` | 현재 스택 프레임과 저장된 반환 주소 등을 확인한다. |
| `disassemble vuln` | `vuln()`의 어셈블리 코드를 확인한다. |
| `x/s 주소` | 해당 주소를 문자열로 읽는다. |
| `x/48bx 주소` | 해당 주소부터 1바이트씩 48개를 16진수로 읽는다. |
| `x/gx 주소` | 해당 주소의 8바이트를 16진수로 읽는다. |
| `continue` | 다음 중단점 또는 프로그램 종료까지 진행한다. |
| `quit` | GDB를 종료한다. |

변수와 메모리는 프로그램이 종료되기 전에 실행이 멈춘 상태에서 확인한다.
x/s 주소	해당 주소를 문자열로 읽는다.
x/48bx 주소	해당 주소부터 1바이트씩 48개를 16진수로 읽는다.
x/gx 주소	해당 주소의 8바이트를 16진수로 읽는다.
continue	다음 중단점 또는 프로그램 종료까지 진행한다.
quit	GDB를 종료한다.


변수와 메모리는 프로그램이 종료되기 전에 실행이 멈춘 상태에서 확인한다.

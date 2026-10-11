# Week 05-A. gcc & Libraries

## 1. 셸 실행 vs C 빌드

```bash
gcc main.c logutil.c -o logcount
./logcount data/app.log
echo $?
```
- **셸**: bash가 스크립트를 **실행하면서** 읽음
- **C**: gcc가 프로그램을 **실행 전에** 빌드함
- **공통점**: 인자, stdout, stderr, 종료상태를 둘 다 가짐 (4주차 개념이 그대로 이어짐)

## 2. 이번 실습 대상: log counter 계약

```
INFO boot
ERROR disk
INFO no ERROR detected
WARN memory
ERROR network

Expected stdout: 2
```
- **정확히 `ERROR ` (공백 포함) 접두사로 시작하는 줄만** 카운트
- 성공 출력: 정수 하나 + 줄바꿈

### 출력과 종료상태

| 입력 조건 | stdout | status |
|---|---|---|
| 매치 2줄 | 2 | 0 |
| 매치 없음 | 0 | 0 |
| 인자 누락 | empty | 2 |
| 파일 누락 | empty | 3 |

- **count 0도 정상 성공 결과**
- 잘못된 인자/읽을 수 없는 파일은 오류
- 이 계약은 **빌드가 성공한 이후에** 적용됨

### 프로젝트 파일 구성
| 파일 | 역할 |
|---|---|
| main.c | 파일 입력, 카운팅 루프, 출력 |
| logutil.h | 공유 선언과 접두사 매크로 |
| logutil.c | 접두사 비교 구현 |
| check.sh | 예상 결과와 비교 검사 |

### 함수 호출 구조
```c
int count = 0;
while (fgets(line, sizeof line, fp) != NULL) {
    count += is_error(line);
}
printf("%d\n", count);
```
- 호출자가 한 줄씩 전달 → 함수가 매치 시 1 반환 → 호출자가 count에 누적

## 3. 소스 파일과 빌드 산출물

| 편집하는 입력 | 생성되는 출력 |
|---|---|
| main.c, logutil.c | main.o, logutil.o |
| logutil.h | liblogutil.a |
| README.md, check.sh | logcount |

생성된 파일은 `build/`에 보관 → **소스 vs 산출물 분리** (4주차 실습과 동일한 원칙)

## 4. gcc 빌드 4단계 (핵심!)

| 단계 | 전형적 출력 | 멈추는 옵션 |
|---|---|---|
| Preprocess(전처리) | source.i | `-E` |
| Compile(컴파일) | source.s | `-S` |
| Assemble(어셈블) | source.o | `-c` |
| Link(링크) | 실행파일 | (멈추는 옵션 없음) |

- 각 단계는 이전 표현을 **소비**함
- gcc 한 명령으로 여러 단계를 한번에 실행 가능
- **멈추는 옵션이 결과물을 결정**함

### 4-1. 전처리 (Preprocessing)
```c
#define ERROR_PREFIX "ERROR "
strncmp(line, ERROR_PREFIX, strlen(ERROR_PREFIX))
/* 매크로 확장 후 */
strncmp(line, "ERROR ", strlen("ERROR "))
```
- 헤더 내용이 번역단위(translation unit)의 일부가 됨
- 매크로는 컴파일 **전에** 확장됨
- **디스크의 원본 소스 파일은 바뀌지 않음**

### 4-2. 컴파일과 어셈블
```bash
gcc -S logutil.c -o build/logutil.s
gcc -c logutil.c -o build/logutil.o
```
- 어셈블리 텍스트는 대상 아키텍처에 의존
- 오브젝트 파일은 기계어 코드 + 링킹 정보를 담음
- **오브젝트 파일은 아직 최종 프로그램이 아님**

### 4-3. 링크
```bash
gcc build/main.o build/logutil.o -o build/logcount
./build/logcount data/app.log
# 2
```
- 주어진 오브젝트 파일들을 결합
- `is_error`에 대한 참조를 해결(resolve)
- `-o`로 지정된 이름의 실행파일 생성

### gcc 명령 읽는 법
```bash
gcc -c main.c -o build/main.o              # 입력 접미사가 시작점, -c가 멈추는 지점
gcc build/main.o build/logutil.o -o build/logcount   # 오브젝트→실행파일
gcc main.c logutil.c -o build/logcount     # 소스→바로 실행파일 (중간 단계 생략)
```
- 입력 확장자가 시작점을 결정, `-E`/`-S`/`-c`가 멈출 지점을 결정, `-o`가 출력 이름 결정

## 5. 선언(Declaration) vs 정의(Definition)

```c
/* logutil.h: 선언 */
int is_error(const char *line);

/* logutil.c: 정의 */
int is_error(const char *line) {
    return strncmp(line, ERROR_PREFIX, strlen(ERROR_PREFIX)) == 0;
}
```
- **선언**: 이름, 매개변수 타입, 반환 타입
- **정의**: 함수 본문
- 함수 **호출**에는 선언과, 링크된 구현체가 모두 필요함

### 공유 헤더
```c
/* main.c */
#include "logutil.h"

/* logutil.c */
#include <string.h>
#include "logutil.h"
```
- main.c와 logutil.c가 **같은 선언**을 공유함 (두 번역단위가 동일한 계약을 씀)

### 헤더 가드
```c
#ifndef LOGUTIL_H
#define LOGUTIL_H
#define ERROR_PREFIX "ERROR "
int is_error(const char *line);
#endif
```
- 번역단위당 가드된 내용이 **한 번만** 처리되도록 함
- 헤더마다 **고유한 매크로 이름** 사용
- (반복되는 호환 가능한 함수 선언 자체는 단독으로 유효할 수 있음)

### 접두사 구현
```c
return strncmp(line, ERROR_PREFIX, strlen(ERROR_PREFIX)) == 0;
```
| 입력 | 결과 |
|---|---|
| "ERROR disk" | 1 |
| "ERRORish text" | 0 |
| "INFO ERROR message" | 0 |

- `strncmp`는 줄의 **시작 부분**만 비교
- `strlen`이 접두사 길이를 제공
- 동등 비교식이 0 또는 1을 만들어냄

## 6. 분리 컴파일 (Separate Compilation)

```bash
mkdir -p build
gcc -std=c11 -Wall -Wextra -I. -c main.c -o build/main.o
gcc -std=c11 -Wall -Wextra -I. -c logutil.c -o build/logutil.o
gcc build/main.o build/logutil.o -o build/logcount
```
- 각 소스를 **자신의 오브젝트로** 컴파일
- 컴파일 성공 후 오브젝트들을 링크
- 결과를 비교하려면 **같은 빌드 플래그**를 유지해야 함

### 이 프로젝트에서 쓴 플래그

| 옵션 | 목적 |
|---|---|
| `-std=c11` | C 언어 표준 선택 |
| `-Wall -Wextra` | 경고 그룹 활성화 |
| `-I.` | 헤더를 찾을 디렉토리 검색 |
| `-c` / `-o` | 링킹 전에 멈춤 / 출력 이름 지정 |

- 작고 일관된 빌드 옵션 세트 사용
- **경고는 리뷰를 돕지만 정확성을 증명하진 않음**

## 7. 오브젝트 파일의 심볼

```bash
nm build/main.o
                U is_error

nm build/logutil.o
...             T is_error
```
- `main.o`는 `is_error`를 **참조**함 (`U` = undefined)
- `logutil.o`는 `is_error`를 **정의**함 (`T` = defined in text section)
- 심볼 주소와 부가 항목은 환경마다 다를 수 있음

### 구현체가 없을 때
```bash
gcc build/main.o -o build/missing
# undefined reference to `is_error'

gcc build/main.o build/logutil.o -o build/logcount   # 정상
```
- main.c는 선언만으로 컴파일됨
- main.o만 링크하면 `is_error`를 찾을 수 없음
- 정의를 제공하는 오브젝트를 추가해야 함

## 8. 실패 단계 파악하기

| 관찰 증상 | 단계 | 첫 확인 지점 |
|---|---|---|
| "expected semicolon" 등 | 컴파일(Compilation) | 진단 메시지 근처의 C 소스 |
| "undefined reference" | 링킹(Linking) | 오브젝트와 심볼 이름 |
| count가 3인데 2가 맞음 | 프로그램 동작(Program behavior) | 접두사 규칙과 구현 |

- **실패한 명령이 단계를 알려줌**
- 첫 유용한 진단이 원인 범위를 좁혀줌
- 단계마다 다른 수정 방법이 필요함

### 빌드는 성공했지만 결과가 틀린 경우
```c
/* 명시된 접두사 규칙에 맞음 */
strncmp(line, "ERROR ", 6) == 0

/* 명시된 접두사 규칙에 틀림 */
strstr(line, "ERROR") != NULL

/* 입력: "INFO no ERROR detected" */
```
- "아무 곳에서나 검색"은 "접두사 매칭"과 다름
- **두 구현 모두 컴파일은 됨** (문법 오류가 아님!)
- **차이를 드러내는 입력**(discriminating input)이 필요함 → 4주차의 "텍스트 매치 ≠ 의미론적 진단"과 같은 교훈

## 9. 소스 편집과 재빌드

```bash
cp variants/logutil_contains.c logutil.c
./build/logcount data/app.log
# 여전히 이전에 빌드된 프로그램을 사용함!

gcc -I. -c logutil.c -o build/logutil.o
gcc build/main.o build/logutil.o -o build/logcount
```
- **소스 편집은 기존 오브젝트를 자동 갱신하지 않음**
- **갱신된 오브젝트도 기존 실행파일을 자동 갱신하지 않음**
- 재컴파일 → 재링크 → 재실행 순서 필요

### 변경과 재빌드 대응표

| 변경된 입력 | 재빌드할 오브젝트 | 마지막 단계 |
|---|---|---|
| logutil.c 본문 | logutil.o | 재링크 |
| main.c | main.o | 재링크 |
| logutil.h | main.o **그리고** logutil.o | 재링크 |

- 변경에 영향받은 소스 파일만 재컴파일
- 입력 오브젝트나 아카이브가 바뀌면 재링크
- **헤더 변경은 여러 소스 파일에 영향을 줄 수 있음**

### 재빌드 체크 예제
```
A. ./build/logcount data/app.log
B. gcc build/main.o build/logutil.o -o build/logcount
C. gcc -E logutil.c -o build/logutil.i
```
(logutil.c만 변경됨, 헤더와 main.c는 안 바뀜 → new logutil.o는 있지만 logcount는 낡음)

| 명령 | 결과 |
|---|---|
| B | 새 오브젝트로 실행파일 **재링크** |
| A | **낡은** 실행파일을 실행 |
| C | 전처리된 텍스트만 생성 |

## 10. 정적 라이브러리 아카이브 (Static Library)

- 아카이브는 **링킹용 오브젝트 파일들을 묶은 것**
- 관례: `libNAME.a`
- **아카이브는 빌드 입력이지 실행 가능한 프로그램이 아님**

### 아카이브 생성
```bash
ar rcs build/liblogutil.a build/logutil.o
ar t build/liblogutil.a
# logutil.o
```
| 플래그 | 의미 |
|---|---|
| `r` | 멤버 삽입 또는 교체 |
| `c` | 생성 경고 없이 생성 |
| `s` | 심볼 인덱스 기록 |
| `t` | 아카이브 멤버 목록 표시 |

### 아카이브 링크
```bash
gcc build/main.o build/liblogutil.a -o build/logcount_static
bash check.sh ./build/logcount_static
# 7 passed, 0 failed
```
- 아카이브를 직접 경로로 지정하면 선택된 파일이 명확해짐
- **호출자가 아카이브보다 앞에** 와야 함
- 결과 프로그램은 동일한 입력 계약을 사용

### 라이브러리 검색 옵션
```bash
gcc build/main.o -Lbuild -llogutil -o build/logcount
# 또는 명시적 아카이브 선택
gcc build/main.o build/liblogutil.a -o build/logcount_static
```
| 옵션 | 의미 |
|---|---|
| `-L경로` | 링크 타임 검색 디렉토리 추가 |
| `-lNAME` | NAME이라는 이름의 라이브러리 요청 |

- 정적/공유 라이브러리가 둘 다 있으면 **공유 라이브러리가 우선될 수 있음**

### 정적 링킹의 범위
- 우리 아카이브가 로그 유틸리티 코드를 제공
- 시스템 라이브러리는 여전히 공유될 수 있음
- **실행파일은 특정 대상 환경에 종속됨**

## 11. 정적 vs 공유 라이브러리

| 질문 | 정적 아카이브 | 공유 오브젝트 |
|---|---|---|
| 링크 시 사용하는 라이브러리 | `.a` | `.so` |
| 런타임에 자체 라이브러리 파일 필요 | 불필요 (코드가 실행파일에 포함됨) | **필요** |
| 라이브러리 코드 변경 시 | 아카이브 갱신 후 재링크 | 호환성과 로더(loader)가 중요 |

- 실행파일은 **공유 오브젝트에 대한 의존성을 유지**함
- 런타임 로더가 호환되는 라이브러리를 **찾아야** 함
- **빌드 성공과 런타임 사용 가능 여부는 별개의 확인 사항**

### 링크 타임 경로 vs 런타임 경로
```bash
gcc build/main.o -Lbuild -llogutil -o build/logcount_shared

LD_LIBRARY_PATH="$PWD/build" \
    ./build/logcount_shared data/app.log
```
- **링크 타임**: `-Lbuild`가 `build/`에서 `liblogutil.so`를 찾음
- **런타임**: `LD_LIBRARY_PATH`가 공유 라이브러리를 찾음
- `"$PWD/build"`: 이 프로젝트 `build/`의 절대경로

## 12. README와 재현 가능한 빌드

- **환경**: gcc와 ar이 설치된 Ubuntu
- **프로젝트 루트**: 빌드 명령과 라이브러리 경로 명시
- **예상 결과**: `data/app.log` → 2, 종료상태 0

## 13. Git과 생성 파일

```bash
# .gitignore
build/

git diff -- logutil.c logutil.h
git status --short
```
- 소스, 헤더, 체크 스크립트, 빌드 지침을 추적
- 이 프로젝트에서는 `build/`를 무시
- 변경을 기록하기 전 diff를 검토

## 14. 빌드와 테스트 증거

- **빌드**: main.o + logutil.o → logcount 생성
- **테스트**: `data/app.log` → 2, 종료상태 0
- **접두사 검사**: `is_error("INFO ERROR message")` → 0 반환

## 15. Make를 위한 준비 (다음 주 예고)

의존관계 정리:
- `main.o`는 `main.c`와 `logutil.h`에 의존
- `logutil.o`는 `logutil.c`와 `logutil.h`에 의존
- `logcount`는 두 오브젝트 모두에 의존

→ 이 의존관계 표가 다음 주 `make`/Makefile의 기반이 됨

## Review (복습 목표)
- 소스, 오브젝트, 아카이브, 실행파일을 구별할 수 있다
- 선언 vs 정의를 설명할 수 있다
- 컴파일/링킹/동작 실패를 찾아낼 수 있다
- 변경된 입력으로부터 재빌드 단계를 선택할 수 있다

## 참고 자료 (Week 05)
- GCC output stages and linking options
- GNU Binutils ar
- GNU C Preprocessor header guards
- Original course sample and supplied input data

## Week 05-A 핵심 요약

> gcc 빌드는 **전처리(-E) → 컴파일(-S) → 어셈블(-c) → 링크**의 4단계를 거치며, 각 단계는 이전 표현을 소비하고 멈추는 옵션이 산출물을 결정한다. 헤더는 **선언**만 담고 `.c` 파일이 **정의**를 담으며, 함수 호출은 선언과 링크된 구현체가 모두 필요하다 — 구현체가 없으면 링크 단계에서 "undefined reference" 오류가 난다. **소스 편집은 기존 오브젝트/실행파일을 자동으로 갱신하지 않으므로** 재컴파일→재링크가 필요하다. 정적 라이브러리(`.a`)는 코드를 실행파일에 통째로 넣어 런타임에 별도 파일이 필요 없지만, 공유 라이브러리(`.so`)는 런타임에 `LD_LIBRARY_PATH` 등으로 찾아야 하는 의존성을 남긴다. 빌드가 성공했다고 프로그램이 옳다는 뜻은 아니며(예: `strstr` vs `strncmp` 둘 다 컴파일되지만 의미가 다름), 차이를 드러내는 입력으로 검증해야 한다.

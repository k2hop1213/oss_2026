# Week 04-A. Shell Scripting (이론)

## 1. 명령에서 스크립트로

명령을 텍스트 파일에 모아두면, bash가 순서대로 읽어 실행함. 변수와 인자를 쓰면 파일 하나를 재사용 가능하게 만들 수 있음.

```bash
#!/usr/bin/env bash
# Print a fixed message.
echo "hello"

# Blank lines improve reading.
printf '%s\n' "log check"
```
- 형식: 평문 UTF-8 텍스트, `#`로 시작하는 줄은 주석
- 명령 텍스트는 프롬프트에서 치는 것과 동일함, 스크립트는 **실행 순서를 보존**함

## 2. 셸은 인터프리터(interpreter)

- 인터프리터는 프로그램 명령을 읽고 실행함. Bash는 대화형 명령과 스크립트 파일 모두 해석
- 셸마다 지원 문법이 다를 수 있음

### 셔뱅(Shebang)
```bash
#!/usr/bin/env bash
```
- `#!` = 셔뱅 표시, `/usr/bin/env` = PATH로 명령을 찾음, `bash` = 실행할 인터프리터
- **`.sh` 확장자 자체는 Bash를 선택하는 게 아님** (그냥 관례)

### 실행 세 가지 방법

| 명령 | 인터프리터 출처 | 실행 권한 |
|---|---|---|
| `bash hello.sh` | 명령줄의 bash | 불필요 |
| `./hello.sh` | 첫 줄의 셔뱅 | **필요** (`chmod +x`) |
| `sh hello.sh` | 명령줄의 sh | 불필요 |

`bash hello.sh`로 실행하면 셔뱅 줄은 무시됨(선택되지 않음).

## 3. 변수(Variable)

```bash
name="log check"
count=3
file=data/events.log

printf '%s\n' "$name"
```
- `=` 주변에 **공백 금지**, `$`로 값을 읽음

### 셸 변수 vs 환경 변수
```bash
COURSE=OSS
export COURSE
bash -c 'echo "$COURSE"'   # OSS
```
- 셸 변수는 해당 셸에만 속함
- `export`는 **이후 시작되는 자식 프로세스에** 전달함. 자식의 변경은 부모에 자동 반영 안 됨

### 작은따옴표 vs 큰따옴표 (⚠️ 실수 포인트)

| 텍스트 | 의미 | name=log일 때 |
|---|---|---|
| `"$name"` | 변수 확장 | log |
| `'$name'` | 문자 그대로 | $name |
| `"result: $name"` | 문장 안에서 확장 | result: log |

- bash는 명령 실행 **전에 먼저** 변수를 확장함. 호출된 명령은 완성된 인자만 받음
- 큰따옴표는 공백 포함 파일명을 **하나의 인자**로 보존 (`file="data/day one.log"`)

### 중괄호로 변수 감싸기
```bash
count=3
report="${count}_errors.txt"   # 3_errors.txt
```
`${name}`은 변수의 경계를 명확히 표시함 (문자/숫자 바로 옆에 변수 붙일 때 필요)

## 4. 위치 인자(Positional Arguments)

| 변수 | 의미 |
|---|---|
| `$0` | 스크립트 호출 이름(경로) 자체 |
| `$1`, `$2` | 첫째, 둘째 인자 |
| `$#` | 전달된 인자의 개수 |
| `"$@"` | 각 인자의 경계를 보존한 전체 목록 |

```bash
printf 'count: %s\n' "$#"
printf '<%s>\n' "$@"
# bash show_all.sh A "day one"
# count: 2 / <A> / <day one>
```

### 인자 vs 표준 입력
- 인자는 명령이 시작될 때 공급됨
- `read -r name`은 **실행 중에** 표준입력을 소비함

## 5. 명령 치환(Command Substitution)과 산술

```bash
count=$(grep -c ERROR "$file")   # 명령 출력으로 자기 자신을 대체
next=$((count + 1))              # 정수 산술
echo "$((3 / 2))"                # 1 (정수 나눗셈)
```
- `$( )` 출력의 뒤따르는 개행문자는 제거됨

## 6. 표준출력 vs 표준에러, 종료 상태

```bash
printf '%s\n' "checking $file"     # stdout
printf '%s\n' "missing file" >&2   # stderr
```
- `>` 는 stdout, `2>` 는 stderr 리다이렉션. 각각 독립적으로 확인 가능

### 종료 상태(Exit Status)
```bash
grep -q ERROR data/events.log
printf 'status: %s\n' "$?"   # 0 = 성공(관례)

grep -q ERROR data/clean.log
printf 'status: %s\n' "$?"   # 1 = 실패
```
- `$?`는 **바로 직전 명령**의 상태. **다음 명령 실행 전에 즉시 저장**해야 함
```bash
grep -q ERROR data/clean.log
status=$?                     # 올바름: 바로 저장
printf 'status: %s\n' "$status"
```
- 상태는 파일이 아니라 **명령을 영구적으로 설명**하는 값

### 한 명령의 세 가지 결과

| 정보 | 목적 |
|---|---|
| 표준출력 | 데이터 |
| 표준에러 | 진단 |
| 종료상태 | 성공 신호 |

## 7. if 구조

```bash
if [ "$#" -eq 1 ]; then
    printf '%s\n' "one file argument"
else
    printf '%s\n' "usage: check FILE" >&2
fi
```
- `if`는 **명령의 종료 상태**를 평가함 (조건식이 아니라 명령!)
- `grep -q`는 출력 없이 매치 여부만 상태로 보고

### `[ ]` 테스트 명령

| 테스트 | 의미 |
|---|---|
| `-eq -ne -lt -le -gt -ge` | 숫자 비교(같음/다름/작음/작거나같음/큼/크거나같음) |
| `= !=` | 문자열 같음/다름 |
| `-z -n` | 빈 문자열 / 비어있지 않음 |
| `-f -d -e -r -w -x -s` | 일반파일/디렉토리/존재/읽기/쓰기/실행가능/크기>0 |

- `test`와 `[ ]`는 같은 검사, **`[ ]` 안 공백 필수**

### elif
```bash
if [ "$count" -eq 0 ]; then echo "none"
elif [ "$count" -lt 5 ]; then echo "small"
else echo "many"
fi
```

### 처리 전 검증
```bash
if [ ! -f "$file" ]; then
    printf 'not a file: %s\n' "$file" >&2
    exit 2
fi
```
- `!`는 테스트 반전, `exit 2`는 nonzero 상태로 스크립트 중단

## 8. 명령 리스트: `;`, `&&`, `||`

| 연산자 | 동작 |
|---|---|
| `;` | 상태와 무관하게 이어서 실행 |
| `&&` | 상태 0(성공)일 때만 이어서 실행 |
| `\|\|` | nonzero(실패)일 때만 이어서 실행 |

## 9. 반복문

### for
```bash
for file in data/events.log "data/day one.log"; do
    count=$(grep -c ERROR "$file")
    printf '%s: %s\n' "$file" "$count"
done
```
- 따옴표는 공백 포함 항목을 하나로 유지, 사용할 때도 `"$file"`로 다시 감싸야 함

### while / until
```bash
n=1
while [ "$n" -le 3 ]; do
    echo "$n"
    n=$((n + 1))
done
```
- `for`는 목록 순회, `while`은 조건이 성공하는 동안, `until`은 실패하는 동안 반복

### continue / break
```bash
for n in 1 2 3 4; do
    if [ "$n" -eq 2 ]; then continue; fi
    if [ "$n" -eq 4 ]; then break; fi
    echo "$n"
done
# 1
# 3
```
- `continue`: 현재 반복의 나머지 건너뜀 / `break`: 가장 안쪽 반복문 종료 / `exit`: 스크립트 자체 종료

## 10. 함수(Function)

```bash
count_errors() {
    local file=$1
    if [ ! -f "$file" ]; then
        printf 'not a file: %s\n' "$file" >&2
        return 2
    fi
    grep -c ERROR "$file"
}
count=$(count_errors data/events.log)
```
- `local`은 변수를 함수 범위로 제한
- 함수 **출력**(stdout)은 명령 치환으로 캡처, **`return`**은 별도의 숫자 상태 제공

## 11. 문법 검사와 실행 추적

```bash
bash -n script.sh   # 실행 없이 문법만 검사
bash -x script.sh   # 실행하며 stderr에 명령 추적
```
- 문법 검사는 **결과가 맞다는 증명이 아님**. 실제 결과를 기대값과 비교해야 함

## 12. 흔한 오독(誤讀) 사례

| 코드 | 잘못된 해석 | 실제 의미 |
|---|---|---|
| `$(grep -c ERROR file)` | 매치된 텍스트 나열 | count 출력값 저장 |
| `[ "$count" -eq 0 ]` | 대괄호는 장식 | 숫자 테스트 실행 |
| printf 뒤의 `$?` | grep의 상태 | printf의 상태 |
| `"$@"` | 하나로 합쳐진 문자열 | 원래 인자들 분리 유지 |

## Week 04-A 핵심 요약

> 셸 스크립트는 셔뱅으로 인터프리터를 지정하고, 변수는 `=`로 저장·`$`로 읽으며 큰따옴표는 확장·작은따옴표는 문자 그대로 보존한다. 모든 명령은 표준출력·표준에러·종료상태 세 가지를 만들어내며, `if`/`[ ]`는 종료상태로 분기하고 `for`/`while`은 반복하며 함수는 `local`로 범위를 제한하고 `return`으로 상태를, 출력으로 값을 전달한다.

---
---


---

# Week 04-B. Shell Script Practice (실습)

## 1. 목표: 재사용 가능한 로그 도구

**정의된 입력, 정의된 출력, 관찰 가능한 실패**
- 입력: 읽을 수 있는 로그 파일 1개
- 출력: 표준출력에 에러 개수 1개
- 실패: 표준에러 메시지 + nonzero 상태
- 증거: 반복 가능한 명령 + 로컬 Git 히스토리

## 2. 환경 준비

```bash
curl -fL -o setup_w04.sh https://gist.githubusercontent.com/Kyungho-Won/.../setup_w04.sh
bash setup_w04.sh
cd oss-w04
ls   # data  reports  solutions  starter
```

### 결정론적 테스트 데이터

| 파일 | 특징 | 예상 count |
|---|---|---|
| data/events.log | 8줄, ERROR 3개 | 3 |
| data/clean.log | INFO 2줄 | 0 |
| data/empty.log | 빈 파일 | 0 |
| data/day one.log | ERROR와 INFO | 1 |

## 3. 코드 전에 입력 확인

```bash
grep -n 'ERROR' data/events.log     # 정확한 대문자 매칭: 3줄
grep -ci 'error' data/events.log    # 대소문자 무시: 4줄
```
→ **텍스트 매칭이 사건(incident)을 정확히 분류해주는 건 아님**

## 4. count_errors.sh 완성본

```bash
#!/bin/bash
if [ "$#" -ne 1 ]; then
  echo "Usage: bash count_errors.sh FILE" >&2
  exit 1
fi
if [ ! -f "$1" ] || [ ! -r "$1" ]; then
  echo "Error: readable file required" >&2
  exit 1
fi

count=$(grep -c 'ERROR' "$1")
status=$?
if [ "$status" -gt 1 ]; then
  echo "Error: search failed" >&2
  exit 1
fi
echo "$count"
exit 0
```

### 단계별 설계 이유
① **인자 개수 검증** — `$1`에 접근하기 전에 먼저 걸러냄
② **읽기 가능한 일반 파일 검증** — `-f`(일반 파일), `-r`(읽기 권한), 따옴표로 공백 경로 보존
③ **출력과 상태를 즉시 캡처** — `status=$?`를 바로 다음 줄에서 저장
④ **계약된 결과 출력** — stdout에는 숫자만, 명시적 `exit 0`으로 계약 문서화

### grep 종료상태의 함정 (핵심!)

| 상태 | 의미 |
|---|---|
| 0 | 매치 있음 |
| **1** | **매치 없음 (정상적인 결과!)** |
| 2 이상 | 진짜 오류 |

- **count와 status는 서로 다른 정보**. count가 0인 것도 유효한 출력
- 그래서 실패 판정은 `-gt 1` (1보다 큼만), `-ge 1`로 쓰면 정상적인 "0개 매치" 파일까지 실패 처리해버림

### 테스트 매트릭스

| 입력 | stdout | status |
|---|---|---|
| events.log | 3 | 0 |
| clean.log | 0 | 0 |
| empty.log | 0 | 0 |
| day one.log | 1 | 0 |
| missing.log | (없음) | 1 |

### 흔한 실수: 상태값 덮어쓰기
```bash
count=$(grep -c 'ERROR' "$1")
echo "search completed"   # 이 echo가 $?를 덮어씀!
status=$?                 # grep이 아니라 echo의 상태가 저장됨
```
→ `$?`는 **바로 다음 줄에서, 다른 명령 전에** 저장해야 함

## 5. summarize.sh — 여러 파일로 확장

**요구사항**: 입력 1개 이상, 요약 출력 **전에 모든 입력 검증**, 각 파일마다 count_errors.sh 호출

```bash
#!/bin/bash
if [ "$#" -lt 1 ]; then
  echo "Usage: bash summarize.sh FILE..." >&2
  exit 1
fi

for file in "$@"; do
  if [ ! -f "$file" ] || [ ! -r "$file" ]; then
    echo "Error: readable file required: $file" >&2
    exit 1
  fi
done

for file in "$@"; do
  count=$(bash ./count_errors.sh "$file")
  status=$?
  if [ "$status" -ne 0 ]; then
    exit 1
  fi
  echo "$file: $count"
done
exit 0
```
- `"$@"`가 원래 각 인자를 보존
- **모든 검사를 통과하기 전에는 요약 줄이 하나도 안 나옴** (부분 실패 방지)

### ⚠️ 리다이렉션이 검증 전에 파일을 비워버림
```bash
bash summarize.sh data/missing.log > reports/summary.txt
# 셸이 summary.txt를 먼저 열고 비워버림(truncate)! 검증 실패와 무관하게

# 안전한 패턴: 임시 파일 + 성공 시에만 이동
if bash summarize.sh data/events.log > reports/tmp.txt; then
  mv reports/tmp.txt reports/summary.txt
fi
```

## 6. verify_counts.sh — 자동 검증

```bash
#!/bin/bash
failures=0
mkdir -p reports || exit 1

check_count() {
  bash ./count_errors.sh "$1" > reports/check.out 2> reports/check.err
  actual_status=$?
  actual_output=$(cat reports/check.out)
  if [ "$actual_status" -eq "$3" ] && [ "$actual_output" = "$2" ]; then
    echo "PASS $1"
  else
    echo "FAIL $1: expected output=<$2> status=$3; actual output=<$actual_output> status=$actual_status"
    failures=1
  fi
}

check_count data/events.log 3 0
check_count data/clean.log 0 0
check_count data/empty.log 0 0
check_count 'data/day one.log' 1 0
check_count data/missing.log '' 1

exit "$failures"
```

### 비교 방식

| 대상 | 비교 방법 |
|---|---|
| 종료상태 | 숫자 비교 (`-eq`) |
| 캡처된 stdout | **문자열 비교** (`=`) — `03`과 `3`은 불일치, 공백 있는 값도 불일치 |

- `failures` 변수는 **한 번이라도 실패하면 끝까지 기억**했다가 마지막에 `exit "$failures"`
- 한계: stderr 텍스트에 대한 자동 검증은 없음

## 7. 통제된 결함(Controlled Defects) 실습

```bash
cp count_errors.sh count_errors.sh.correct   # 정상본 백업
```

| 결함 | 틀린 코드 | 실패 패턴 | 원인 |
|---|---|---|---|
| 1 | `[ "$status" -ge 1 ]` | clean, empty만 FAIL | grep의 정상 상태 1(매치없음)까지 오류로 취급 |
| 2 | stdout에 `echo "search completed"` 추가 | 성공 4건 FAIL (상태는 0) | 출력 계약 위반, 상태만 봐선 못 잡음 |
| 3 | `grep -c 'ERROR' $1` (따옴표 제거) | day one.log만 FAIL | 공백 경로가 여러 인자로 쪼개짐 |

**방법론**: 실패 **패턴**이 원인 코드 영역을 가리키는 **가설**을 제시 → 실패 케이스를 직접 재현(`bash count_errors.sh "data/day one.log" > out 2> err; echo $?`)해서 확인 → 복구 후 전체 검증기 재실행

### 실패 패턴 ↔ 코드 영역 매핑

| 관찰된 실패 | 후보 영역 |
|---|---|
| clean, empty만 실패 | 검색 상태 조건 (`-gt` vs `-ge`) |
| 4개 유효 파일 모두 실패 | 성공 시 stdout에 여분 출력 |
| 공백 포함 경로만 실패 | 인자 따옴표 처리 |

## 8. README 문서화

담을 내용:
- **실행 전제조건**: 실행 위치(oss-w04), bash 필요, 입력은 읽을 수 있는 로그 파일 경로
- **명령과 계약**: 각 스크립트 사용법, 검색 규칙(대문자 ERROR), 정상 시 stdout=COUNT·상태 0, 잘못된 입력 시 stderr·상태 1

→ README의 명령이 실제 소스코드와 일치해야 함

## 9. Git으로 관리하기

```bash
git init
printf 'reports/\n' > .gitignore     # 생성 출력물 제외

bash verify_counts.sh                 # 스테이징 전에 테스트
git add count_errors.sh summarize.sh verify_counts.sh README.md .gitignore
git diff --staged
git commit -m "Add reusable log summary tools"
git status --short
```
- **소스만 명시적으로 스테이징** (`git add .` 지양) — `data/`, `starter/`, `count_errors.sh.correct`는 제외
- 생성된 `reports/`는 커밋에서 제외, 소스 vs 빌드 산출물 분리는 C 프로젝트도 동일

## 10. 검증의 한계

`verify_counts.sh`는 stdout과 상태만 검사하고, **stderr 텍스트는 검증하지 않으며 모든 입력 형태를 다 커버하지 않음**. 검증 스크립트도 완벽하지 않다는 것을 아는 게 중요.

## 완료 체크리스트

- ✅ `count_errors.sh`가 유효/빈 파일/누락된 입력을 모두 처리
- ✅ `summarize.sh`가 요약 전에 모든 경로를 검증
- ✅ 따옴표로 감싼 경로가 하나의 인자로 유지됨
- ✅ Git이 스크립트는 기록하고 `reports/`는 무시함

## 실전에서 겪은 문제들 (트러블슈팅)

| 증상 | 원인 | 해결 |
|---|---|---|
| `echo $?`가 2로 나옴 | starter의 `exit 2`가 안 지워짐 / Windows 줄바꿈(CRLF)으로 문법 오류 | `grep -n exit count_errors.sh`로 확인, `bash -n`으로 문법 검사, `sed -i 's/\r$//' *.sh`로 CRLF 제거 |
| `grep -c '$2'` 결과가 이상함 | **작은따옴표라 변수가 확장 안 됨** (`$2`가 문자 그대로 검색됨) | 큰따옴표로: `grep -c "$2" "$1"` |
| `/mnt/c/...`(Windows 마운트)에서 작업 | 권한/줄바꿈 문제, OneDrive 동기화 이슈 | 리눅스 홈(`~`)으로 `cp -r` 이동 후 작업 |
| 폴더 복사 시 `oss-w04/oss-w04` 중첩 생성 | 대상 폴더가 이미 있을 때 `cp -r 원본 대상`은 대상 안에 복사됨 | `cp -r 원본/. 대상/`으로 내용만 복사 |
| `git diff --staged` 실행 후 멈춘 것처럼 보임 | `less` 페이저가 뜬 것 | `q`로 종료, 또는 `git --no-pager diff --staged` |

## Week 04-B 핵심 요약

> 셸 도구를 만들 때는 **입력 검증 → 출력/상태 즉시 캡처 → 정상 결과 출력**의 순서를 지켜야 하며, `$?`는 항상 직전 명령의 상태이므로 다른 명령 실행 전에 저장해야 한다. `grep -c`처럼 "매치 없음"이 정상적인 nonzero 상태를 반환하는 도구는, 진짜 실패와 정상적인 "0개 결과"를 구분하는 조건(`-gt 1` vs `-ge 1`)이 중요하다. 여러 파일을 다루는 도구는 처리 전에 **모든 입력을 먼저 검증**해야 하며, 리다이렉션은 검증과 무관하게 대상 파일을 먼저 비우므로 임시 파일 + 성공 시 이동 패턴이 안전하다. 자동 검증 스크립트는 stdout(텍스트 비교)과 종료상태(숫자 비교)를 모두 확인하며, 실패를 끝까지 기억(`failures` 변수)해야 한다.

---

## 참고 자료 (Week 04)
- MIT Missing Semester: Shell Tools and Scripting
- Software Carpentry: The Unix Shell / Shell Scripts
- William Shotts: Writing Shell Scripts
- GNU Bash Reference Manual: gnu.org/software/bash/manual/
- GNU Grep Manual: gnu.org/software/grep/manual/

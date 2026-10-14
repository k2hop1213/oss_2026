# Week 07-A. Testing and Review

> 프로그램이 맞게 동작하는지 **테스트로 확인**하고, 버그를 고친 뒤 **회귀 테스트**로 다시 안 생기게 막고, push할 때마다 **GitHub Actions(CI)**가 자동으로 `make check`를 돌리게 한다.

## 1. 프로젝트 (`oss-w07`)

```
src/main.c        인자 검사 후 처리 함수 호출
src/filters.c     brighten / grayscale / invert
src/image_io.c    이미지 읽기/저장 (vendor/stb 사용)
scripts/batch.sh  여러 사진을 한 번에 gray 변환
tests/test_filters.c, tests/test_batch.sh
Makefile, .github/workflows/check.yml
```

### `argc` / `argv`

```
./bin/image-lab "assets/day one.jpg" output/one.png gray
argv[0]          argv[1]              argv[2]        argv[3]     argc = 4
```
- `argc`는 실행파일 이름까지 셈 (셸의 `$0`, `$1`, `$#`과 같은 개념)
- `main`은 인자 개수 → 모드 → gain → 이미지 크기를 검사하고, 문제가 있으면 stderr + `return 2`
- `gain`은 `bright`에서만, 기본값 `DEFAULT_GAIN` = 60, 범위 -255~255 (`strtol`로 변환, `"40x"`, `"hello"`, `"300"`은 거부)

## 2. 테스트의 기본

테스트 = 정한 입력으로 실행 → **예상 결과와 실제 결과 비교**. 예상 결과는 코드가 아니라 **요구사항**에서 나온다.

| 입력 | 예상 |
|---|---|
| `grayscale` (90, 30, 0) | (40, 40, 40) |
| `grayscale` (255, 255, 255) | (255, 255, 255) |
| `brighten_channel(240, 40)` | 255 (clamp) |

## 3. 단위 테스트 (C 함수 직접 호출)

```c
#include <assert.h>
#include "filters.h"
int main(void) {
    unsigned char rgb[] = {90, 30, 0, 255, 255, 255};
    grayscale(rgb, sizeof(rgb));               // 6바이트 = 픽셀 2개
    assert(rgb[0] == 40 && rgb[1] == 40 && rgb[2] == 40);
    assert(brighten_channel(100, 40) == 140);
    puts("PASS filter contracts");
    return 0;
}
```
- `assert(조건)`: 거짓이면 실패를 보고하고 **즉시 중단** (puts까지 못 감)
- 테스트는 **자기 main을 가진 별도 실행파일**, `src/main.c`와 같이 링크하지 않음
- `-DNDEBUG`를 쓰면 assert가 꺼지므로 쓰지 않음

```bash
gcc -Iinclude -std=c11 -Wall -Wextra -g -O0 tests/test_filters.c src/filters.c -o bin/test_filters
./bin/test_filters      # PASS filter contracts
```

## 4. 명령줄(CLI) 테스트

| 경우 | 예상 exit | 확인 |
|---|---|---|
| 정상 사진, gray | 0 | PNG 생성 |
| 모드 `sepia` | 2 | stderr `unknown filter`, PNG 없음 |
| 깨진 `broken.jpg` | 2 | `cannot load input image` |
| 출력 폴더 없음 | 2 | `cannot write output image` |

```bash
work=$(mktemp -d) || exit 2                 # 임시 폴더
trap 'rm -rf -- "$work"' EXIT               # 끝나면 자동 정리

./bin/image-lab assets/sample.jpg "$work/result.png" sepia \
    >"$work/stdout" 2>"$work/stderr"
status=$?                                   # 바로 저장 (4주차)

[[ $status -eq 2 ]] || exit 1
[[ ! -e "$work/result.png" ]] || exit 1
grep -Fxq "unknown filter" "$work/stderr" || exit 1
printf "PASS unsupported mode\n"
```

### 프로그램의 exit vs 테스트의 exit

- 잘못된 입력에 **프로그램이 exit 2를 내는 게 정답**인 경우가 있음 → 그걸 확인한 **테스트는 exit 0 (PASS)**
- 테스트가 실패하면 반드시 **`exit 1`** → Make와 CI가 실패를 알아챔. `echo FAIL`만 하고 `exit 0`이면 실패가 묻힘

## 5. 회귀 테스트와 `batch.sh` 버그

### 증상

입력 순서 `day one.jpg`(정상) / `broken.jpg`(깨짐) / `day two.jpg`(정상)

```bash
./bin/image-lab "$input" "$out/$name.png" gray
printf 'DONE %s\n' "$input"     # 실패해도 DONE
...
exit 0                          # 실패가 있어도 0
```
→ broken.jpg에 **거짓 DONE**, 최종 상태 0. 기존 테스트는 정상 입력만 넣어서 못 잡음

### 요구사항

- 실패 후에도 **다음 입력 계속 처리**
- 성공 → `DONE`(stdout), 실패 → `FAILED`(stderr)
- 하나라도 실패하면 **최종 1**, 전부 성공이면 0

### 틀린 수정 두 가지

| 수정 | 문제 |
|---|---|
| 실패하면 바로 `exit 1` | 뒤의 `day two.jpg` 처리 안 함 |
| 성공할 때마다 `failed=0` | 마지막이 성공이면 앞의 실패가 지워짐 → 0 |

### 맞는 수정

```bash
failed=0
for input in "$@"; do
    name=$(basename -- "$input")
    if ./bin/image-lab "$input" "$out/$name.png" gray; then
        printf 'DONE %s\n' "$input"
    else
        printf 'FAILED %s\n' "$input" >&2
        failed=1                 # 다시 0으로 되돌리지 않음
    fi
done
exit "$failed"
```
4주차 `verify_counts.sh`의 `failures` 변수와 같은 원리

### 회귀 테스트(regression test)

- 버그를 드러낸 `정상/깨짐/정상` 케이스를 **고친 뒤에도 계속 남겨둠** → 나중에 다시 망가지면 바로 걸림
- 검사: 상태 1 / `day two.jpg.png` 존재(`-s`) / stderr에 `FAILED assets/broken.jpg` / stdout에 `DONE assets/broken.jpg` 없음
- **테스트의 기대값을 바꿔서(`-eq 0`) 통과시키면 안 됨**. 코드를 고친다

## 6. Makefile

```make
.PHONY: all check unit-test batch-test clean
bin/test_filters: tests/test_filters.c build/filters.o include/filters.h
	mkdir -p bin
	$(CC) $(CPPFLAGS) $(CFLAGS) tests/test_filters.c build/filters.o -o $@
unit-test: bin/test_filters
	./bin/test_filters
batch-test: bin/image-lab
	bash tests/test_batch.sh
check: unit-test batch-test
```
- `make check` = 빌드 + 두 테스트. 테스트가 `exit 1` → `make: *** [batch-test] Error 1`

## 7. GitHub Actions (CI)

push할 때마다 GitHub 서버(runner)가 자동으로 `make check` 실행

```yaml
# .github/workflows/check.yml
name: Build and test
on: [push, pull_request]
permissions:
  contents: read
jobs:
  check:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v7
      - run: make check
```
- YAML은 **스페이스 들여쓰기**로 구조 표현 (Makefile은 Tab)
- 저장소 **Actions 탭**에서 커밋별 결과 확인
- 예: 버그 커밋 `b8990c5` ❌ (`FAIL mixed status: 0 (expected 1)`) → `batch.sh`만 고친 `78851c8` ✅. 테스트와 워크플로는 그대로

## Week 07-A 핵심 요약

> 테스트는 요구사항에서 나온 예상 결과와 실제 결과를 비교한다. C 함수는 `assert`로 단위 테스트하고, 명령은 stdout·stderr·종료 상태·생성 파일을 각각 캡처해 검사한다. 잘못된 입력에 프로그램이 실패하는 것이 정답일 수 있으며, 테스트 자체는 실패 시 반드시 nonzero로 끝나야 Make와 CI가 알 수 있다. 버그를 드러낸 입력은 회귀 테스트로 남기고, 테스트 기대값이 아니라 코드를 고친다. GitHub Actions는 push마다 같은 `make check`를 자동으로 실행해 커밋별 결과를 기록한다.

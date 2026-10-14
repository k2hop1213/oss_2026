# 7주차 문법: 테스트 / CI

명령어 설명 없이 문법 형태만 빠르게 찾아보는 용도.

## C 단위 테스트

```c
#include <assert.h>
assert(조건);              // 거짓이면 즉시 중단
puts("PASS ...");
```
```bash
gcc -Iinclude -g -O0 tests/test_x.c src/x.c -o bin/test_x
```

## 테스트 스크립트 뼈대

```bash
work=$(mktemp -d) || exit 2
trap 'rm -rf -- "$work"' EXIT

명령 >"$work/stdout" 2>"$work/stderr"
status=$?

[[ $status -eq 2 ]] || exit 1          # 숫자 비교
[[ -e "$work/out.png" ]] || exit 1     # 존재
[[ ! -e "$work/out.png" ]] || exit 1   # 없어야 함
[[ -s "$work/out.png" ]] || exit 1     # 존재 + 비어있지 않음
grep -Fxq "메시지" "$work/stderr" || exit 1
echo "PASS ..."
```

## 새 문법

```
mktemp -d              임시 폴더 생성 후 경로 출력
trap '명령' EXIT        스크립트 끝날 때 실행
[[ ... ]]              bash의 확장 test
basename -- 경로        파일 이름만
grep -F                글자 그대로 (정규식 X)
grep -x                줄 전체가 일치
grep -q                출력 없이 상태만
```

## 실패를 끝까지 기억

```bash
failed=0
for input in "$@"; do
    if 명령 "$input"; then echo "DONE $input"
    else echo "FAILED $input" >&2; failed=1; fi
done
exit "$failed"
```

## C main 인자

```c
int main(int argc, char **argv)   // argc: 실행파일 포함 개수, argv[0]: 실행파일
long v = strtol(str, &end, 10);   // end: 변환이 멈춘 위치
```

## Makefile 테스트 타깃

```make
check: unit-test batch-test
unit-test: bin/test_filters
	./bin/test_filters
```

## GitHub Actions

```yaml
name: Build and test
on: [push, pull_request]
jobs:
  check:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v7
      - run: make check
```
위치: `.github/workflows/check.yml` (스페이스 들여쓰기)

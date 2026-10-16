# W07A 슬라이드 따라 읽기 (Testing and Review)

> 슬라이드 번호 순서대로, **이 장이 무슨 말을 하는지** 한국어로 풀어둔 가이드.
> 강의 흐름은 크게 4덩어리: ① 프로그램 소개(4~11) → ② 테스트 기본(12~23) → ③ 버그 찾고 고치기(24~36) → ④ GitHub Actions(37~42). 43~59는 부록(전체 코드).

---

## ① 이번 주 프로그램 소개 (슬라이드 4~11)

> 테스트를 하려면 **뭘 테스트하는지** 알아야 하니까, 먼저 프로그램을 보여주는 파트.

### 4. Image Processing: Input and Result
- `./bin/image-lab assets/sample.jpg output/gray.png gray` → 사진을 흑백 PNG로 만듦
- "출력은 **크기는 그대로, RGB 값만** 바뀜"

### 5. Command Arguments Enter main
```
./bin/image-lab "assets/day one.jpg" output/one.png gray
argv[0]          argv[1]              argv[2]        argv[3]     argc = 4
```
- C 프로그램이 명령줄 인자를 받는 방법
- `argc` = 인자 개수 (**실행 파일 이름도 셈**) → 4주차 셸의 `$#`, `$0 $1 $2`와 같은 개념
- argv[1] 입력, argv[2] 출력, argv[3] 모드

### 6. Project Files and Source Locations
| 파일 | 역할 |
|---|---|
| `src/main.c` | 인자 검사하고 처리 함수 부르기 |
| `src/filters.c` + `include/filters.h` | RGB 바꾸는 함수 (밝게/흑백/반전) |
| `src/image_io.c` + `include/image_io.h` | 이미지 읽기·저장·해제 (vendor/stb 사용) |
| `scripts/batch.sh` | 여러 사진 한 번에 처리 |
| `tests/` | 테스트 |
| `Makefile` | 빌드 + 테스트 실행 |
| `.github/workflows/check.yml` | GitHub 자동 실행 설정 |

### 7~9. main.c
- 7: 전체 코드. **파란 박스 = image_io.c 함수 호출, 초록 박스 = filters.c 함수 호출**
- 8: `load_rgb`로 사진 읽기 → 실패하면 **NULL** → `cannot load input image` + `return 2` / 크기 이상하면 `image too large`
- 9: 모드에 따라 `grayscale` / `brighten` / `invert` → `save_rgb`로 저장 → `free_rgb`로 메모리 해제 → 저장 실패면 `return 2`, 성공이면 `Created ...` 출력 후 `return 0`
- 요점: **문제 있으면 stderr + return 2, 정상이면 return 0**

### 10. filters.c: Brightness
```c
int result = value + delta;
if (result > 255) result = 255;
if (result < 0) result = 0;
return (unsigned char)result;
```
- 6주차에서 고친 **clamp 버전**이 이미 들어가 있음
- `brighten`은 모든 칸에 `brighten_channel`을 호출

### 11. filters.c: Grayscale and Inversion
- `grayscale`: 3칸(R,G,B)씩 묶어서 평균 → 세 칸 다 그 값 (`i += 3`)
- `invert`: 각 칸을 `255 - 값`

---

## ② 테스트 기본 (슬라이드 12~23)

### 12. Software Testing: Expected and Actual Results
> "테스트는 정한 입력으로 코드를 실행하고, **실제 결과를 기대 결과와 비교**한다."

| 입력 | 기대 |
|---|---|
| grayscale (90, 30, 0) | (40, 40, 40) ← (90+30+0)/3 = 40 |
| grayscale (255, 255, 255) | (255, 255, 255) |
- "기대 결과는 **요구되는 동작**에서 나온다" (코드에서 베끼는 게 아님)

### 13. A Unit Test Calls the Function Directly
```c
unsigned char rgb[] = {90, 30, 0, 255, 255, 255};
grayscale(rgb, sizeof(rgb));
assert(rgb[0] == 40 && rgb[1] == 40 && rgb[2] == 40);
```
- **단위 테스트** = 함수를 직접 불러서 확인
- "테스트의 main은 src/main.c와 **다른 별도 실행 파일**"
- `sizeof(rgb)` = 6바이트 = 픽셀 2개 × 3칸

### 14. assert: Condition and Execution
- `assert(조건)`: **참이면 다음 줄 계속, 거짓이면 실패 보고하고 즉시 중단** (뒤의 `puts("PASS...")`까지 못 감)
- "`-DNDEBUG` 안 씀" → 이거 쓰면 assert가 꺼지니까

### 15. Compile the Test with Its Implementation
```bash
gcc -Iinclude -std=c11 -Wall -Wextra -g -O0 tests/test_filters.c src/filters.c -o bin/test_filters
./bin/test_filters     # PASS filter contracts
```
- 테스트 파일 + **테스트할 대상(filters.c)** 을 같이 링크 (5주차: 선언만으론 안 되고 정의가 필요)

### 16. Test Entry Points in the Project
- 왼쪽: 함수 테스트 → **배열 값**을 확인
- 오른쪽: CLI/batch 테스트 → **파일과 종료 상태**까지 확인
- batch.sh는 입력마다 image-lab을 한 번씩 실행

### 17. Capture a Command-Line Result
```bash
work=$(mktemp -d)
./bin/image-lab assets/sample.jpg "$work/out.png" sepia >"$work/stdout" 2>"$work/stderr"
status=$?
```
- sepia(없는 모드) 넣으면 기대값: stdout 비어있음 / stderr `unknown filter` / 상태 2 / out.png 없음
- "에러 메시지와 종료 상태는 **같은 명령의 서로 다른 결과**" (4주차: stdout, stderr, 상태 3가지)

### 18. Normal and Invalid CLI Inputs
| 경우 | 상태 | 관찰 |
|---|---|---|
| 정상 사진, gray | 0 | PNG 생김 |
| sepia | 2 | `unknown filter` |
| 텍스트를 broken.jpg로 저장 | 2 | `cannot load input image` |
| 출력 폴더 없음 | 2 | `cannot write output image` |

### 19. Reproduce an Unsupported Mode
- main.c의 모드 검사 코드가 바로 그 `unknown filter` + `return 2`를 만든다는 걸 보여줌

### 20. Isolate the Test Output
```bash
work=$(mktemp -d) || exit 2           # 새 임시 폴더
trap 'rm -rf -- "$work"' EXIT         # 테스트 끝나면 그 폴더 자동 삭제
```
- 테스트 결과물을 **임시 폴더에 격리** → 다른 파일 안 건드리고, 매번 깨끗하게 시작

### 21. Capture Output and Exit Status
- "`$?`는 테스트할 명령 **바로 다음에** 저장. 다음 명령이 덮어씀" (4주차 그대로)

### 22. Compare the Captured Values
```bash
[[ $status -eq 2 ]] || exit 1                          # 상태가 2가 아니면 테스트 실패
[[ ! -e "$work/result.png" ]] || exit 1                # PNG가 있으면 실패
grep -Fxq "unknown filter" "$work/stderr" || exit 1    # 메시지 없으면 실패
printf "PASS unsupported mode\n"
```
- `[[ ]]` = `[ ]`의 확장판 (조건 검사)
- `-eq` 정수 비교 / `-e` 파일 존재 / `!` 반대
- `grep -Fxq`: `-F` 글자 그대로, `-x` 줄 전체가 정확히 일치, `-q` 출력 없이 상태만
- `A || exit 1` = A가 실패하면 테스트를 실패로 끝냄

### 23. Program Exit and Test Exit ⭐
- 프로그램(image-lab)은 **exit 2** (sepia 거부 = 맞는 동작)
- 테스트는 그걸 확인했으니 **exit 0 (PASS)**
- "**프로그램 exit 2가 기대 동작이고, 테스트 exit 0이 그걸 확인한다**"

### 24. Inputs Covered by the Supplied Tests
- 지금 있는 테스트들은 정상 입력만 씀 (batch 테스트에 깨진 사진 없음)
- "통과했다 = **그 입력은** 괜찮다. 깨진 입력은 **테스트 안 된 상태**로 남아있다"

---

## ③ 버그 찾고 고치기 (슬라이드 25~36)

### 25. Batch Processing with a Damaged Input
- 입력 순서: `day one.jpg`(정상) → `broken.jpg`(깨짐) → `day two.jpg`(정상)
- 원하는 결과: 정상 2개는 PNG가 나오고, **실패는 실패라고 알려줘야** 함

### 26. Observed Output Can Contradict the Result
- 실제로 돌려보니: broken.jpg에도 **DONE**(거짓말), 최종 상태 **0**

### 27. Regression Test: Reproducing a Known Failure
- 기존 케이스(정상/정상, 기대 0)는 **그대로 유지**
- 새 케이스(정상/깨짐/정상, 기대 **1**, 뒤 PNG 존재) 추가
- "**고친 뒤에도 이 케이스를 남겨둬서** 나중에 같은 버그가 생기면 잡는다" = 회귀 테스트

### 28. Makefile: Building and Running Tests
```make
unit-test: bin/test_filters      → ./bin/test_filters
batch-test: bin/image-lab        → bash tests/test_batch.sh
check: unit-test batch-test      → 둘 다
```
- `make check` 한 번이면 빌드 + 테스트 2개

### 29. make check: Locating the Failed Test
```
PASS valid batch
FAIL mixed status: 0 (expected 1)
make: *** [Makefile:25: batch-test] Error 1
```
- 새 테스트가 버그를 잡음 → 테스트가 exit 1 → make가 Error로 알려줌
- 오른쪽 원인: batch.sh가 image-lab 결과와 상관없이 무조건 `DONE` 찍고 마지막에 `exit 0`

### 30. The Batch Script Reports False Success
- day one 성공 / broken **실패** / day two 성공인데 → 셋 다 DONE, 최종 0
- "성공 메시지와 최종 상태를 고치되, **뒤의 변환은 계속** 해라"

### 31. Checks for the Mixed-Input Test
```bash
[[ $status -eq 1 ]] || exit 1                                         # 상태 1
[[ -s "$work/mixed/day two.jpg.png" ]] || exit 1                      # 뒤 PNG 있고 비어있지 않음
grep -Fxq "FAILED assets/broken.jpg" "$work/stderr" || exit 1         # FAILED 있음
if grep -Fxq "DONE assets/broken.jpg" "$work/stdout"; then exit 1; fi # DONE 있으면 안 됨
```
- `-s` = 파일이 있고 **크기가 0보다 큼**

### 32. Requirements for the Batch Correction
1. 실패 후에도 **모든 입력 시도**
2. 성공 → DONE, 실패 → FAILED(stderr)
3. 하나라도 실패 → **1**, 전부 성공 → 0
4. 기존 인자 검사, 폴더 검사, 따옴표는 유지

### 33. Immediate Exit Skips the Later Input ❌
```bash
else printf 'FAILED ...' >&2; exit 1     # 실패하자마자 끝
```
- 상태 1은 맞지만 **day two.jpg를 처리 안 함** → "뒤 PNG 존재" 검사 실패

### 34. Later Success Erases the Failure Status ❌
```bash
then ...; failed=0     # 성공할 때마다 0으로
```
- broken에서 1 → day two 성공에서 다시 **0** → 최종 0 → 상태 검사 실패

### 35. Preserve Failure Until the Batch Ends ✅
```bash
failed=0
for input in "$@"; do
    name=$(basename -- "$input")
    if ./bin/image-lab "$input" "$out/$name.png" gray; then
        printf 'DONE %s\n' "$input"
    else
        printf 'FAILED %s\n' "$input" >&2
        failed=1
    fi
done
exit "$failed"
```
- `failed`는 **처음에 한 번만 0**, 실패하면 1, **다시 0으로 안 바꿈** → 끝까지 기억
- `if 명령; then` = 명령의 종료 상태로 분기 (image-lab이 0이면 then, 2면 else)

### 36. Keep the Expected Batch Status Unchanged ❌
- 테스트를 `[[ $status -eq 0 ]]`으로 바꿔서 통과시키기 → **버그를 인정해버리는 것**
- 요구사항은 "실패 있으면 1" → **테스트 기대값은 그대로, 코드를 고쳐라**

### 37. Report a Failed Comparison to Make ❌
```bash
if [[ $status -ne 1 ]]; then echo "FAIL mixed status" >&2; fi
exit 0          # FAIL 찍고도 0 → make는 성공인 줄 앎
```
- 고침: FAIL 블록 안에 **`exit 1`**
- "Make는 테스트가 **0이 아닌 상태**로 끝나야 실패를 안다"

---

## ④ GitHub Actions (슬라이드 38~42)

### 38. GitHub Actions
- 저장소에서 빌드·테스트 같은 명령을 **자동 실행**하는 기능
- push하면 설정한 workflow가 GitHub **runner**(GitHub의 서버 컴퓨터)에서 돎
- **Actions 탭**에서 실행마다 커밋과 결과(✅/❌) 확인

### 39. Workflow File in the Repository
1. 빈 GitHub 저장소 생성 → 2. `.github/workflows/check.yml` 저장 → 3. commit, push → 4. GitHub가 workflow 읽음 → 5. runner가 `make check` 실행
- workflow = **YAML 텍스트 파일**로 정의한 자동화 과정

### 40. Workflow Configuration and Execution
```yaml
name: Build and test           # 이름
on: [push, pull_request]       # 언제: push나 PR 때
permissions:
  contents: read               # 저장소 읽기 권한만
jobs:
  check:                       # 작업 이름
    runs-on: ubuntu-24.04      # 어떤 컴퓨터에서
    steps:                     # 순서대로
      - uses: actions/checkout@v7   # 1) 커밋된 코드 가져오기
      - run: make check             # 2) make check 실행
```
- "**들여쓰기가 구조를 정한다**" (스페이스, Tab 아님)
- 흐름: push → 그 커밋 checkout → Ubuntu runner에서 make check

### 41. GitHub Actions: Failed Regression Test
- 커밋 `b8990c5` (테스트만 추가, batch.sh는 그대로) → ❌ `FAIL mixed status: 0 (expected 1)`

### 42. GitHub Actions: Results for Two Commits
- `b8990c5` ❌ → `78851c8` ✅
- "**테스트와 workflow는 그대로**, batch.sh만 고쳤다"

---

## 부록 (슬라이드 43~59) — 전체 코드
- 44~49: main.c를 줄 번호별로 다시 (인자 검사 → 모드 → gain → 불러오기 → 처리 → 저장)
- 47: gain 변환 예
  | 입력 | 결과 |
  |---|---|
  | `"40"` | 허용 |
  | `"40x"` | 거부 (뒤에 글자 남음) |
  | `"hello"` | 거부 (숫자 없음) |
  | `"300"` | 거부 (범위 밖) |
- 50: filters.h — 헤더 가드, `DEFAULT_GAIN 60`, 함수 선언 4개
- 53~55: image_io — stb 라이브러리로 JPEG/PNG 읽고 PNG 쓰기, `free_rgb`로 해제
- 56~59: Makefile — 변수, `all: bin/image-lab`, 오브젝트별 헤더 의존성, 테스트 규칙 (6주차 그대로)

---

## 한 장 요약
1. 테스트 = 입력 → 실제 vs **기대(요구사항)** 비교
2. 단위 테스트(assert, 함수 직접) / CLI 테스트(실행해서 stdout·stderr·상태·파일 확인)
3. 프로그램 exit 2가 **정답**일 수 있음 / 테스트 실패는 반드시 **exit 1**
4. 버그 재현 테스트 먼저 → 코드 수정 → 같은 테스트 통과 → **계속 남김**(회귀)
5. 틀린 수정 3개: 즉시 exit / 성공 때 failed=0 / 테스트 기대값 바꾸기 (+ FAIL 찍고 exit 0)
6. GitHub Actions: push마다 runner가 `make check` → 커밋별 ✅❌

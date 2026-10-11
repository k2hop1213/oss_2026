# Week 06-A. Make & Debugging

## 1. 이번 주 프로젝트: image-lab

`oss-w06`은 사진을 읽어 필터를 적용하고 PNG로 저장하는 C 프로그램입니다.

```
assets/sample.jpg → load_rgb() → filter(rgb, count) → save_rgb() → output/*.png
```

- 필터 3개: `grayscale`(흑백), `brighten`(밝게), `invert`(반전). 명령줄 인자로 선택
- 학생이 직접 다루는 부분: 필터의 조건과 반복문, 함수 인자와 결과, 헤더, 일부 Make 규칙, 빌드 로그와 검증
- 제공되는 부분: 이미지 읽기/저장, CLI 검사, stb 라이브러리 내부

### 폴더 구조

| 위치 | 내용 |
|---|---|
| `src/` | `main.c`, `filters.c`, `image_io.c` |
| `include/` | `filters.h`, `image_io.h` |
| `vendor/stb/` | 이미지 입출력 라이브러리 (+ 라이선스) |
| `assets/` | `sample.jpg`, `ATTRIBUTION.txt` |
| `tests/` | `test_filters.c`, `check_cli.sh` |
| `Makefile`, `.gitignore` | 빌드 규칙, Git 무시 목록 |
| **생성물** | `build/*.o`, `lib/libfilters.a`, `bin/image-lab`, `output/*.png` |

## 2. 이미지 = RGB 숫자 배열

픽셀 하나가 숫자 3개(R, G, B)이고, 각 값은 0~255입니다. 그래서 필터는 **배열을 3칸씩 건너뛰며 값을 바꾸는 반복문**입니다.

```c
for (size_t i = 0; i < count; i += 3) {
    int gray = (rgb[i] + rgb[i+1] + rgb[i+2]) / 3;
    rgb[i] = rgb[i+1] = rgb[i+2] = (unsigned char)gray;
}
```

영역(가로 w, 세로 h) 안의 픽셀 위치:

```c
size_t i = ((size_t)y * image_width + x) * 3;
```

## 3. 수동 빌드 (5주차 복습)

```bash
gcc -Iinclude -E src/filters.c -o build/filters.i
gcc -Iinclude -S src/filters.c -o build/filters.s
gcc -Iinclude -c src/filters.c -o build/filters.o
ar rcs lib/libfilters.a build/filters.o
gcc -o bin/image-lab build/main.o build/image_io.o lib/libfilters.a -lm
```

파일이 많아지면 매번 이걸 치기 힘들고, 무엇을 다시 컴파일해야 하는지 헷갈립니다. 이걸 자동화하는 것이 **Make**입니다.

## 4. Makefile 규칙

```make
build/filters.o: src/filters.c include/filters.h
	mkdir -p build
	gcc -Iinclude -c src/filters.c -o build/filters.o
```

| 부분 | 이름 | 뜻 |
|---|---|---|
| `build/filters.o` | 타깃 (target) | 만들 결과물 |
| `src/filters.c include/filters.h` | 전제조건 (prerequisites) | 재료. 바뀌면 다시 만듦 |
| 들여쓴 줄 | 레시피 (recipe) | 만드는 명령 |

- **레시피 줄은 반드시 Tab으로 시작.** 스페이스면 `*** missing separator.  Stop.`
- 실행: `make` (첫 타깃), `make images` (지정 타깃)

## 5. 변수

```make
CC       = gcc
AR       = ar
CPPFLAGS = -Iinclude -Ivendor/stb
CFLAGS   = -std=c11 -Wall -Wextra -g -O0
LDLIBS   = -lm
```

- 읽을 때: `$(CC) $(CPPFLAGS) $(CFLAGS)`
- Make의 `$(NAME)` = 변수 읽기 / Bash의 `$(cmd)` = 명령 결과 치환 → **다른 것**

## 6. 자동 변수

| 기호 | 뜻 | `app: main.o util.o`에서 |
|---|---|---|
| `$@` | 타깃 | `app` |
| `$<` | 첫 번째 전제조건 | `main.o` |
| `$^` | 전제조건 전부 (중복 제거) | `main.o util.o` |

```make
build/filters.o: src/filters.c include/filters.h
	$(CC) $(CPPFLAGS) $(CFLAGS) -c $< -o $@

lib/libfilters.a: build/filters.o
	$(AR) rcs $@ $^

bin/image-lab: build/main.o build/image_io.o lib/libfilters.a
	$(CC) $(CFLAGS) -o $@ $^ $(LDLIBS)
```

- 컴파일 → `$<` (소스 하나. `$^`면 헤더까지 gcc에 넘어감)
- 링크, 아카이브 → `$^` (재료 전부)
- `$(LDLIBS)` (`-lm`)는 **맨 뒤** (링크 순서, 5주차)

## 7. Make가 다시 만들지 판단하는 법

**파일 수정 시각 비교.** 전제조건이 타깃보다 최신이거나 타깃이 없으면 레시피를 실행합니다. 전제조건을 먼저 만들고 타깃을 만듭니다.

```
make: Nothing to be done for 'all'.     # 전부 최신
make: 'bin/image-lab' is up to date.    # 지정한 타깃이 최신
```

### 헤더 의존성

| 오브젝트 | 전제조건 |
|---|---|
| `build/main.o` | `src/main.c`, `include/filters.h`, `include/image_io.h` |
| `build/filters.o` | `src/filters.c`, `include/filters.h` |
| `build/image_io.o` | `src/image_io.c`, `include/image_io.h`, stb 헤더 |

### 무엇을 고치면 무엇이 다시 만들어지나

| 고친 파일 | 다시 만들어지는 것 |
|---|---|
| `src/filters.c` | `filters.o` → `libfilters.a` → `image-lab` (`main.o`, `image_io.o`는 재사용) |
| `include/filters.h` | `main.o`, `filters.o` → `libfilters.a` → `image-lab` (`image_io.o`는 재사용) |
| `include/image_io.h` | `main.o`, `image_io.o` → `image-lab` |
| 없음 | 없음 |

방법: 고친 파일을 재료로 가진 타깃 → 그 타깃을 재료로 가진 타깃 → … 끝까지 따라간다.

### ⚠️ 헤더를 전제조건에서 빼먹으면

헤더를 고쳐도 그 `.o`는 다시 만들어지지 않음 → **예전 `.o`(stale)가 그대로 링크** → 결과가 이상해짐. 5주차의 `ERROR` → `WARN` 문제와 같은 원리.

## 8. 이미지 타깃과 가짜 타깃

```make
output/gray.png: bin/image-lab assets/sample.jpg
	./bin/image-lab assets/sample.jpg $@ gray

images: output/gray.png output/bright.png output/invert.png

.PHONY: all images check clean

check: bin/test_filters bin/image-lab
	./bin/test_filters
	bash tests/check_cli.sh

clean:
	rm -f build/*.o lib/libfilters.a bin/image-lab bin/test_filters output/*.png
```

| 질문 | 답 |
|---|---|
| `.PHONY` 왜 씀? | 파일이 아니라 **동작 이름**. 같은 이름의 파일이 있어도 항상 실행 |
| `touch assets/sample.jpg` 후 `make images` | PNG 레시피만 다시 실행 |
| `make -n` | **실행하지 않고** 실행할 명령만 출력 |
| `CFLAGS`만 바꿨다면? | Make는 시각만 보고 옵션 변경은 모름 → `make clean; make images; make check` |

## 9. 버그: 밝게 했더니 하늘이 어두워짐

```c
return (unsigned char)(value + delta);   // 버그
```

`unsigned char`는 0~255. 넘으면 한 바퀴 돈다(wrap).

| 계산 | 결과 | 계산법 |
|---|---|---|
| 10 + 40 | 50 | 범위 안 |
| 250 + 40 = 290 | **34** | 290 − 256 |
| 240 + 30 = 270 | **14** | 270 − 256 |
| 200 + 100 = 300 | **44** | 300 − 256 |
| 20 − 50 = −30 | **226** | −30 + 256 |

검산: 결과 + 256 (또는 − 256) = 원래 값

**수정 (clamp): `int`로 계산 → 자르기 → 마지막에 변환**

```c
int result = value + delta;
if (result > 255) result = 255;
if (result < 0)   result = 0;
return (unsigned char)result;
```

## 10. 테스트 드라이버

```c
check("upper clamp", brighten_channel(250, 40), 255);
// check(이름, 실제값, 기대값) → PASS / FAIL 출력
```

- 실패 개수를 종료 상태로 돌려줌 → `make check`에서 `Error N`으로 보임
- 실패 **패턴**으로 원인 추측 (4주차 verify_counts와 같은 방법)

## 11. GDB

**준비:** `-g` (디버그 정보), `-O0` (최적화 끔). `CFLAGS`에 포함.

```bash
cp variants/filters_wrap.c src/filters.c
make bin/test_filters
gdb --args ./bin/test_filters
```

| 명령 | 하는 일 |
|---|---|
| `break 함수` | 그 함수에서 멈추도록 설정 |
| `break 함수 if value == 250 && delta == 40` | 조건 맞을 때만 멈춤 |
| `run` | 처음부터 실행 |
| `print 식` | 값 보기 (`print /u value`: unsigned 숫자로) |
| `next` | 한 줄 실행, 함수 호출은 **건너뜀** |
| `step` | 한 줄 실행, 함수 **안으로 들어감** |
| `finish` | 현재 함수 끝까지 실행 + **리턴값** 출력 |
| `bt` | 호출 스택 (어떤 함수를 거쳐 왔나) |
| `continue` | 다음 중단점까지 |

수업 흐름:

```
(gdb) break brighten_channel if value == 250 && delta == 40
(gdb) run
(gdb) print /u value        → $1 = 250
(gdb) print delta           → $2 = 40
(gdb) print value + delta   → $3 = 290
(gdb) finish                → Value returned is $4 = 34
(gdb) bt                    → #0 brighten_channel  #1 main
```

290이 들어갔는데 34가 리턴됨 → 오버플로 확인 → 코드 수정 → `make check` → `make images`

## 12. 에러 메시지 → 의심할 곳

| 에러 / 증상 | 의심할 곳 |
|---|---|
| `missing separator` | 레시피 들여쓰기 (Tab) |
| `filters.h: No such file or directory` | `-I` 경로, 헤더 위치 |
| `undefined reference` | 링크 입력(`.o`, `.a`) 누락, 함수 정의 없음 |
| `cannot write output` | 출력 폴더, 쓰기 권한 |
| 밝은 부분이 어두워짐 | 픽셀 값 범위, 변환 로직 (오버플로) |

## 13. Git으로 관리할 것

| 올림 | 안 올림 (`.gitignore`) |
|---|---|
| `src/`, `include/`, `Makefile`, `tests/`, `README.md`, `.gitignore`, `vendor/`(+라이선스), `assets/`(+출처) | `build/*.o`, `lib/libfilters.a`, `bin/image-lab`, `bin/test_filters`, `output/*.png` |

기준: **소스에서 다시 만들 수 있는 것은 올리지 않는다.**

## 14. 실습 (W06B region): 영역만 밝게

```c
size_t pixels = 0;
for (int y = r.y; y < r.y + r.height; ++y) {
    for (int x = r.x; x < r.x + r.width; ++x) {
        size_t index = ((size_t)y * width + x) * 3;
        for (int c = 0; c < 3; ++c)
            rgb[index + c] = brighten_channel(rgb[index + c], gain);
        ++pixels;          // ✅ 안쪽 반복문: 픽셀 수
    }
    // ++pixels;           // ❌ 여기면 줄 수를 셈
}
return pixels;
```

- `++pixels`가 바깥에 있으면 가로 4 × 세로 2 영역에서 8이 아니라 2를 리턴
- 테스트 패턴: 가로 1칸짜리(세로 영역), 1×1 픽셀은 통과하고 가로가 넓은 영역만 실패 → 개수 세는 위치를 의심

## Week 06-A 핵심 요약

> Make는 Makefile에 적힌 **타깃: 전제조건 + (Tab) 레시피** 규칙을 보고, 전제조건이 타깃보다 **최신일 때만** 레시피를 다시 실행해 5주차의 수동 컴파일/링크를 자동화한다. `$@`(타깃), `$<`(첫 전제조건), `$^`(전부)로 규칙을 줄이고, 헤더도 전제조건에 적어야 오래된 `.o`가 남지 않는다. Make는 파일 시각만 보므로 옵션을 바꾸면 `make clean`이 필요하다. 빌드가 성공해도 `unsigned char` 오버플로(250+40→34) 같은 논리 버그는 남을 수 있으며, `-g -O0`으로 빌드한 뒤 GDB의 `break` → `run` → `print` → `finish`로 값을 직접 확인하고, clamp로 고친 뒤 `make check`로 검증한다.

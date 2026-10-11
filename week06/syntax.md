# 6주차 문법: Make / GDB

명령어 설명 없이 문법 형태만 빠르게 찾아보는 용도.

## Make (6주차)

### 규칙 기본 형태

```make
타깃: 전제조건1 전제조건2
<TAB>레시피 명령어
```

- 레시피 줄은 **반드시 Tab**으로 시작 (스페이스면 `missing separator`)
- 전제조건이 타깃보다 최신이면 레시피 실행

```make
build/filters.o: src/filters.c include/filters.h
	mkdir -p build
	gcc -Iinclude -c src/filters.c -o build/filters.o
```

### 변수

```make
CC       = gcc
AR       = ar
CPPFLAGS = -Iinclude -Ivendor/stb
CFLAGS   = -std=c11 -Wall -Wextra -g -O0
LDLIBS   = -lm
```

```make
$(CC) $(CPPFLAGS) $(CFLAGS) ...     # Make 변수 읽기: $(이름)
```

(Bash의 `$(명령)`은 명령 결과 치환 — 다른 것)

### 자동 변수

```
$@    타깃
$<    첫 번째 전제조건
$^    전제조건 전부 (중복 제거)
```

### 컴파일 / 아카이브 / 링크 규칙

```make
build/filters.o: src/filters.c include/filters.h
	$(CC) $(CPPFLAGS) $(CFLAGS) -c $< -o $@

lib/libfilters.a: build/filters.o
	$(AR) rcs $@ $^

bin/image-lab: build/main.o build/image_io.o lib/libfilters.a
	$(CC) $(CFLAGS) -o $@ $^ $(LDLIBS)
```

### 헤더 의존성 (전제조건에 헤더도 적기)

```make
build/main.o:     src/main.c include/filters.h include/image_io.h
build/filters.o:  src/filters.c include/filters.h
build/image_io.o: src/image_io.c include/image_io.h vendor/stb/stb_image.h vendor/stb/stb_image_write.h
```

### 이미지 타깃

```make
output/gray.png: bin/image-lab assets/sample.jpg
	./bin/image-lab assets/sample.jpg $@ gray

images: output/gray.png output/bright.png output/invert.png
```

### 가짜 타깃 / clean / check

```make
.PHONY: all images check clean

all: bin/image-lab

check: bin/test_filters bin/image-lab
	./bin/test_filters
	bash tests/check_cli.sh

clean:
	rm -f build/*.o lib/libfilters.a bin/image-lab bin/test_filters output/*.png
```

### make 명령

```
make               # 첫 번째 타깃(all) 빌드
make 타깃           # 지정 타깃 빌드      예: make images
make -n            # 실행 안 하고 할 일만 출력
make clean         # 결과물 삭제
```

```
make: Nothing to be done for 'all'.    # 이미 최신
```

옵션(CFLAGS)을 바꿨을 때 (Make는 파일 시각만 봄):

```
make clean; make images; make check
```

### Make 에러 → 원인

```
missing separator          레시피가 Tab이 아님
filters.h: No such file    -I 경로 / 헤더 위치
undefined reference        링크 입력 / 정의 빠짐
cannot write output        출력 폴더 없음 / 쓰기 불가
```

---


## GDB (6주차)

### 준비 / 시작

```
gcc -g -O0 ...                    # 디버그 정보 포함, 최적화 끄기
gdb ./실행파일
gdb --args ./실행파일 인자1 인자2
```

### 명령어

```
break 함수명                  # 중단점 (줄번호도 가능: break file.c:42)
break 함수명 if 조건           # 조건부 중단점   예: break brighten_channel if value == 250 && delta == 40
run                          # 실행 시작
print 값                      # 값 출력         예: print value, print value + delta
print /u 변수                  # unsigned 형식으로 출력
next                         # 한 줄 실행 (함수 안으로 안 들어감)
step                         # 한 줄 실행 (함수 안으로 들어감)
finish                       # 현재 함수 끝까지 실행 + 리턴값 출력
bt                           # 호출 스택 (backtrace)
continue                     # 다음 중단점까지 계속
quit                         # 종료
```

### 실습 흐름 예

```
cp variants/filters_wrap.c src/filters.c
make bin/test_filters
gdb --args ./bin/test_filters
(gdb) break brighten_channel if value == 250 && delta == 40
(gdb) run
(gdb) print value
(gdb) print delta
(gdb) print value + delta
(gdb) finish
(gdb) bt
```

### 버그 수정 패턴 (clamp)

```c
int result = value + delta;
if (result > 255) result = 255;
if (result < 0)   result = 0;
return (unsigned char)result;
```

수정 후: `make check` → `make images`

### 필터 루프 (RGB 배열, 3칸씩)

```c
for (size_t i = 0; i < count; i += 3) {
    int gray = (rgb[i] + rgb[i+1] + rgb[i+2]) / 3;
    rgb[i] = rgb[i+1] = rgb[i+2] = (unsigned char)gray;
}
```

### Git 추적 구분 (6주차 프로젝트)

```
올림:   src/ include/ Makefile tests/ README.md .gitignore vendor/ assets/
안 올림: build/*.o  lib/libfilters.a  bin/image-lab  bin/test_filters  output/*.png
```

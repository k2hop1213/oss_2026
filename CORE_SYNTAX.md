# 핵심 개념별 문법 (CORE.md 번호와 1:1)

> CORE.md의 각 개념을 **실제로 어떻게 쓰는지**만. 번호가 같으니 나란히 보면 됨.

---

## 2주차 — 셸

| # | 개념 | 문법 |
|---|---|---|
| 1 | 셸이 먼저 처리 | `ls *.c` → 셸이 `ls a.c b.c main.c`로 바꿔서 실행 / `?` 1글자, `[ab]` 하나, `*` 0글자 이상 |
| 2 | `>`는 먼저 비움 | `cmd > f` 덮어쓰기 · `cmd >> f` 추가 |
| 3 | stdout ≠ stderr | `cmd > out.txt 2> err.txt` · `2> /dev/null` · `a \| b` |
| 4 | ls vs find | `ls src/*.c` · `find . -name "*.c"` · `find . -type d` · `find . -size +1M` |
| 5 | grep은 글자 | `grep -c ERROR f` · `-i` 대소문자 무시 · `-v` 반대 · `-n` 줄번호 · `-r` 재귀 · `-q` 조용히 |
| 6 | uniq는 인접만 | `cut -d" " -f4 f \| sort \| uniq -c \| sort -rn` · `sort -n` 숫자 |
| 7 | 하드 vs 심볼릭 | `ln a b` · `ln -s a b` · `ls -li` (inode) · `ls -l` → `l ... b -> a` |
| 8 | ./ + x 권한 | `chmod +x f.sh` → `./f.sh` · `bash f.sh` (x 불필요) |
| 9 | 권한 3글자씩 | `-rw-r--r--` = 타입 / 소유자 rw- / 그룹 r-- / 기타 r-- |
| 10 | root, builtin | 프롬프트 `#` / `$` · `type cd` → builtin · `which ls` → /usr/bin/ls |

**자주 쓰는 조합**
```bash
cut -d: -f7 /etc/passwd | sort | uniq -c | sort -rn
grep -v INFO app.log | grep -v WARN
cp -r data data_backup ; rm -r data_backup ; mkdir -p a/b
tar -xzf oss-w02.tar.gz
```

---

## 3주차 — Git

| # | 개념 | 문법 |
|---|---|---|
| 1 | 세 영역 | `git add f` → `git commit -m "msg"` |
| 2 | add = 사진 | `git add f` 후 수정 → `git status --short` → `MM f` |
| 3 | diff 두 종류 | `git diff` (작업↔스테이징) · `git diff --staged` (스테이징↔커밋) |
| 4 | 상태 코드 | `??` · `A ` · `M ` · ` M` · `MM` · ` D` · `D ` · `R ` |
| 5 | 되돌리기 | `git restore --staged f` → `git restore f` |
| 6 | 삭제 3종 | `rm f` (` D`) · `git rm f` (`D `) · `git rm --cached f` (`D ` + `??`) |
| 7 | .gitignore | `echo "*.log" > .gitignore` → 이미 추적 중이면 `git rm --cached f` → `git add .gitignore` → commit |
| 8 | 결과물 제외 | `.gitignore`에 `build/` `reports/` |
| 9 | 원격 | `git clone URL` · `git fetch` · `git pull --ff-only` · `git push -u origin main` |
| 10 | clean ≠ push | `git status` (로컬) vs `git push` 했는지 |

**보기**
```bash
git log --oneline            # 최신이 위
git show HEAD:f              # 커밋된 내용 (파일 안 바뀜)
git show HEAD~1:f            # 하나 전
git mv a b                   # R
git init -b main
git config user.name "이름"   # 로그인 아님
```

---

## 4주차 — 셸 스크립트

| # | 개념 | 문법 |
|---|---|---|
| 1 | 따옴표 | `grep -c ERROR "$file"` · `"$1"` · `"$@"` · `${count}_x.txt` |
| 2 | `$?` 즉시 | `count=$(grep -c ERROR "$1")` 다음 줄 `status=$?` |
| 3 | grep 상태 | `if [ "$status" -gt 1 ]; then ... exit 1; fi` |
| 4 | if = 상태 | `if [ "$#" -ne 1 ]; then ... fi` · `if grep -q ERROR "$f"; then` · `[ ! -f "$1" ] \|\| [ ! -r "$1" ]` |
| 5 | && \|\| ; | `false && echo A` (없음) · `false \|\| echo B` (B) · `false ; echo C` (C) |
| 6 | stdout/stderr | `echo "$count"` (결과) · `echo "Usage: ..." >&2` (진단) |
| 7 | export | `export COURSE` → `bash -c 'echo "$COURSE"'` |
| 8 | shebang | `#!/usr/bin/env bash` · `bash a.sh` / `./a.sh` |
| 9 | -n / -x | `bash -n a.sh` (문법만) · `bash -x a.sh` (`+` 추적) |
| 10 | 출력·상태 검증 | `[ "$actual_status" -eq "$3" ] && [ "$actual_output" = "$2" ]` |

**뼈대**
```bash
#!/bin/bash
if [ "$#" -ne 1 ]; then echo "Usage: bash c.sh FILE" >&2; exit 1; fi
count=$(grep -c 'ERROR' "$1")
status=$?
if [ "$status" -gt 1 ]; then exit 1; fi
echo "$count"
exit 0
```
```bash
for f in "$@"; do ...; done
while [ "$n" -le 3 ]; do n=$((n + 1)); done
f() { local x=$1; echo "$x"; return 0; }   r=$(f 3)
```
- 비교: `-eq -ne -lt -le -gt -ge` (숫자) · `=` (문자열)
- 특수 변수: `$0` 이름 · `$1` · `$#` 개수 · `"$@"` 전부 · `$?` 상태

---

## 5주차 — gcc

| # | 개념 | 문법 |
|---|---|---|
| 1 | 4단계 | `gcc -E a.c -o a.i` · `-S` → `.s` · `-c` → `.o` · 옵션 없음 → 실행 파일 |
| 2 | 컴파일 = 문법 | `gcc -std=c11 -Wall -Wextra -c a.c` |
| 3 | .o ≠ 프로그램 | `gcc -c main.c -o build/main.o` |
| 4 | 선언/정의 | 헤더 `int is_error(const char *line);` · `.c`에 본문 · `#include "logutil.h"` · `-I.` |
| 5 | nm | `nm build/main.o` → `U is_error` · `nm build/logutil.o` → `T is_error` |
| 6 | 에러 → 단계 | `No such file`(.h) · `expected ';'` · `undefined reference to 'is_error'` |
| 7 | 분리 컴파일 | 아래 3줄 |
| 8 | 헤더 수정 | main.o + logutil.o 둘 다 `-c` 다시 → 재링크 |
| 9 | 정적 .a | `ar rcs build/liblogutil.a build/logutil.o` · `ar t ...` · `gcc build/main.o build/liblogutil.a -o app` |
| 10 | 공유 .so | 아래 4줄 |

```bash
# 7. 분리 컴파일
gcc -std=c11 -Wall -Wextra -I. -c main.c -o build/main.o
gcc -std=c11 -Wall -Wextra -I. -c logutil.c -o build/logutil.o
gcc build/main.o build/logutil.o -o build/logcount

# 10. 공유 라이브러리
gcc -fPIC -I. -c logutil.c -o build/logutil_pic.o
gcc -shared build/logutil_pic.o -o build/liblogutil.so
gcc build/main.o -Lbuild -llogutil -o build/logcount_shared
LD_LIBRARY_PATH="$PWD/build" ./build/logcount_shared data/app.log
```
- `-o` 출력 이름 / `-O0` 최적화 끔 / `-I` 헤더 경로 / `-L` 라이브러리 폴더 / `-lNAME` → libNAME

---

## 6주차 — Make & GDB

| # | 개념 | 문법 |
|---|---|---|
| 1 | 시각 비교 | `make` → `Nothing to be done for 'all'` / `'X' is up to date` |
| 2 | 맨 위 타깃 | `all: bin/image-lab` (맨 위) · `make` · `make images` · `make -n` |
| 3 | 헤더 적기 | `build/filters.o: src/filters.c include/filters.h` |
| 4 | 옵션 변경 | `make clean` → `make` |
| 5 | 자동 변수 | `$(CC) $(CPPFLAGS) $(CFLAGS) -c $< -o $@` · `$(AR) rcs $@ $^` · `$(CC) $(CFLAGS) -o $@ $^ $(LDLIBS)` |
| 6 | .PHONY | `.PHONY: all images check clean` |
| 7 | clamp | `int r = value + delta; if (r > 255) r = 255; if (r < 0) r = 0; return (unsigned char)r;` |
| 8 | -g -O0 | `CFLAGS = -std=c11 -Wall -Wextra -g -O0` |
| 9 | GDB | 아래 |
| 10 | index | `size_t index = ((size_t)y * width + x) * 3;` |

```make
타깃: 전제조건
<Tab>레시피
```
```
gdb ./app            gdb --args ./bin/test_filters        gdb -q -tui ./bin/debug_region
(gdb) break brighten_channel if value == 250 && delta == 40
(gdb) run
(gdb) next | step | finish | continue
(gdb) print /u value | display width
(gdb) bt
(gdb) quit
```
- 계산: 250+40 → **34** · 200+100 → **44** · 20−50 → **226** · 너비 4의 (1,1) → **15**

# OSS 문법 알짜 정리 (2~6주차, 강의·실습 슬라이드 기준)

> 사용법: 오른쪽 설명을 가리고 왼쪽 명령의 뜻을 말해보기 → 반대로 뜻을 보고 명령을 써보기.
> ⚠️ = 오늘 실제로 틀렸던 것

---

## 2주차 — 셸

### 이동·목록
| 명령 | 뜻 |
|---|---|
| `pwd` | 현재 위치 |
| `cd 경로` / `cd ..` / `cd ~` / `cd -` | 이동 / 부모 / 홈 / **직전 위치** |
| `ls -l` | 상세 (타입+권한, 링크수, 소유자, 그룹, 크기, 시각, 이름) |
| `ls -a` | 숨김(`.`으로 시작) 포함 |
| `ls -lh` | 크기를 K/M/G로 (**-l과 같이**) |
| `ls -t` / `-r` / `-S` | 시간순 / 역순 / 크기순 |
| `ls -d 폴더` | 폴더 **자체** 정보 |
| `ls -li` | inode 번호 |
| `ls -lhS /var/log \| head -2` | 가장 큰 파일 |

### 파일 조작
| 명령 | 뜻 |
|---|---|
| `mkdir -p a/b/c` | 중간 폴더까지 생성 |
| `touch f` | 빈 파일 / 시각 갱신 |
| `cp a b` / `cp -r dir dir2` | 복사 / **디렉토리 복사는 -r** |
| `mv a b` | 이름 변경 = 이동 |
| `rm f` / `rm -r dir` / `rm -ri dir` | 삭제 / 폴더 삭제 / 하나씩 확인 (**복구 불가**) |

### 읽기·도움말
| 명령 | 뜻 |
|---|---|
| `cat` / `less` | 전체 출력 / 스크롤 (q 종료, /검색) |
| `head -n 5 f` / `tail -n 3 f` / `tail -f f` | 앞 5줄 / 뒤 3줄 / 실시간 |
| `wc f` → `19 21 888 f` | **줄 단어 바이트** |
| `wc -l f` | 줄 수만 |
| `cmd --help` | 사용법 |
| `type cd` / `which ls` | builtin인지 / 실행 파일 경로 |

### 와일드카드 (셸이 **실행 전에** 확장)
| 패턴 | 뜻 | `a.c a.o b.c b.o main.c`에서 |
|---|---|---|
| `*` | 0글자 이상 | `*.c` → a.c b.c main.c |
| `?` | **정확히 1글자** | `?.c` → a.c b.c |
| `[ab]` | 그중 1글자 | `[ab].o` → a.o b.o |
| `[a-z]` | 범위 중 1글자 | `*.[co]` → 5개 전부 |

### 리다이렉션·파이프
| 기호 | 뜻 |
|---|---|
| `> f` | stdout을 파일에 **덮어쓰기** (실행 전에 파일 비움) |
| `>> f` | 끝에 추가 |
| `2> f` | stderr를 파일에 |
| `< f` | 파일을 stdin으로 |
| `a \| b` | a의 stdout → b의 stdin |
| `2> /dev/null` | 에러 버리기 |

### grep / cut / sort / uniq / find
| 명령 | 뜻 |
|---|---|
| `grep ERROR f` | 매칭 줄 |
| `grep -c` / `-i` / `-n` / `-v` | 개수 / 대소문자 무시 / 줄번호 / **반대** |
| `grep -r TODO docs` / `-l` / `-rn` | 재귀 / 파일명만 / 파일:줄:내용 |
| `grep -q` | 출력 없이 **상태만** (if용) |
| `grep -c ERROR *.log` | 파일별: `app.log:12` `old.log:3` |
| `cut -d: -f7` / `cut -d" " -f4` | 구분자 `:`의 7번째 / 공백의 4번째 필드 |
| `sort` / `sort -n` / `sort -rn` | 글자순 (`100`<`21`) / 숫자순 / 숫자 큰 순 |
| `uniq -c` | **인접** 중복 합치고 개수 → 먼저 `sort` |
| `find . -name "*.c"` | 트리 전체 (따옴표 필수) |
| `find . -type f` / `-type d` / `-size +1M` | 파일만 / 폴더만 / 1MB 초과 |

**빈도 순위 공식** ⚠️ 파일명 빼먹지 말기
```bash
cut -d: -f7 /etc/passwd | sort | uniq -c | sort -rn
grep ERROR app.log | cut -d" " -f4 | sort | uniq -c | sort -rn
grep -v INFO app.log | grep -v WARN          # INFO도 WARN도 아닌 줄
```

### 링크·권한·실행
| 명령 | 뜻 |
|---|---|
| `ln a b` | 하드 링크 (같은 inode, 원본 지워도 데이터 유지) |
| `ln -s a b` | 심볼릭 링크 ⚠️ **경로만 저장** (`l`, `b -> a`). 원본 지우면 깨짐, **다시 만들면 재연결** |
| `-rw-r--r--` | ⚠️ 타입 / 소유자 rw / **그룹 r** / 기타 r |
| `chmod +x f` | 실행 권한 (디렉토리 x = cd 가능) |
| `bash f.sh` | x 권한 불필요 |
| `./f.sh` | **x 권한 필요** |
| `f.sh` | ⚠️ command not found (PATH에 `.` 없음) |
| `tar -xzf a.tar.gz` | ⚠️ x 풀기 / z gzip / f 이 파일 |
| `nano f` | Ctrl+O 저장, Ctrl+X 종료 |
| Tab · ↑↓ · Ctrl+R · Ctrl+C · Ctrl+L · Ctrl+A/E | 자동완성 · 히스토리 · 히스토리 검색 · 취소 · 화면 지움 · 줄 처음/끝 |

프롬프트 ⚠️ `$` = 일반 사용자 / `#` = root

---

## 3주차 — Git

### 시작
```bash
git init -b main
git config user.name "Student"          # 작성자 이름만 (로그인 X)
git config user.email "a@b.com"
```

### 기록
| 명령 | 뜻 |
|---|---|
| `git status` / `--short` | 상태 / 짧게 |
| `git add f` | 그 순간 내용을 스테이징 (사진 찍기) |
| `git commit -m "msg"` | 스테이징된 것만 커밋 |
| `git diff` | **작업 폴더 ↔ 스테이징** |
| `git diff --staged` | **스테이징 ↔ 마지막 커밋** |
| `git log --oneline` | 히스토리 (**최신이 위**) |
| `git log -2 --oneline --stat` | 최근 2개 + 바뀐 파일 |
| `git log --oneline -- f` | 그 파일이 바뀐 커밋만 |
| `git show HEAD:f` / `git show 커밋:f` | 커밋된 내용 보기 (파일은 안 바뀜) |
| `git show :f` | 스테이징에 있는 내용 |
| `git show --stat HEAD` | 커밋에 들어간 파일 |

### 상태 코드 (왼쪽 = 커밋↔스테이징, 오른쪽 = 스테이징↔작업 폴더)
| 코드 | 상황 |
|---|---|
| `??` | 추적 안 함 |
| `A ` | 새 파일 add |
| `M ` | 수정 후 add |
| ` M` | ⚠️ **수정만, add 안 함** |
| `MM` / `AM` | add 후 **또** 수정 |
| ` D` | 그냥 `rm` (스테이징엔 남음) |
| `D ` | `git rm` |
| `R ` | ⚠️ `git mv` |
| `!!` | 무시됨 (`--ignored`) |

### 되돌리기·삭제
| 명령 | 작업 폴더 | 스테이징 |
|---|---|---|
| `git restore --staged f` | 그대로 | add 취소 |
| `git restore f` | **스테이징 버전으로 되돌림** (수정 버림) | 그대로 |
| `rm f` | 삭제 | 그대로 (` D`) |
| `git rm f` | 삭제 | 삭제 (`D `) |
| `git rm --cached f` | ⚠️ **그대로** | 삭제 (추적만 끊음) |
| `git mv a b` | 이름 변경 | 스테이징까지 (`R`) |

- 실수로 덮어쓰고 add까지: `git restore --staged f` → `git restore f` (순서!)
- `.gitignore`는 **untracked에만** 적용 → 이미 추적 중이면 `git rm --cached` 먼저

### 원격
```bash
git clone URL
git remote -v
git remote add origin URL
git push -u origin main     # 처음
git push                    # 이후
git fetch                   # 받기만 (통합 X)
git pull --ff-only          # fetch + 통합, 갈라졌으면 중단
```
- clean ≠ push됨 (로컬 상태일 뿐)

### Classroom 제출
```bash
gh auth status
gh auth setup-git
git clone 과제URL 폴더 && cd 폴더
# 수정 → git add → git commit
git push
```

---

## 4주차 — 셸 스크립트

### 실행
| 명령 | 인터프리터 | x 권한 |
|---|---|---|
| `bash a.sh` | bash | 불필요 |
| `./a.sh` | shebang `#!/usr/bin/env bash` | **필요** |
| `sh a.sh` | sh | 불필요 |
| `bash -n a.sh` | **문법만 검사 (실행 X)** | |
| `bash -x a.sh` | 실행 + 추적 (`+`, `$( )` 안은 `++`, stderr) | |

### 변수·따옴표
```bash
name="log check"          # = 양쪽 공백 금지
echo "$name"              # 확장 → log check
echo '$name'              # 그대로 → $name
report="${count}_errors.txt"   # ⚠️ 중괄호 = 변수 이름 경계
export COURSE             # 이후 자식 프로세스에 복사본 전달
grep -c ERROR "$file"     # ⚠️ 따옴표 없으면 공백에서 쪼개짐
```

### 특수 변수
| 기호 | 뜻 |
|---|---|
| `$0` | 스크립트 이름 |
| `$1 $2` | 1·2번째 인자 |
| `$#` | 인자 개수 (`a "b c" d` → 3) |
| `"$@"` | 인자 전부, **경계 유지** |
| `$?` | ⚠️ **바로 앞 명령**의 상태 → `status=$?` 즉시 저장 |

### 치환·입출력
```bash
count=$(grep -c ERROR "$f")   # 명령 출력 캡처 (끝 줄바꿈 제거)
next=$((count + 1))           # 정수 산술, $((3/2)) = 1
read -r name                  # 실행 중 stdin에서 한 줄
echo "error" >&2              # stderr로
bash c.sh > out.txt 2> err.txt
```

### 조건
```bash
if [ "$#" -ne 1 ]; then
  echo "Usage: bash c.sh FILE" >&2
  exit 1
elif [ "$count" -lt 5 ]; then
  echo small
else
  echo many
fi
```
| 테스트 | 뜻 |
|---|---|
| `-eq -ne` | 같다 / 다르다 |
| `-lt -le -gt -ge` | ⚠️ < ≤ > ≥ (끝이 e면 같음 포함) |
| `=` | 문자열 같음 |
| `-f` / `-r` | 일반 파일 / 읽기 가능 |
| `!` | 뒤집기 |
- `[ ]` = `test` 명령, **안쪽 공백 필수**
- `if grep -q ERROR "$f"; then` → 명령의 종료 상태로 분기

### 명령 리스트
| | 뒤 명령 실행 |
|---|---|
| `a ; b` | 항상 |
| `a && b` | a 성공(0) |
| `a \|\| b` | a 실패(≠0) |
- ⚠️ `false && echo A || echo B` → **B**

### 반복·함수
```bash
for f in "$@"; do echo "$f"; done
while [ "$n" -le 3 ]; do n=$((n + 1)); done
continue   # 이번 회차 건너뜀
break      # 루프 종료
exit 2     # 스크립트 종료

count_errors() {
  local file=$1
  grep -c ERROR "$file"     # 결과는 stdout
  return 0                  # 상태는 return
}
c=$(count_errors data/events.log)
```

### grep 종료 상태 ⚠️
| 상황 | 출력 | `$?` |
|---|---|---|
| 찾음 | 개수 | 0 |
| 못 찾음 | 0 | **1 (정상)** |
| 에러 | 없음 | 2 |
→ 에러 검사는 `[ "$status" -gt 1 ]`

### 실습 결함 → FAIL ⚠️
| 결함 | FAIL |
|---|---|
| `-gt 1` → `-ge 1` | clean, empty |
| stdout에 echo 추가 | 정상 4개 전부 |
| `"$1"` → `$1` | day one.log만 |

---

## 5주차 — gcc

### 단계 (E-S-c)
| 옵션 | 멈추는 곳 | 출력 |
|---|---|---|
| `-E` | 전처리 | `.i` |
| `-S` | 컴파일 | `.s` |
| `-c` | 어셈블 | `.o` |
| (없음) | 링크 | 실행 파일 |
| `-o 이름` | 출력 이름 (⚠️ `-O0`과 다름) | |

### 빌드
```bash
gcc -std=c11 -Wall -Wextra -I. -c main.c -o build/main.o
gcc -std=c11 -Wall -Wextra -I. -c logutil.c -o build/logutil.o
gcc build/main.o build/logutil.o -o build/logcount     # ⚠️ 경로까지
./build/logcount data/app.log ; echo $?
```
| 옵션 | 뜻 |
|---|---|
| `-std=c11` | C 표준 |
| `-Wall -Wextra` | 경고 |
| `-I.` / `-Iinclude` | ⚠️ 헤더 검색 경로 (대시 필수) |

### 심볼·라이브러리
| 명령 | 뜻 |
|---|---|
| `nm a.o` | `T` 정의됨 / `U` 정의 없음 (호출해야 U) |
| `ar rcs build/liblogutil.a build/logutil.o` | ⚠️ r 넣기/교체, c 생성, s 심볼 인덱스 |
| `ar t build/liblogutil.a` | 멤버 목록 |
| `gcc build/main.o build/liblogutil.a -o app` | 정적 링크 (**호출자 먼저**) |
| `gcc build/main.o -Lbuild -llogutil -o app` | `-L` 폴더, `-lNAME` → libNAME |

### 공유 라이브러리
```bash
gcc -fPIC -I. -c logutil.c -o build/logutil_pic.o
gcc -shared build/logutil_pic.o -o build/liblogutil.so
gcc build/main.o -Lbuild -llogutil -o build/logcount_shared
LD_LIBRARY_PATH="$PWD/build" ./build/logcount_shared data/app.log
```
⚠️ 링크 때 `-L`, **실행 때 `LD_LIBRARY_PATH`**

### 에러 → 단계
| 에러 | 단계 |
|---|---|
| `No such file or directory` (헤더) | 전처리 |
| `expected ';'` | 컴파일 |
| `undefined reference to 'is_error'` | 링크 |
| 결과 3 (기대 2) | 동작 (strstr vs strncmp) |

### 재빌드 ⚠️
| 바뀐 것 | 다시 컴파일 |
|---|---|
| logutil.c | logutil.o → 재링크 |
| main.c | main.o → 재링크 |
| logutil.h | **main.o + logutil.o** → 재링크 |

---

## 6주차 — Make & GDB

### Makefile
```make
CC       = gcc
AR       = ar
CPPFLAGS = -Iinclude -Ivendor/stb
CFLAGS   = -std=c11 -Wall -Wextra -g -O0
LDLIBS   = -lm

all: bin/image-lab

build/filters.o: src/filters.c include/filters.h
	mkdir -p build
	$(CC) $(CPPFLAGS) $(CFLAGS) -c $< -o $@

lib/libfilters.a: build/filters.o
	$(AR) rcs $@ $^

bin/image-lab: build/main.o build/image_io.o lib/libfilters.a
	$(CC) $(CFLAGS) -o $@ $^ $(LDLIBS)

.PHONY: all images check clean
```
- `타깃: 전제조건` / 레시피는 **Tab** (아니면 `missing separator`)
- Make `$(NAME)` = 변수 읽기 / Bash `$(cmd)` = 명령 결과 → 다름

### 자동 변수 ⚠️
| | 뜻 | `app: main.o util.o` |
|---|---|---|
| `$@` | 타깃 | app |
| `$<` | 첫 전제조건 → **컴파일** | main.o |
| `$^` | 전부 → **링크, ar** | main.o util.o |

### 명령
| 명령 | 뜻 |
|---|---|
| `make` | **맨 위 타깃** (보통 all) |
| `make 타깃` | 그 타깃만 (재료 중 오래된 것만) |
| `make -n` | ⚠️ 실행 안 하고 명령만 출력 |
| `make clean` | 생성 파일 삭제 |
| `make check` / `make images` | 테스트 / PNG |
| `Nothing to be done` / `is up to date` | 이미 최신 |
- ⚠️ CFLAGS만 바꾸면 Make는 모름 → `make clean` 후 다시
- ⚠️ `.PHONY` 없고 `clean` 파일 있으면 → `'clean' is up to date` (실행 안 됨)

### GDB
```bash
gdb ./app
gdb --args ./bin/test_filters
gdb -q -tui ./bin/debug_region
```
| 명령 | 뜻 |
|---|---|
| `break 함수` | 중단점 (**run 전에**) |
| `break f if value == 250 && delta == 40` | 조건부 |
| `run` | 실행 시작 |
| `next` | ⚠️ 한 줄, 함수 **건너뜀** |
| `step` | ⚠️ 한 줄, 함수 **안으로** |
| `finish` | 함수 끝까지 + **리턴값** |
| `continue` | 다음 중단점까지 |
| `print x` / `print /u x` | 한 번 / unsigned로 |
| `display x` | ⚠️ **멈출 때마다** 자동 |
| `bt` | ⚠️ 호출 스택 |
| `quit` | 종료 |
- 빌드: ⚠️ **`-g`**(디버그 정보) **`-O0`**(최적화 끔)

### 계산 ⚠️
- `unsigned char` 0~255, 넘으면 ±256: 250+40=**34**, 200+100=**44**, 20−50=**226**
- clamp: `int`로 계산 → 255/0으로 자르기 → 마지막에 변환
- index = `((size_t)y * width + x) * 3` → 너비 4, (1,1) = **15**
- stride 버그 `y * height` → 정사각형만 통과

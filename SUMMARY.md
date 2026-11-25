# OSS 퀴즈 직전 알짜 요약 (2~6·9~12주차)

> 시험 직전 10분: **⚠️ 표시**만 다시 보기. 내가 실제로 틀렸던 것들.

---

## 0. ⚠️ 내가 틀렸던 것 TOP 12

1. `$?`는 **바로 앞 명령** 결과 → `echo` 한 번만 해도 0으로 덮임 → `status=$?`로 즉시 저장
2. `false && echo A || echo B` → **B** (A && B || C = A 성공→B, 실패→C)
3. 커밋 후 수정, add 안 함 → **` M`** (앞에 공백). `MM`은 add 후 **또** 수정
4. `git mv` → **R** (이름 변경 + 스테이징까지)
5. 이미 추적 중인 파일 → `.gitignore` 무시됨 → **`git rm --cached 파일`** 먼저
6. 프롬프트 **`#` = root**, `$` = 일반 사용자
7. `ls -l` 맨 앞 **`l` = 심볼릭 링크** (`-` 파일, `d` 디렉토리)
8. 심볼릭 링크 = **경로만 저장** → 원본 지우면 깨짐, 같은 이름 다시 만들면 **다시 연결**
9. `chmod +x` 해도 `hello.sh`(./ 없이) → **command not found** (PATH에 `.` 없음)
10. `-rw-r--r--` = 타입 / 소유자 rw / **그룹 r** / 기타 r. 디렉토리 x = **cd 진입 가능**
11. `tar -xzf` = **x 풀기 / z gzip / f 이 파일**
12. **답안에 파일명·대시 빼먹지 말기** (`-I.`, `/etc/passwd` 등)

---

## 1. 2주차 — 리눅스 셸

### 기본
| 개념 | 핵심 |
|---|---|
| 터미널 vs 셸 | 터미널 = 창, 셸(bash) = 명령 해석 프로그램 |
| 명령 구조 | `명령 옵션 인자` / 짧은 옵션 합치기 `-lah`, 긴 옵션 `--all`은 못 합침 |
| 절대경로 | `/`로 시작 |
| 상대경로 | 현재 위치 기준 `.` 현재 `..` 부모 `~` 홈 |
| `cd -` | **직전 위치**로 |
| `cd`가 builtin인 이유 | 별도 프로그램은 부모 셸의 위치를 못 바꿈 |

### FHS
`/etc` 설정 · `/var/log` 로그 · `/usr/bin` 명령 · `/tmp` 임시 · `/dev` 장치(`/dev/null`) · `/proc` 커널정보 · `/home` 사용자 홈 · `/root` root 홈 · `/bin -> usr/bin` (심볼릭 링크)

### ls 옵션
`-l` 상세 · `-a` 숨김 포함 · `-h` 크기 단위(**-l 필요**) · `-t` 시간순 · `-r` 역순 · `-S` 크기순 · `-d` 디렉토리 자체

`ls -l` 열: `타입+권한 / 링크수 / 소유자 / 그룹 / 크기 / 시각 / 이름`

### 파일 명령
`mkdir -p` 중간 경로까지 · `touch` 빈 파일/시각 갱신 · `cp -r` 디렉토리 복사 · `mv` 이동=이름변경 · `rm -r` 디렉토리 삭제 (**되돌릴 수 없음**, 먼저 `ls 패턴`)

### 읽기
`cat` 전체 · `head -n 5` · `tail` / `tail -f` 실시간 · `less` 큰 파일 · `wc` = **줄 단어 바이트** 순

### 도움말
`--help` · `type ls` (builtin/외부 구분) · `which ls` (실행 파일 경로)

### 와일드카드 (셸이 **실행 전에** 확장)
`*` 0글자 이상 · `?` **정확히 1글자** · `[ab]` 하나 · `[a-z]` 범위 중 1글자
- 파일 `a.c a.o b.c b.o main.c` → `ls ?.c` = a.c b.c / `ls *.[co]` = 전부 5개

### 리다이렉션 & 파이프
`>` 덮어쓰기 · `>>` 추가 · `2>` 에러 · `<` 입력 · `|` 앞 stdout → 뒤 stdin
- `>`는 **명령 실행 전에** 파일을 비움

### grep
`-c` 개수 · `-i` 대소문자 무시 · `-n` 줄번호 · `-v` 반대 · `-r` 재귀 · `-l` 파일명만 · `-q` 출력 없이 상태만
- `grep -c ERROR *.log` → `app.log:12` / `old.log:3` (파일별, 합계 아님)
- grep은 **의미가 아니라 글자**를 봄 (`no error detected`도 `-i error`에 걸림)

### find (트리 전체) vs ls (한 디렉토리)
`find . -name "*.c"` (따옴표 필수) · `-type f/d` · `-size +1M` · `2> /dev/null`

### 빈도 순위 관용구
```bash
grep ERROR app.log | cut -d" " -f4 | sort | uniq -c | sort -rn
cut -d: -f7 /etc/passwd | sort | uniq -c | sort -rn
```
- `uniq`는 **인접한** 중복만 → `sort` 먼저
- `sort`는 텍스트 순 (`100` < `21`) → 숫자면 `-n`

### 링크
| | 하드 `ln a b` | 심볼릭 `ln -s a b` |
|---|---|---|
| 저장 | 같은 데이터(inode) | **경로 문자열** |
| `ls -l` | `-`, 링크수 2 | `l`, `b -> a` |
| 원본 삭제 | 데이터 유지 | 깨짐 |
| 원본 이름 재생성 | 무관 | **다시 연결** |

### 실행
- `bash hello.sh` → x 권한 불필요
- `./hello.sh` → **x 권한 필요** (`chmod +x`)
- `hello.sh` → command not found (PATH에 `.` 없음)

### 단축키
Tab 자동완성 · ↑↓ 히스토리 · Ctrl+R 히스토리 검색 · Ctrl+C 취소 · Ctrl+L 화면 지움 · Ctrl+A/E 줄 처음/끝
- `.bashrc` = 셸 시작마다 실행 (`export LANG=C.UTF-8`, `alias`) / 점(.)으로 시작 = 숨김 → `ls -a`
- nano: Ctrl+O 저장, Ctrl+X 종료

---

## 2. 3주차 — Git

### 세 영역
**작업 폴더**(현재 파일) → `git add` → **스테이징**(다음 커밋 후보) → `git commit` → **저장소**(히스토리)
- `add`는 **그 순간 내용을 사진 찍음**. 이후 수정은 커밋 안 됨
- clean ≠ GitHub에 올라감 → **push** 해야 함

### status --short
| 코드 | 뜻 |
|---|---|
| `??` | 추적 안 함 (untracked) |
| `A ` | 새 파일 스테이징 |
| `M ` | 수정 후 스테이징 |
| ` M` | **수정만, add 안 함** |
| `MM` | add 후 또 수정 |
| `AM` | 새 파일 add 후 또 수정 |
| ` D` / `D ` | 그냥 삭제 / `git rm`으로 삭제 |
| `R` | 이름 변경 (`git mv`) |
| `!!` | 무시됨 (`--ignored`일 때) |

왼쪽 글자 = 커밋↔스테이징 / 오른쪽 글자 = 스테이징↔작업 폴더

### diff / show / log
- `git diff` = **작업 폴더 ↔ 스테이징**
- `git diff --staged` = **스테이징 ↔ 마지막 커밋**
- 새 파일은 add 전엔 `git diff`에 안 나옴
- `git show HEAD:파일` 커밋된 내용 / `git log --oneline` **최신이 위**

### 되돌리기
| 명령 | 효과 |
|---|---|
| `git restore --staged f` | 스테이징만 취소, 파일 유지 |
| `git restore f` | 작업 파일을 **스테이징 버전**으로 되돌림 (수정 버림) |
| `git rm f` | 파일 **삭제** + 삭제 스테이징 |
| `git rm --cached f` | 추적만 끊음, 디스크엔 남음 |

- 실수로 `WRONG` 쓰고 add까지 → **`restore --staged` → `restore`** 순서

### .gitignore
- **untracked 파일에만** 적용. 이미 추적 중이면 `git rm --cached` 먼저
- 생성 결과물(`build/`, `reports/`, `*.o`)은 무시 — **다시 만들 수 있는 건 안 올림**

### 설정
`git init -b main` · `git config user.name "이름"` (**로그인 아님**, 작성자 이름만)

### 예측 문제 (빈출)
```
현재 커밋 A → echo B > f; git add f; echo C > f; git commit
```
커밋 = **B** / 작업 파일 = **C** / 남은 변경 = B→C (` M`)

### 브랜치 / 원격
- 브랜치 = 커밋을 가리키는 이동식 포인터 / merge 커밋은 부모 **2개**
| 명령 | 역할 |
|---|---|
| clone | 원격 복제 |
| push | 내 커밋 업로드 |
| fetch | 받기만 (`origin/main` 갱신, **통합 X**) |
| pull | fetch + 통합 |
- `git pull --ff-only`: fast-forward 안 되면 멈춤
- fast-forward = 내 브랜치에 새 커밋이 없어서 포인터만 앞으로 이동
- `git remote add origin URL` → `git push -u origin main`

---

## 3. 4주차 — 셸 스크립트

### 실행 방식
| 명령 | 인터프리터 | x 권한 |
|---|---|---|
| `bash a.sh` | bash | 불필요 |
| `./a.sh` | **shebang** (`#!/usr/bin/env bash`) | **필요** |
| `sh a.sh` | sh | 불필요 |
- `.sh` 확장자는 bash를 선택하지 않음

### 변수
- `name=val` (**= 양쪽 공백 금지**)
- `"$name"` 확장 / `'$name'` 그대로 / `${count}_errors.txt` 경계 표시
- `export` = 이후 **자식** 프로세스에 복사본 전달 (자식 변경은 부모에 안 돌아옴)
- 따옴표 없으면 공백에서 쪼개짐 → 경로는 항상 `"$1"`

### 인자
`$0` 스크립트 이름 · `$1 $2` · `$#` 개수 · `"$@"` 각 인자 경계 유지
- `./run.sh a "b c" d` → `$#` = 3

### 입력·출력
- `read -r name` 실행 중 stdin에서 한 줄 읽음
- `$(cmd)` 출력 캡처 (끝 줄바꿈 제거) · `$((3/2))` = **1** (정수)
- stdout = 결과 / stderr = 진단 → `>&2`
- `bash x.sh > out.txt 2> err.txt`

### 종료 상태
- 0 = 성공, 그 외 = 실패
- **grep: 0 찾음 / 1 못 찾음(정상) / 2 이상 에러** → 에러 검사는 **`-gt 1`**
- `$?` 즉시 저장: `count=$(grep -c ...)` 다음 줄 `status=$?`

### 조건
```bash
if [ "$#" -ne 1 ]; then ... elif ...; else ...; fi
```
- `if`는 **명령의 종료 상태**로 분기 (`if grep -q ...`)
- `[ ]` = `test`, **안쪽 공백 필수**
- 숫자: `-eq -ne -lt -le -gt -ge` / 문자열: `=` / 파일: `-f` 일반파일 `-r` 읽기가능 / `!` 부정
- `;` 무조건 · `&&` 성공 시 · `||` 실패 시

### 반복
```bash
for f in "$@"; do ...; done
while [ "$n" -le 3 ]; do n=$((n+1)); done
```
- `continue` 이번 회차 건너뜀 / `break` 루프 종료 / `exit` 스크립트 종료
- `1 2 3 4`에서 2 continue, 4 break → 출력 **1 3**

### 함수
```bash
f() { local x=$1; echo "$x"; return 0; }
r=$(f 3)   # 결과는 echo(stdout), 상태는 return
```

### 디버그
`bash -n` 문법만 검사(실행 X) · `bash -x` 실행하며 추적(stderr, `+`)

### W04B 실습 핵심
- count_errors.sh: 인자 1개 검사 → `-f`/`-r` 검사 → `count=$(grep -c 'ERROR' "$1")` → `status=$?` → `-gt 1`이면 exit 1 → `echo "$count"; exit 0`
- 결과표: events 3/0 · clean 0/0 · empty 0/0 · day one 1/0 · missing (빈칸)/1
- **결함 → FAIL 패턴**
  | 결함 | FAIL |
  |---|---|
  | `-ge 1` | clean, empty |
  | stdout에 echo 추가 | 정상 4개 전부 |
  | `$1` 따옴표 없음 | day one.log만 |
- `> summary.txt`는 스크립트 시작 **전에** 파일을 비움 → 임시파일 + `mv`
- verify: stdout은 **문자열** 비교(`=`), status는 **숫자** 비교(`-eq`). `failures=1`은 다시 0으로 안 돌아감

---

## 4. 5주차 — gcc & 라이브러리

### 빌드 4단계 (외우기: **E-S-c**)
| 단계 | 출력 | 멈춤 옵션 |
|---|---|---|
| 전처리 | `.i` | `-E` |
| 컴파일 | `.s` | `-S` |
| 어셈블 | `.o` | `-c` |
| 링크 | 실행파일 | 없음 |
- 전처리 = `#include` 붙이기, `#define` 매크로 치환
- 컴파일 = 문법·타입 검사 → 어셈블리
- `.o`는 최종 프로그램이 **아님** (링크 정보 포함)
- `-o` = 출력 이름 (`-O0` 최적화 끔이랑 다름!)

### 선언 vs 정의
- 헤더(`.h`) = **선언** (이름, 매개변수, 반환형) / `.c` = **정의** (본문)
- 헤더만으론 링크 불가 → 정의가 든 `.o`가 필요
- 헤더 가드: `#ifndef X_H / #define X_H / ... / #endif`

### 분리 컴파일
```bash
gcc -std=c11 -Wall -Wextra -I. -c main.c -o build/main.o
gcc -std=c11 -Wall -Wextra -I. -c logutil.c -o build/logutil.o
gcc build/main.o build/logutil.o -o build/logcount
```
- `-I.` 헤더 검색 경로 · `-Wall -Wextra` 경고 (정확성 보장 X)
- 이유: 바뀐 파일만 다시 컴파일 → 빠름

### nm
`T` = 여기 **정의**됨 / `U` = 쓰는데 정의 **없음**(Undefined)

### 실패 단계
| 증상 | 단계 |
|---|---|
| `expected ';'` | 컴파일 |
| `undefined reference` | **링크** (정의 든 .o 빠짐) |
| 개수가 2가 아니라 3 | 프로그램 동작 (로직) |

- `strstr`(어디든 포함) vs `strncmp`(접두사) → `INFO no ERROR detected` 때문에 3. **컴파일은 문법만 봐서** 성공
- 경계값 예: 18세 이상 → `>=` vs `>` 구분은 age=18

### logcount 계약
| 상황 | stdout | status |
|---|---|---|
| 매칭 2줄 | 2 | 0 |
| 0줄 | 0 | **0** (정상) |
| 인자 없음 | 없음 | **2** |
| 파일 없음 | 없음 | **3** |

### 재빌드
| 바뀐 것 | 다시 컴파일 |
|---|---|
| logutil.c | logutil.o → 재링크 |
| main.c | main.o → 재링크 |
| **logutil.h** | **main.o + logutil.o** → 재링크 |
- 소스만 바꾸고 실행 → **옛날 실행파일** 그대로. `.o`만 새로 만들고 실행 → 여전히 옛날. **재컴파일 → 재링크 → 실행**
- 헤더 `ERROR` → `WARN` 바꾸면 app.log 결과 **1** (`WARN memory`)

### 정적 라이브러리
```bash
ar rcs build/liblogutil.a build/logutil.o   # r 넣기/교체, c 생성, s 심볼 인덱스
ar t build/liblogutil.a                     # 멤버 목록
gcc build/main.o build/liblogutil.a -o app
gcc build/main.o -Lbuild -llogutil -o app   # -L 검색 폴더, -lNAME → libNAME
```
- **호출하는 쪽(main.o)이 라이브러리보다 앞**

### 공유 라이브러리
```bash
gcc -fPIC -I. -c logutil.c -o build/logutil_pic.o
gcc -shared build/logutil_pic.o -o build/liblogutil.so
gcc build/main.o -Lbuild -llogutil -o build/logcount_shared
LD_LIBRARY_PATH="$PWD/build" ./build/logcount_shared data/app.log
```
| | 정적 `.a` | 공유 `.so` |
|---|---|---|
| 실행 시 라이브러리 파일 | 불필요 | **필요** |
| 위치 지정 | 링크: `-L` | 링크: `-L` / **실행: `LD_LIBRARY_PATH`** |

### Git
`.gitignore`에 `build/` → 소스·헤더·check.sh·README만 추적

---

## 5. 6주차 — Make & GDB

### Makefile 규칙
```make
타깃: 전제조건
<Tab>레시피
```
- **레시피는 Tab** (스페이스면 `missing separator`)
- 전제조건이 타깃보다 **최신**이거나 타깃이 없으면 실행 (**수정 시각만** 봄)
- `make` = 첫 타깃 / `make 타깃` / `make -n` = 실행 안 하고 명령만 출력

### 변수
`CC = gcc` · `CPPFLAGS = -Iinclude` · `CFLAGS = -std=c11 -Wall -Wextra -g -O0` · `LDLIBS = -lm` · `AR = ar` → `$(CC)`로 읽기

### 자동 변수 (`app: main.o util.o`)
| | 뜻 | 값 |
|---|---|---|
| `$@` | 타깃 | app |
| `$<` | 첫 전제조건 | main.o |
| `$^` | 전제조건 전부 | main.o util.o |
- 컴파일엔 `$<`, 링크·`ar`엔 `$^`, `$(LDLIBS)`는 **맨 뒤**

### 재빌드 판단
- 헤더를 전제조건에 안 넣으면 → 헤더 바꿔도 `.o` 안 바뀜 (stale)
- **CFLAGS만 바꾸면 Make는 모름** → `make clean` 후 다시
- `Nothing to be done` / `is up to date` = 이미 최신

### .PHONY
`clean`, `check`, `all`, `images`는 파일이 아니라 **동작 이름** → `.PHONY: clean ...`
- 안 쓰면 `clean`이라는 파일이 있을 때 **"is up to date"** 되고 실행 안 됨

### unsigned char 오버플로 (0~255)
넘으면 **±256**: 250+40 = **34** · 240+30 = **14** · 200+100 = **44** · 20−50 = **226**
- 수정: `int`로 계산 → 0~255로 자르기(clamp) → 마지막에 변환

### GDB
- 빌드: **`-g`** (디버그 정보) **`-O0`** (최적화 끔)
- `gdb --args ./bin/test_filters`
| 명령 | 뜻 |
|---|---|
| `break 함수` / `break f if v==250` | 중단점 (조건부) |
| `run` | 처음부터 실행 |
| `print x` / `print /u x` | 값 보기 / unsigned로 |
| `display x` | 멈출 때마다 자동 출력 |
| `next` | 한 줄 (함수 **건너뜀**) |
| `step` | 한 줄 (함수 **안으로**) |
| `finish` | 함수 끝까지 + **리턴값** |
| `bt` | 호출 스택 |
| `continue` | 다음 중단점까지 |

### 에러 → 의심할 곳
`missing separator` → Tab · `No such file (.h)` → `-I` · `undefined reference` → 링크 입력 · 밝은 곳이 어두워짐 → 오버플로

### region 실습
- index = `((size_t)y * width + x) * 3` (RGB 3바이트)
- stride 버그(`y * height`): 정사각형만 통과 (width == height라 티가 안 남)

### Git
올림: `src/ include/ Makefile tests/ README.md .gitignore` / 안 올림: `build/*.o lib/*.a bin/ output/*.png`

---

## 6. 9주차 — 파일 입출력

### 두 층
| | fd | FILE \* |
|---|---|---|
| 함수 | open/read/write/close | fopen/getline/fprintf/fclose |
| 정체 | 정수 번호 (보통 3) | fd + **버퍼** |
- fd 0 stdin / 1 stdout / 2 stderr

### 반환값 ⚠️
| 함수 | 성공 | EOF | 실패 |
|---|---|---|---|
| open | fd ≥ 0 | | **-1** |
| read | 바이트 수 | **0** | -1 |
| getline | 글자 수 | **-1** | -1 → `ferror`로 구분 |
| write | 쓴 수 (**일부만** 가능) | | -1 |
| fopen/fdopen | FILE\* | | **NULL** |
| fclose/fflush | 0 | | **EOF** |

### 데이터
- minute 0~1439, **시 = minute / 60**
- `?` = missing (평균 분모 제외), `0.0` = valid. 1.0, ?, 3.0 → **2.0**
- 유효값 0 → mean, peak 빈칸 (`02,0,2,,`)

### 안전한 저장
`open(O_WRONLY|O_CREAT|O_EXCL, 0644)` → `fdopen` → `fprintf` → **`fclose` 확인** → 실패면 `unlink`

| 상황 | 출력 파일 |
|---|---|
| 입력 없음 / 잘못된 입력 | 안 만듦 |
| 이미 있음 | **보존** |
| 쓰기/닫기 실패 | **삭제** |
| 출력 폴더 없음 | open 실패 (ENOENT) |
- `>`는 셸이 **먼저 비움** / 이름을 받으면 프로그램이 O_EXCL로 보호
- strace: stderr가 먼저 찍힘 = **stdout 버퍼링** 증거, getline이어도 실제는 `read(4096)`

### 과제
`[start, end)` → `if (minute/60 < start || minute/60 >= end) continue;` · `for (h = start; h < end; h++)` · 600·719분 포함, 599·720 제외

---

## 7. 10주차 — 프로세스

### fork / exec / wait
| | |
|---|---|
| `fork()` | 부모 = **자식 PID** / 자식 = **0** / 실패 = **-1** |
| `execvp(argv[0], argv)` | 성공 = 바뀜, **PID 유지**, 안 돌아옴. argv 끝 NULL |
| exec 실패 | `perror` + **`_exit(127)`** |
| `waitpid(pid, &st, 0)` | 기다림, EINTR이면 재시도 |
| `WIFEXITED → WEXITSTATUS` | 1792 → **7** |
| `WIFSIGNALED → WTERMSIG` | 시그널 번호 |
- fork 후 **메모리 따로** (자식 20, 부모 10 그대로)
- 실행 순서는 OS가 정함

### 종료 코드 ⚠️
| 상황 | 코드 |
|---|---|
| 자식 반환값 | 그대로 (7 → 7) |
| exec 실패 | **127** |
| fork/wait 실패 (runner) | **125** |
| 시그널 종료 (실습) | **128 + n** |

### 셸 실습
- start: fork → 자식 exec → 부모 `return pid` / **fork 없이 exec → 셸이 사라짐**
- wait: `WEXITSTATUS` (A `return 0` → 0, B `return status` → 1792, **C** → 7). 0만 테스트하면 버그 못 찾음
- cd: **부모에서 chdir** (자식에서 하면 부모 그대로) → 테스트에 **pwd**

---

## 8. 11주차 — 시그널

### 이름 ⚠️
| 시그널 | 발생 | 기본 |
|---|---|---|
| SIGINT (2) | **Ctrl+C** | 종료 |
| SIGTERM (15) | **kill** | 종료 |
| SIGTSTP | **Ctrl+Z** | 정지 |
| SIGCONT | bg/fg | 재개 |
| SIGKILL (9) | kill -9 | 강제 종료 |
| SIGALRM (14) | alarm | 종료 |
| SIGCHLD | 자식 상태 변화 | |
- **SIGKILL, SIGSTOP은 잡기·차단·무시 불가**

### 작업 제어
`jobs` 목록 · `bg` 뒤에서 재개 · `fg` 앞으로 · `cmd &` · `$!` 마지막 백그라운드 PID
- 포그라운드/백그라운드(입력) ≠ 실행/정지(상태) / Ctrl+C는 **포그라운드에만**
- bg로 재개하면 jobs에 `&`가 붙어 표시됨

### 핸들러
- `volatile sig_atomic_t` 플래그만, 처리는 main (printf·malloc 안전 X)
- `sigaction(SIG, &sa, NULL) == -1`이면 실패
- 플래그 복사·지우기 → `sigprocmask(SIG_BLOCK)` → 복원 (차단 = 미룸, 무시 = 버림)
- 같은 시그널 3번 → pending **1개** (개수 세기 X)

### 완료
- **kill = 요청, waitpid = 완료 증거**
- `waitpid(-1/child, &st, WNOHANG)` → PID 회수 / **0** 실행 중 / **-1 ECHILD** 자식 없음
- SIGTERM + 핸들러 → 정리 후 **exit 0** / SIGKILL → **signal 9**

### limit ⚠️
| 상황 | `$?` |
|---|---|
| 시간 초과 | **124** (자식 status는 143) |
| Ctrl+C / kill -INT 부모 | **130** |
| exec 실패 | 127 |
| grep 못 찾음 | **1** (그대로) |

---

## 9. 12주차 — 스레드

### 기본
- 스레드 = 프로세스 안 실행 흐름. **스택만 따로**, 코드·데이터·힙 공유
- `pthread_create(&tid, NULL, fn, &arg)` / `pthread_join(tid, NULL)` → **0 = 성공** (에러 번호, -1 아님)
- `gcc -pthread`

| | 프로세스 | 스레드 |
|---|---|---|
| 만들기 | fork | pthread_create |
| 기다리기 | waitpid | pthread_join |
| 메모리 | 따로 | **공유** |

### 규칙 ⚠️
- **join 후에** 결과 읽기 / join = 완료, 성공은 결과로
- join 순서 ≠ 끝나는 순서
- 생성 실패 → **성공한 것만 join** 후 return
- 인자는 join 전까지 살아 있어야 함

### 동기화
- lost update: A·B 둘 다 0 읽음 → 1+2가 **2**
- data race = UB, 한 번 맞아도 안전 증거 X, sleep은 해결책 X
- mutex: **읽기 + 더하기 + 쓰기** 전부 lock 안, 같은 mutex
- 좌석: 확인과 예약을 **같은 lock 안에서** (`if (seats > 0)`)
- 이체: lock 한 번에 빼고 더하기, 읽을 때 두 값 함께 복사

### 데드락
- lock 순서 엇갈림 → **같은 순서**
- mutex 쥔 채 join → **unlock 후 join**
- create 직후 join(반복문 안) → 정확하지만 **동시성 없음**

### 나누기
- 겹치지 않는 칸 → mutex 불필요, **join은 필요**
- `k*(N/W)` → 10,3이면 9번 누락 / `k*N/W` → [0,3) [3,6) [6,10)
- 직렬 4 + 병렬 6: 2개 7초(1.43배), 최대 **2.5배**

---

## 10. 시험장 체크리스트 ✅
- [ ] 단답 명령: **파일명, 대시, 경로** 다 썼나?
- [ ] 출력 문제: **모든 줄** 다 썼나?
- [ ] `$?` 앞에 다른 명령 끼어 있나?
- [ ] 경계값 (`-lt` vs `-le`, `>` vs `>=`) 숫자 넣어 확인
- [ ] status 코드 왼쪽/오른쪽 자리
- [ ] 오버플로 ±256 검산
- [ ] 반환값: read **0** = EOF / getline **-1** / pthread **0** = 성공 (에러 번호)
- [ ] 종료 코드: 127 exec 실패 / 124 시간 초과 / 130 SIGINT / 128+n
- [ ] fork 반환값 부모·자식 반대로 쓰지 않았나
- [ ] "왜?" 서술형: 무엇을 → 왜 → 그래서 (O_EXCL, unlink, builtin cd, join 후 읽기)

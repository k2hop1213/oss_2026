# OSS 실습 핵심 정리 (W02B ~ W06B + W07B 대비, W09B ~ W12B)

> 실습 슬라이드 기준. 각 실습마다 **무엇을 했나 → 체크포인트 정답 → 핵심 포인트**.
> ⚠️ = 오늘 틀렸던 것

---

## W02B — 셸 실습 (`oss-w02`)

**준비:** `bash setup_w02.sh` → `tar -xzf oss-w02.tar.gz` (⚠️ x 풀기 / z gzip / f 파일) → `cd oss-w02`
구조: `README.txt data/ docs/ logs/ src/ (src/old/) tests/`

### 영상 Practice Set (`~/oss/w02`)
```bash
ls *.c | wc -l                     # 3
grep -c ERROR app.log              # 2
grep ERROR app.log | wc -l         # 2 (같은 결과, 다른 방법)
grep ERROR app.log > errors.txt    # wc -l → 2
grep ERROR app.log >> errors.txt   # wc -l → 4 (> 덮어쓰기, >> 추가)
ls -lhS /var/log | head -2         # 가장 큰 파일 (보통 dpkg.log)
```

### Lab 1 — 탐색과 와일드카드
```bash
find . -type d                  # 트리 구조 파악
ls src/*.c                      # 3개 (한 폴더만)
find . -name "*.c" | wc -l      # 7개 (트리 전체)
cat src/old/legacy.c            # Checkpoint 1: FIXME 줄
```
- 3 vs 7 차이 = `src/old`의 옛 소스 2개 + `tests/`의 테스트 2개

### Lab 2 — 복사·이동·삭제
```bash
cp -r logs logs_backup                           # 폴더는 -r
mv logs_backup/old.log logs_backup/archive.log   # 이름 변경 = 이동
ls tmp*.txt                                      # 지우기 전에 같은 패턴으로 확인
rm tmp*.txt
rm -r logs_backup draft.txt
```
- Task: `data/`로 같은 과정 (백업 → 백업 안에서 이름 변경 → 백업 삭제)

### Lab 3 — grep과 리다이렉션 (`logs/`)
```bash
head -3 app.log                      # 측정 전에 데이터 보기
grep -c ERROR app.log > count.txt    # Checkpoint 2: cat → 12
grep -in error app.log | grep -v ERROR
```
- `-c ERROR` 12 vs `-ci error` **14** → 남는 2줄은 "0 error rows", "no error detected" (문제없다는 INFO)
- grep은 **의미가 아니라 글자**

### Lab 4 — 파이프
```bash
grep ERROR app.log | cut -d" " -f4 | sort | uniq -c | sort -rn
# 5 db / 3 net / 2 auth / 1 ui / 1 sys   ← Checkpoint 3
grep -c ERROR *.log                  # app.log:12 / old.log:3
```
- sort 없이 `uniq -c` → 인접한 것만 합쳐져 개수가 쪼개짐

### Lab 5 — find와 sort
```bash
grep -rn FIXME src                       # 알려진 문제 위치
find . -size +1M                         # ./data/big.bin
find . -name "*.md"                      # 3개
sort -n data/scores.txt | tail -3        # 87 kim / 100 choi / 100 lim
sort data/fruits.txt | uniq -c | sort -rn            # apple 5
cut -d" " -f4 logs/app.log | sort | uniq -c | sort -rn   # db 51 (전체 200줄이라 grep 없이)
```

### Lab 6 — 링크 ⚠️
```bash
echo "the original line" > original.txt
ln original.txt hard.txt          # ls -li: 같은 inode, 링크 수 2
ln -s original.txt soft.txt       # soft.txt -> original.txt
rm original.txt
cat hard.txt                      # the original line (살아 있음)
cat soft.txt                      # No such file or directory
echo "restored" > original.txt
cat soft.txt                      # restored (경로만 저장하니까 재연결)
```

### Lab 7 — 파일을 명령으로 ⚠️
```bash
echo "echo hello from my first command" > hello.sh
bash hello.sh      # OK
./hello.sh         # Permission denied
chmod +x hello.sh  # -rwxr-xr-x
./hello.sh         # OK → Checkpoint 4
hello.sh           # command not found (PATH에 . 없음)
```

### Lab 8 — nano
- `nano hello.sh` → Ctrl+O 저장(Enter) → Ctrl+X 종료 (`^` = Ctrl)

### Extra / Challenge 정답
| 문제 | 답 |
|---|---|
| old.log WARN | `grep -c WARN logs/old.log` → 5 |
| 두 로그 ERROR 합계 | `cat logs/*.log \| grep -c ERROR` → 15 (`grep -c`만 쓰면 파일별) |
| docs TODO | `grep -r TODO docs \| wc -l` → 4 |
| INFO·WARN 아닌 줄 | `grep -v INFO logs/app.log \| grep -v WARN` |
| `sort scores.txt` 예측 | 100 choi, 100 lim, 21 shin... (글자순) |
| 시간대별 | `cut -d" " -f2 logs/app.log \| cut -d: -f1 \| sort \| uniq -c` |
| db ERROR만 | `grep ERROR logs/app.log \| grep " db " > db_errors.txt` → 5 |
| 레벨 분포 | `cat logs/*.log \| cut -d" " -f3 \| sort \| uniq -c \| sort -rn` |
| 가장 긴 C 파일 | `wc -l src/*.c src/old/*.c tests/*.c \| sort -n \| tail -2` |

---

## W03B — Git 실습 (`oss-w03`)

**준비:**
```bash
bash setup_w03.sh && cd oss-w03
git init -b main
git config user.name "이름"; git config user.email "메일"
git add README.md commands.md .sample logs/git-practice.log
git commit -m "Add practice sample"
```
파일: README.md(틀린 설명 1줄), commands.md, .sample(숨김), logs/git-practice.log(ERROR 3줄), reports/

### 예측 연습
```bash
echo "A" > note.txt; git add note.txt; echo "B" >> note.txt
```
- 다음 커밋 = **A만** / `git diff`에 보이는 것 = **B 추가** / add는 A 시점을 보존

### Lab 1 — 한 번에 하나씩 커밋
```bash
grep -c ERROR logs/git-practice.log > reports/count.txt   # 3
nano README.md                                           # ls vs ls -a 설명 수정
git add README.md && git commit -m "Correct hidden file explanation"
git add reports/count.txt && git commit -m "Record ERROR count"
git log -2 --oneline --stat
```
- **Checkpoint 1:** count.txt = 3 / 두 커밋에 파일이 따로 / **untracked 파일은 `git diff`에 안 나옴** (diff는 추적 중인 파일의 작업↔스테이징 비교)
- `git add .` 대신 파일 경로 명시

### Lab 2 — add 후 수정 (`MM`)
```bash
echo "pwd: print the working directory" >> commands.md
git add commands.md
echo "ls: list directory entries" >> commands.md
git status --short     # MM commands.md
git diff --staged      # + pwd
git diff               # + ls
git commit -m "Document pwd"      # 다시 add 안 함 → pwd만
git show HEAD:commands.md
git add commands.md && git commit -m "Document ls"
```
- 세 스냅샷: HEAD = 제목만 / 스테이징 = 제목+pwd / 작업 = 제목+pwd+ls
- **Checkpoint 2:** 첫 커밋 pwd, 둘째 ls, 최종 clean

### restore 두 종류
| 명령 | 변화 |
|---|---|
| `restore --staged f` | 스테이징 ← HEAD, 작업 파일 유지 |
| `restore f` | 작업 파일 ← 스테이징, 수정 덮어씀 |
| `show HEAD:f` | 읽기만, 파일 변화 없음 |

### Lab 3 — 좋은 수정은 남기고 실수만 복구
```bash
echo "Verified in Ubuntu." >> README.md
echo "WRONG" > commands.md          # 실수
git add README.md commands.md
git restore --staged commands.md    # ① 스테이징만 취소 (파일은 아직 WRONG)
git restore commands.md             # ② 작업 파일을 스테이징 버전으로
git commit -m "Record Ubuntu verification"
```
- **Checkpoint 3:** `restore`만 하면 왜 안 되나? → 스테이징에 이미 WRONG이 있어서 **WRONG으로 되돌아감**

### Lab 4 — 생성 파일 무시 ⚠️
```bash
echo "debug run 1" > reports/debug.log
git add reports/debug.log && git commit -m "Record debug sample"
echo "*.tmp" > .gitignore
echo "reports/*.log" >> .gitignore
echo "scratch" > scratch.tmp
echo "debug run 2" >> reports/debug.log
git status --short      # scratch.tmp는 안 보임,  M reports/debug.log는 계속 보임
git rm --cached reports/debug.log
git add .gitignore
git commit -m "Stop tracking generated files"
```
- **Checkpoint 4:** debug.log는 디스크에 남고 추적 끝 / 둘 다 규칙에 매칭 / `git show HEAD~1:reports/debug.log` → **옛 커밋엔 남아 있음**
- 핵심: **ignore 규칙은 추적 중인 파일을 untrack하지 않음**

### Classroom 제출
```bash
gh auth status; gh auth setup-git
cd ..                                            # oss-w03 밖에서
git clone 과제URL git-w03-submit && cd git-w03-submit
git remote -v
echo "pwd: print the working directory" >> README.md
git add README.md; git diff --staged
git commit -m "Test passing submission"
git push
git rev-parse --short HEAD                       # Actions·Classroom의 커밋 ID와 일치 확인
```
- Actions 통과 → Classroom 점수 1/1

### Individual Check
- `final-check.md` 작성 → "clean ≠ push" 설명 + 명령 하나와 출력 → **그 파일만** 커밋 + diff 설명
- ⚠️ clean = 로컬 상태일 뿐, push는 별개

### 도전 문제
```bash
git log --oneline -- commands.md       # pwd만 있는 커밋 찾기
git show 커밋:commands.md              # 파일 안 바꾸고 읽기
git diff 커밋 HEAD -- commands.md      # 다음 커밋과 비교
git mv commands.md command-guide.md    # ⚠️ R, 커밋 후 원래 이름으로 다시
rm commands.md                         #  D → git restore commands.md (커밋된 버전 덕에 복구)
```

### 명령이 실패할 때 (부록)
| 에러 | 확인 |
|---|---|
| Author identity unknown | `git config user.name/email` |
| Repository not found | 계정, 초대, URL |
| Authentication failed | `gh auth status` |
| Push rejected | 먼저 확인, **force-push 금지** |
도움 요청 전: `pwd`, `git status -sb`, `git remote -v`, `git log -3 --oneline`, `gh auth status`

---

## W04B — 셸 스크립트 실습 (`oss-w04`)

**목표 계약:** 입력 = 읽을 수 있는 로그 파일 1개 / 출력 = stdout에 ERROR 개수 하나 / 실패 = stderr + 0 아닌 상태

**데이터**
| 파일 | 기대 개수 |
|---|---|
| events.log (8줄) | 3 |
| clean.log | 0 |
| empty.log | 0 |
| day one.log (공백!) | 1 |
- `grep -ci error events.log` = 4 (대소문자 무시하면 하나 더)

### count_errors.sh (완성본)
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
status=$?                        # ⚠️ 바로 저장
if [ "$status" -gt 1 ]; then     # ⚠️ 1(못 찾음)은 정상
  echo "Error: search failed" >&2
  exit 1
fi
echo "$count"
exit 0
```
| 입력 | stdout | status |
|---|---|---|
| events | 3 | 0 |
| clean / empty | 0 | 0 |
| day one | 1 | 0 |
| missing | (없음) | 1 |

- 실패 예: `count=$(...)` → `echo "search completed"` → `status=$?` → **echo의 상태가 저장됨**
- 입력을 바꾸면 결과도 바뀌어야 함: `changed.log`에 ERROR 추가 → 4

### summarize.sh
```bash
#!/bin/bash
if [ "$#" -lt 1 ]; then echo "Usage: bash summarize.sh FILE..." >&2; exit 1; fi
for file in "$@"; do                       # 1) 전부 먼저 검사
  if [ ! -f "$file" ] || [ ! -r "$file" ]; then
    echo "Error: readable file required: $file" >&2; exit 1
  fi
done
for file in "$@"; do                       # 2) 처리
  count=$(bash ./count_errors.sh "$file")
  status=$?
  if [ "$status" -ne 0 ]; then exit 1; fi
  echo "$file: $count"
done
exit 0
```
- ⚠️ `> reports/summary.txt`는 **스크립트 시작 전에** 파일을 비움 → 안전 패턴:
```bash
if bash summarize.sh data/events.log > reports/tmp.txt; then
  mv reports/tmp.txt reports/summary.txt
fi
```

### verify_counts.sh
```bash
failures=0
check_count() {
  bash ./count_errors.sh "$1" > reports/check.out 2> reports/check.err
  actual_status=$?
  actual_output=$(cat reports/check.out)
  if [ "$actual_status" -eq "$3" ] && [ "$actual_output" = "$2" ]; then
    echo "PASS $1"
  else
    echo "FAIL $1: expected output=<$2> status=$3; actual output=<$actual_output> status=$actual_status"
    failures=1                 # 한 번 1이면 끝까지 1
  fi
}
check_count data/events.log 3 0
check_count data/clean.log 0 0
check_count data/empty.log 0 0
check_count 'data/day one.log' 1 0
check_count data/missing.log '' 1
exit "$failures"
```
- 상태는 **숫자**(`-eq`), 출력은 **문자열**(`=`): `03`, ` 3`은 `3`과 다름
- stderr 문구는 검사 안 함 (검증의 한계)

### 통제된 결함 ⚠️
| 결함 | FAIL | 이유 |
|---|---|---|
| ① `-gt 1` → `-ge 1` | **clean, empty** | 정상인 상태 1을 실패로 |
| ② 성공 시 stdout에 echo 추가 | **정상 4개 전부** | 출력이 두 줄 (상태는 0) |
| ③ `"$1"` → `$1` | **day one만** | 공백에서 쪼개짐 |
- 패턴 → 의심 영역 → `bash count_errors.sh "data/day one.log" > one.out 2> one.err`로 직접 재현
- 매번 `cp count_errors.sh.correct count_errors.sh`로 복구 후 다음 결함

### Git
```bash
git init
printf 'reports/\n' > .gitignore          # 생성 결과 무시
bash verify_counts.sh                     # 테스트 먼저
git add count_errors.sh summarize.sh verify_counts.sh README.md .gitignore
git diff --staged
git commit -m "Add reusable log summary tools"
```

---

## W05B — C 빌드 실습 (`oss-w05`)

**준비:** `command -v gcc ar nm git curl` → `bash setup_w05.sh` → `cd oss-w05`
**계약:** `ERROR `(공백 포함)로 **시작하는** 줄 개수 → data/app.log = **2**

| 입력 | stdout | status |
|---|---|---|
| 매칭 2줄 | 2 | 0 |
| 매칭 없음 | 0 | **0** |
| 인자 없음 | (없음) | **2** |
| 파일 없음 | (없음) | **3** |

### 기준 빌드
```bash
mkdir -p build
gcc -std=c11 -Wall -Wextra main.c logutil.c -o build/logcount
./build/logcount data/app.log ; echo $?     # 2 / 0
bash check.sh ./build/logcount              # 7 passed, 0 failed
```

### Exercise 1 — 분리 컴파일
```bash
gcc -std=c11 -Wall -Wextra -I. -c main.c -o build/main.o
gcc -std=c11 -Wall -Wextra -I. -c logutil.c -o build/logutil.o
gcc build/main.o build/logutil.o -o build/logcount
```

### Exercise 2 — 빠진 오브젝트
```bash
gcc build/main.o -o build/missing     # undefined reference to `is_error' → 링크 단계
```
- 헤더(`#include "logutil.h"`)는 **선언만** → 몸통 없음 → logutil.o 추가로 해결 (소스 수정 X)
- `nm build/main.o` → `U is_error` / `nm build/logutil.o` → `T is_error`

### Exercise 3 — 문법 오류
```bash
gcc ... -c variants/logutil_syntax.c -o build/bad.o   # expected ';' → 컴파일 단계
```
- 진단 = `파일:줄:칸: error: 내용` / 보고된 줄과 **앞 문장**을 봄

### Exercise 4 — 소스 변경 ⚠️
```bash
cp variants/logutil_contains.c logutil.c
./build/logcount data/app.log     # 아직 2 (옛 실행 파일)
gcc ... -c logutil.c -o build/logutil.o
./build/logcount data/app.log     # 아직 2 (재링크 안 함)
gcc build/main.o build/logutil.o -o build/logcount
./build/logcount data/app.log     # 3
```

### Exercise 5 — 동작 버그
- contains(`strstr`) vs starts with(`strncmp`) → `INFO no ERROR detected` 때문에 3
- 컴파일은 **문법만** 봐서 성공 / 경계값 예: 18세 이상 `>=` vs `>` → age=18

### Exercise 6 — 정적 라이브러리
```bash
ar rcs build/liblogutil.a build/logutil.o
ar t build/liblogutil.a                       # logutil.o
gcc build/main.o build/liblogutil.a -o build/logcount_static
bash check.sh ./build/logcount_static
```

### Exercise 7 — 헤더 변경 ⚠️
- `#define ERROR_PREFIX "WARN "` → **main.o와 logutil.o 둘 다** 재컴파일 → app.log 결과 **1** (`WARN memory`)

### 복구
```bash
cp baseline/logutil.h logutil.h; cp baseline/logutil.c logutil.c
gcc -std=c11 -Wall -Wextra main.c logutil.c -o build/logcount
bash check.sh ./build/logcount
```

### Challenge — 디버그 출력
- `variants/logutil_debug.c` → 진단 메시지를 stdout에 찍어서 check 실패 → **진단은 stderr로**

### 확장 — 공유 라이브러리 ⚠️
```bash
gcc -fPIC -I. -c logutil.c -o build/logutil_pic.o
gcc -shared build/logutil_pic.o -o build/liblogutil.so
gcc -I. -c main.c -o build/main.o
gcc build/main.o -Lbuild -llogutil -o build/logcount_shared
env -u LD_LIBRARY_PATH ./build/logcount_shared data/app.log   # 실패 (못 찾음)
LD_LIBRARY_PATH="$PWD/build" bash check.sh ./build/logcount_shared
```

### Individual Check
1. 헤더만으론 왜 링크 안 되나 → 선언뿐, 정의(몸통)가 든 `.o` 필요
2. 소스 변경 후 → 재컴파일 → 재링크 → 실행
3. contains 버그를 드러내는 입력 → `INFO no ERROR detected`
4. 버전 관리 대상 → 소스·헤더·check.sh·README·.gitignore / `build/`는 제외

### Git
```bash
git add main.c logutil.c logutil.h check.sh README.md .gitignore
git add -f data/*.log          # 무시 규칙을 넘어 강제로 추가
git commit -m "Record W05 baseline"
```

---

## W06B — Make & GDB 실습 (`oss-w06-region`)

**준비:** `bash setup_w06.sh oss-w06-region` → `cd oss-w06-region`
```bash
make region region-check          # 12 PASS
./bin/region-lab assets/sample.jpg output/region.png 400 200 400 250 80
#                                                    x   y   w   h  gain
# Processed 100000 pixels  (= 400 × 250)
```

### 함수 계약
```c
typedef struct { int x, y, width, height; } Region;
int region_valid(int width, int height, Region r);
size_t brighten_region(unsigned char *rgb, int width, int height, Region r, int gain);
```
- 유효: 크기 양수 + 모든 픽셀이 이미지 안
- 영역 안 RGB만 수정, **처리한 픽셀 수** 리턴 / 잘못된 입력 → 0 리턴, 버퍼 그대로

### 좌표 → index ⚠️
```
          x=0  x=1  x=2  x=3      (4×3 이미지)
y=0        0    3    6    9
y=1       12   15   18   21
y=2       24   27   30   33
```
- `index = ((size_t)y * width + x) * 3` → (1,1) = (1×4+1)×3 = **15**
- 영역 (1,1,2,1) = 픽셀 (1,1), (2,1)

### 입력 검사
```c
return width > 0 && height > 0 && r.x >= 0 && r.y >= 0
    && r.x < width && r.y < height && r.width > 0 && r.height > 0
    && r.width <= width - r.x && r.height <= height - r.y;
```
- 너비 0, 음수 좌표, 밖으로 넘치는 사각형은 실패

### 구현
```c
if (!rgb || !region_valid(width, height, r)) return 0;
size_t pixels = 0;
for (int y = r.y; y < r.y + r.height; ++y)
    for (int x = r.x; x < r.x + r.width; ++x) {
        size_t index = ((size_t)y * width + x) * 3;
        for (int c = 0; c < 3; ++c)
            rgb[index + c] = brighten_channel(rgb[index + c], gain);
        ++pixels;
    }
return pixels;
```
- 저장 → `make region-check` → 12 PASS

### Makefile 포인트
```make
build/region.o: src/region.c include/region.h include/filters.h
	mkdir -p build
	$(CC) $(CPPFLAGS) $(CFLAGS) -c $< -o $@        # $< = src/region.c, $@ = build/region.o

bin/region-lab: build/region_main.o build/region.o build/image_io.o lib/libfilters.a
	mkdir -p bin
	$(CC) $(CFLAGS) -o $@ $^ $(LDLIBS)             # $^ = 재료 전부, LDLIBS = -lm
```
- `make -n region` = 실행 없이 미리 보기 / 안 바뀐 모듈은 `.o` 재사용

### stride 버그 조사
```bash
cp variants/region_stride.c src/region.c
make region && make region-check     # 정사각형만 통과, 위치가 틀림
```
- 실행은 성공하지만 **엉뚱한 위치**가 밝아짐 → 밝기 말고 **위치**를 비교

### GDB 재현 입력 (`tests/debug_region.c`)
- 4×3, 값 전부 10, 영역 `{1,1,2,1}`, gain 40 → 기대: rgb[15] = 50, rgb[12] = 10 유지
```
make bin/debug_region
gdb -q -tui ./bin/debug_region      (이미 열려 있으면 layout src, 화면 깨지면 Ctrl+L)
(gdb) break brighten_region
(gdb) run
(gdb) display width / display height / display r
(gdb) next ... (index 줄까지)        → x=1, y=1
(gdb) next                           → index = 12 (기대 15)
(gdb) step                           → brighten_channel 안으로
(gdb) finish                         → 50 리턴
(gdb) print rgb[12] / rgb[15]        → 50 / 10  (엉뚱한 칸이 바뀜)
(gdb) quit
```
| GDB | VS Code |
|---|---|
| next | F10 |
| step | F11 |
| finish | Shift+F11 |
| continue | F5 |
| print / display | Variables / Watch |

### 수정과 회귀 확인
- `height` → `width`로 수정 → GDB 끄고 → `make region-check` → `make region` → 같은 사각형 다시 생성
- 구별 입력: 정사각형·가로형·세로형 / 마지막 픽셀 / 잘못된 사각형

### 디버깅 증거
1. 바뀐 함수와 인터페이스
2. 소스 변경 후 실행한 빌드 명령
3. 멈춘 줄, 관찰한 index(12), 기대 index(15)
4. 고친 사각형 결과 + 회귀 테스트

---

## 내일 W07B 대비 (W07A 내용)

### 테스트의 원리
- 정한 입력 → **요구사항에서 나온** 기대값과 실제값 비교
- 단위 테스트: `assert(조건)` → 거짓이면 즉시 중단 / 테스트는 **자기 main을 가진 별도 실행 파일**
- CLI 테스트: stdout·stderr·**상태**·생성 파일을 각각 확인

### 프로그램 exit vs 테스트 exit ⚠️
- 잘못된 입력에 프로그램이 **exit 2 = 정답**일 수 있음 → 그걸 확인한 테스트는 PASS(0)
- 테스트가 실패하면 **반드시 `exit 1`** → 그래야 Make·CI가 알아챔

### CLI 테스트 틀
```bash
work=$(mktemp -d) || exit 2
trap 'rm -rf -- "$work"' EXIT
./bin/image-lab assets/sample.jpg "$work/result.png" sepia >"$work/stdout" 2>"$work/stderr"
status=$?
[[ $status -eq 2 ]] || exit 1
[[ ! -e "$work/result.png" ]] || exit 1
grep -Fxq "unknown filter" "$work/stderr" || exit 1
printf "PASS unsupported mode\n"
```

### batch.sh 버그 ⚠️
- 증상: `정상 / broken / 정상` → broken에 거짓 `DONE`, 최종 상태 0
- 틀린 수정: 실패 즉시 `exit 1`(뒤를 처리 안 함) / 성공마다 `failed=0`(앞의 실패가 지워짐)
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
- W04B `failures=1`과 같은 원리
- **회귀 테스트**: 버그를 드러낸 입력을 계속 남김 / **기대값을 바꿔서 통과시키지 말고 코드를 고침**

### make check & CI
```make
check: unit-test batch-test      # 테스트 실패 → make: *** [batch-test] Error 1
```
```yaml
# .github/workflows/check.yml   ← YAML은 스페이스 (Makefile은 Tab)
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
- push마다 GitHub가 `make check` 실행 → **Actions 탭**에서 커밋별 ✅/❌

---

## W09B — 파일 입출력 실습 (`oss-w09`)

**준비:**
```bash
base=https://gist.githubusercontent.com/Kyungho-Won
id=f1b4aec11633fcb4a74d0d99b705a08e
curl -L "$base/$id/raw/setup_w09.sh" -o setup_w09.sh && bash setup_w09.sh && cd oss-w09
make && make check-base          # 3개 통과, power-report는 "Function not implemented"
```
- 셋업은 기존 `oss-w09`를 덮어쓰지 않음 → 이름 겹치면 새 폴더에서

### load.c (`load_hours`)
```c
char *line = NULL; size_t capacity = 0; int rc = -1, previous = -1;
if (getline(&line, &capacity, in) < 0) goto done;
line[strcspn(line, "\r\n")] = '\0';
if (strcmp(line, "minute,power_kw") != 0) goto done;
while (getline(&line, &capacity, in) >= 0) {
    int minute; double power;
    int kind = parse_row(line, &minute, &power);     // 1 측정 / 0 ? / -1 잘못됨
    if (kind < 0 || minute <= previous) goto done;
    previous = minute;
    Hour *s = &hours[minute / 60];
    if (kind == 0) { s->missing++; continue; }
    if (s->valid == 0 || power > s->peak) s->peak = power;
    s->valid++; s->sum += power;
}
if (ferror(in)) { perror("read input"); goto done; }
rc = 0;
done: free(line); return rc;
```
- 확인: `./bin/power-report data/power_20061221.csv | head -14` → `11,58,2,...`

### save.c (`save_report`)
```c
int fd = open(path, O_WRONLY | O_CREAT | O_EXCL, 0644);
if (fd < 0) return -1;
FILE *out = fdopen(fd, "w");
if (!out) { int saved = errno; close(fd); unlink(path); errno = saved; return -1; }
int rc = emit_csv(out, hours);
int saved = rc < 0 ? errno : 0;
if (fclose(out) == EOF) { if (rc == 0) saved = errno; rc = -1; }
if (rc < 0) { unlink(path); errno = saved; }
return rc;
```

### 체크포인트 ⚠️
```bash
mkdir -p output        # ⚠️ 없으면 "write report: No such file or directory"
./bin/power-report data/power_20061221.csv output/hourly.csv; echo $?   # 0
cp output/hourly.csv output/before.csv
./bin/power-report data/power_20061221.csv output/hourly.csv; echo $?   # 1 (File exists)
cmp output/before.csv output/hourly.csv && echo same                    # same
printf 'minute,power_kw\n0,NaN\n' > output/bad.csv
./bin/power-report output/bad.csv output/bad-report.csv; echo $?        # 1
test ! -e output/bad-report.csv && echo "no file"
make check                                                               # 19/19
```
- viewer.html: 탐색기 `\\wsl$\Ubuntu\home\사용자\...\oss-w09` → 크롬으로 열고 output/hourly.csv 선택

### 추가 활동: check_extra.sh
```bash
curl -L "$base/$id/raw/setup_w09_activities.sh" -o setup_w09_activities.sh && bash setup_w09_activities.sh
```
| 경우 | 예측 |
|---|---|
| valid → 새 경로 | 0, 생성 |
| invalid → 새 경로 | 1, 없음 |
| valid → 이미 있는 경로 | 1, 그대로 |

```bash
#!/usr/bin/env bash
set -eu
work=$(mktemp -d); trap 'rm -rf "$work"' EXIT
./bin/power-report activities/missing.csv "$work/missing.csv"
grep -qx '02,0,2,,' "$work/missing.csv"
./bin/power-report activities/valid.csv "$work/report.csv"
cp "$work/report.csv" "$work/before.csv"
if ./bin/power-report activities/valid.csv "$work/report.csv"; then echo FAIL >&2; exit 1; fi
cmp -s "$work/before.csv" "$work/report.csv"
if ./bin/power-report activities/invalid.csv "$work/bad.csv"; then echo FAIL >&2; exit 1; fi
test ! -e "$work/bad.csv"
echo "PASS extra checks"
```
- `make check` + `bash tests/check_extra.sh; echo $?` → 둘 다 통과, 0
- 버그 넣기 확인: `O_EXCL` → `O_TRUNC`로 바꾸면 FAIL이 나야 함

### W09 과제 (시간 범위)
```c
if (minute / 60 < start || minute / 60 >= end) continue;   // load.c L1, previous = minute 다음
for (int h = start; h < end; h++) {                         // report.c R1
```
- `make check` 19/19, `make check-range` 3/10 → 10/10
- 고치기 전 `git commit --allow-empty -m "Check starter"` → push → 실패 캡처 / 수정 후 성공 캡처 → PPT 2장

---

## W10B — 셸 실습 (`oss-w10-submission`)

**준비:** Classroom50 W10 Command Shell → clone → `make demo` → `./bin/shell-demo`
```
oss-shell> wc -l data/sample.csv      → 6 data/sample.csv [exit=0]
oss-shell> ./missing-command          → [exit=127]
oss-shell> quit
```

### 활동 1 — start.c
```c
pid_t pid = fork();
if (pid < 0) { perror("fork"); return -1; }
if (pid == 0) { execvp(argv[0], argv); perror(argv[0]); _exit(127); }
return pid;
```
- `make check-start`: 인자 전달, 127, **다음 명령(NEXT)도 받음**
- ⚠️ fork 없이 exec → 출력은 같지만 셸이 wc로 바뀌어 사라짐 → NEXT 못 받음

### 활동 2 — wait.c
```c
if (WIFEXITED(status)) return WEXITSTATUS(status);   // 시그널 분기 앞에
```
| probe 7 (1792) | 결과 |
|---|---|
| `return 0;` | [exit=0] |
| `return status;` | [exit=1792] |
| `WEXITSTATUS` | **[exit=7]** |
- ⚠️ 0으로 끝나는 명령만 테스트하면 셋 다 0 → 못 찾음

### 활동 3 — builtin.c
```c
if (chdir(argv[1]) != 0) perror("cd");
return 1;
```
- ⚠️ 검사 강화: `cd data → cd .. → wc` 는 셸 chdir / 자식 chdir 둘 다 6 → **cd data 다음 pwd** 넣으면 구분
- 실패 cd: `cd data → cd ./missing-dir → pwd` → 여전히 .../data

### 실패 원인
| 명령 | exit | 보고자 |
|---|---|---|
| `grep ERROR events.log` | 0 | |
| `grep WARNING events.log` | 1 | grep |
| `grep ERROR missing.log` | 2 | **grep** (exec 성공) |
| `./missing-command` | 127 | **exec 실패** |
| `/bin/echo NEXT` | 0 | 셸 살아 있음 |

### 제출
```bash
make check                                     # 9개
git add src && git commit -m "Implement command shell" && git push
```

---

## W11B — 시그널 실습 (`oss-w11-signals`)

**준비:** Classroom50 W11 Signal Handling → clone → `make demo` → `mkdir -p out; cp data/events.log out/live.log`

### 터미널 관찰 ⚠️
```bash
ping 127.0.0.1                      # Ctrl+C → 통계 출력 후 종료
tail -n 1 -f out/live.log           # Ctrl+Z → [1]+ Stopped
printf 'ERROR request=/new status=500\n' >> out/live.log
bg                                  # 재개 → 이제 ERROR 줄 보임 (jobs에 & 표시)
fg                                  # 앞으로 → Ctrl+C
timeout -v 2s tail -n 1 -f data/events.log; echo $?     # 124
```
- tail을 여러 번 실행하고 Ctrl+Z하면 jobs에 쌓임 → `kill %1 %2 ...`로 정리

### limit 데모
```
./bin/limit-demo 2 tail -n 1 -f data/events.log
START parent=.. child=.. limit=2s
REQUEST child=.. signal=SIGTERM
COLLECTED child=.. status=143
$? = 124
```

### 활동 1 — control.c
```c
static void on_signal(int signo) { requested_signal = signo; }
/* install_control */
if (sigaction(SIGALRM, &sa, NULL) == -1) return -1;
if (sigaction(SIGINT,  &sa, NULL) == -1) return -1;
```
- `make check1` (deadline 14 → 124, interrupt 2 → 130)

### 활동 2 — monitor.c
```c
pid_t result = waitpid(child, status, WNOHANG);
if (result == child) return 0;
if (result == -1 && errno != EINTR) return -1;
if (requested_signal && !sent) {
    if (kill(child, SIGTERM) == -1 && errno != ESRCH) return -1;
    sent = 1;
}
```
- `make check2` → `make check` 6/6

### 활동 3 — 예측 ⚠️
| 명령 | TERM | `$?` |
|---|---|---|
| `5 true` | 없음 | 0 |
| `5 grep ERROR` | 없음 | 0 |
| `5 grep FATAL` | 없음 | **1** |
| `2 tail -f` | 있음 | **124** |
| `30 tail -f &` → `kill -INT "$parent"` → `wait` | 부모가 보냄 | **130** |
- 리뷰: `kill(child, SIGTERM); return 0;` → kill 검사 없음 + **waitpid 회수 전 return** → status 미설정, 좀비

### 제출
```bash
git add src/control.c src/monitor.c && git commit -m "Implement signal handling" && git push   # 6/6
```

---

## W12B — 스레드 실습 (`oss-w12-threads`)

**준비:** Classroom50 W12 Threads 실습 → clone → `make examples && make demo`

| 타깃 | 내 파일 |
|---|---|
| `make check1` | launch.c |
| `make check2` | merge.c |
| `make check` | 둘 다 (7/7) |

### 활동 1 — launch.c
```c
for (size_t i = 0; i < count; i++) {
    rc = pthread_create(&tids[i], NULL, scan_file, &jobs[i]);
    if (rc != 0) break;
    started++;
}
for (size_t i = 0; i < started; i++)
    require_ok(pthread_join(tids[i], NULL), "pthread_join");
return rc;
```
- 3개 중 2개 성공 → tids[0], [1]만 유효, join 2번, join 후 return, 전부 성공 0
- 결과 `TOTAL lines=7 errors=3 failed=0`

### 활동 2 — merge.c
```c
require_ok(pthread_mutex_lock(&summary->mutex), "lock");
summary->counts.lines  += part->lines;
summary->counts.errors += part->errors;
summary->counts.failed += part->failed;
require_ok(pthread_mutex_unlock(&summary->mutex), "unlock");
```
- `/tmp/observe lost` → 1 (기대 2) / `mutex` → 2 / `-fsanitize=thread` → data race 경고

### 활동 3 — seat.c ⚠️
```c
if (available) { --seats; booked[id] = 1; }   // ✗ 옛 확인 결과 → 1석에 2명
if (seats > 0) { --seats; booked[id] = 1; }   // ✓ 예약 순간 재확인
```
- 좌석 0/1/2 → confirmed 0/1/2, remaining 0. analysis.md에 실패 순서 기록

### 활동 4 — 코드 리뷰 (analysis.md)
| 코드 | 판단 | 수정 |
|---|---|---|
| 반복문 안 create 직후 join | 정확, **동시성 없음** | create/join 반복문 분리 |
| mutex 쥔 채 run_jobs | **데드락** (main은 join, worker는 mutex 대기) | mutex 없이 호출 |

### 활동 5 — strips.c ⚠️
```c
s[k] = (struct Strip){ k*N/W, (k+1)*N/W };    // 원래 k*(N/W) → 10,3이면 9번 누락
```
- (12,3) [0,4)[4,8)[8,12) / (10,3) [0,3)[3,6)[6,10) / (2,3) [0,0)[0,1)[1,2) 빈 띠 OK
- 겹치지 않는 칸 → mutex 불필요, join은 필요

### 제출
```bash
cd "$(git rev-parse --show-toplevel)"
make check && git diff
git add src/launch.c src/merge.c analysis.md activities/seat/seat.c activities/image/strips.c
git commit -m "Implement thread activities" && git push
git rev-parse HEAD       # Actions SHA와 비교, 7/7
```

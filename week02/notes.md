# Week 02-A. Shell and Filesystem

## 1. 실습 환경

두 플랫폼 모두 **Ubuntu 26.04** 유저랜드를 사용합니다.

| 항목 | Windows | macOS |
|---|---|---|
| Linux 실행 | WSL2 | Docker 컨테이너 |
| 계정 | 일반 사용자 | root |
| 프롬프트 끝 | `$` | `#` |
| 패키지 설치 | `sudo apt install` | `apt install` |

- 설치된 도구: git, gcc, make, gdb, curl
- 한글 입력: `~/.bashrc`에 `export LANG=C.UTF-8` 설정 필요

---

## 2. 셸(Shell)과 명령어 구조

### 터미널 vs 셸 vs 프롬프트
- **터미널**: 화면을 보여주는 창
- **셸**: 명령을 해석·실행하는 프로그램 (bash가 기본)
- **프롬프트**: `user@host:cwd$` 형태

### 명령어 구조
```
명령어이름 옵션 인자
```
- 짧은 옵션: `-l -a -h` = `-lah` (조합 가능)
- 긴 옵션: `--all --human-readable` (조합 불가)
- 공백이 의미 있으므로 공백 포함 파일명은 따옴표/이스케이프 필요

### `.bashrc`
터미널을 열 때마다 **자동 실행되는 설정 파일** (`~/.bashrc`)
- `export`, `alias` 등을 적어두면 매번 자동 적용
- 이름이 `.`으로 시작 → 숨김 파일 → `ls -a`로만 보임
- 수정 후 즉시 반영: `source ~/.bashrc`

```bash
# 자주 넣는 내용 예시
export LANG=C.UTF-8
alias ll='ls -lh'
export PATH=$PATH:/some/custom/path
```

### `$PATH`
셸이 명령어를 찾을 때 **뒤지는 디렉토리 목록** (콜론으로 구분)
```bash
echo $PATH
# /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```
- `ls`라고만 쳐도 실행되는 이유는, 셸이 PATH 목록을 순서대로 뒤져서 찾아주기 때문
- 현재 디렉토리는 보통 PATH에 없어서 `./파일명`처럼 직접 경로를 줘야 실행됨

---

## 3. 파일시스템 계층 구조 (FHS)

| 경로 | 내용 |
|---|---|
| `/` | 전체 트리의 루트 |
| `/home`, `/root` | 사용자 홈 |
| `/etc` | 설정 파일 |
| `/usr/bin`, `/usr/sbin` | 명령어/관리자용 실행파일 |
| `/var/log` | 로그 |
| `/tmp` | 임시 파일 |
| `/dev` | 장치 파일 (`/dev/null` 등) |
| `/proc` | 커널/프로세스 정보 |

**"모든 것은 파일"** 철학: 일반 파일, 디렉토리, 장치 파일, `/proc` 모두 동일한 read/write 인터페이스 사용

### `/dev/null`
- 뭘 써도 그냥 **버려지는** 특수 장치 파일
- 불필요한 출력/에러를 숨길 때 사용: `command > /dev/null`

### 절대경로 vs 상대경로
- **절대경로**: `/`로 시작, 항상 동일한 파일 가리킴
- **상대경로**: 현재 위치 기준. `.`(현재), `..`(부모), `~`(홈)

---

## 4. 탐색 명령어

| 명령어 | 기능 |
|---|---|
| `pwd` | 현재 위치 출력 |
| `cd <dir>` / `cd ..` / `cd ~` / `cd -` | 이동 / 부모로 / 홈으로 / 이전 위치로 |
| `ls`, `ls <dir>` | 목록 보기 (이동은 안 함) |

> `cd`는 **셸 내장 명령(builtin)** — 별도 프로그램으로는 부모 셸의 위치를 바꿀 수 없기 때문

### `ls` 주요 옵션

| 옵션 | 효과 |
|---|---|
| `-l` | 상세 정보 |
| `-a` | 숨김 파일(`.`으로 시작) 포함 |
| `-h` | 사람이 읽기 쉬운 크기 단위 (`-l` 필요) |
| `-t` | 수정시간순 정렬(최신 먼저) |
| `-r` | 역순 |
| `-S` | 크기순 정렬(큰 것 먼저) |
| `-d` | 디렉토리 **자체** 정보만 (내용물 아님) |
| `-R` | 하위 디렉토리까지 **재귀적으로** 전부 펼쳐서 보여줌 |

### `ls -l` 출력 해석
```
d rwxr-xr-x 2 root root 4096 Aug 17 09:04 apt
^ ^^^^^^^^^ ^ ^^^^ ^^^^ ^^^^ ^^^^^^^^^^^^ ^^^
타입 권한   링크수 소유자 그룹 크기(byte) 수정시각 이름
```
타입: `-` 일반파일, `d` 디렉토리, `l` 심볼릭 링크

> 디렉토리의 크기는 안에 든 파일들의 총 용량이 아니라, 디렉토리 자체의 "색인(index)" 크기임

### 권한 (`rwx`)

`-rw-r--r--` → 타입 / 소유자 `rw-` / 그룹 `r--` / 기타 `r--`

- 파일의 `x`: **실행 가능**
- 디렉토리의 `x`: **`cd`로 진입 가능**
- `chmod +x file`로 실행 권한 부여

---

## 5. 하드 링크 vs 심볼릭 링크

| | 하드 링크 | 심볼릭 링크 |
|---|---|---|
| 저장하는 것 | 같은 inode(데이터 자체) | **경로(이름) 문자열** |
| `ls -l` 표시 | 타입 `-` | 타입 `l` + `-> 대상경로` |
| 원본 삭제 시 | 영향 없음, 데이터 유지 | 깨진 링크가 됨 |
| 원본을 같은 이름으로 재생성 시 | 무관하게 예전 데이터 유지 | **다시 그 이름을 찾아가서 연결됨** |
| 디렉토리/다른 디스크 | 보통 불가 | 가능 |

강의 예시: `/bin -> usr/bin` (심볼릭 링크로 하위 호환 유지)

> 심볼릭 링크는 데이터가 아니라 **"이름(경로 문자열)"만 저장**하기 때문에, 원본을 지웠다가 같은 이름으로 다시 만들면 다시 연결됨. 하드 링크는 데이터(inode) 자체를 붙잡고 있어서 원본과 무관하게 유지됨.

---

## 6. 파일/디렉토리 조작

| 명령어 | 기능 |
|---|---|
| `mkdir <dir>` / `mkdir -p <path>` | 생성 / 중간 경로까지 생성 |
| `touch <file>` | 빈 파일 생성, 타임스탬프 갱신 |
| `cp <src> <dst>` / `cp -r` | 복사 / 디렉토리 통째로 복사 |
| `mv <src> <dst>` | 이름변경=이동 (같은 연산) |
| `rm <file>` / `rm -r <dir>` | 삭제 |

> ⚠️ **삭제는 되돌릴 수 없음** — 휴지통 없음
> - `rm -rf /`, `~`, `*` 위험
> - 안전 습관: `rm` 전에 `ls`로 확인, `rm -ri`로 항목별 확인

---

## 7. 파일 내용 읽기

| 명령어 | 기능 | 적합한 상황 |
|---|---|---|
| `cat` | 전체 출력 | 짧은 파일 |
| `head` / `head -n 5` | 앞 10줄 / 지정 줄 수 | 형식 확인 |
| `tail` / `tail -f` | 뒤 10줄 / 추가되는 줄 실시간 | 최근 로그 / 로그 모니터링 |
| `less` | 대화형 스크롤 | 큰 파일 |
| `wc` / `wc -l` | 줄/단어/바이트 수 / 줄 수만 | 크기 측정 |

`wc` 출력 순서: **줄, 단어, 바이트**

---

## 8. 도움말

- `<command> --help`: 사용법 요약
- `type <command>`: builtin/alias/외부 프로그램 구분
- `which <command>`: **실제 실행 파일의 경로** 확인 (PATH를 뒤져서 찾음)
- `--help` 출력이 길면 `grep`으로 필요한 옵션만 찾기

---

## 9. 와일드카드, 리다이렉션, 파이프

### 와일드카드

| 패턴 | 매칭 |
|---|---|
| `*` | 임의 길이 문자열 |
| `?` | 정확히 한 문자 |
| `[abc]` | 나열된 문자 중 하나 |
| `[a-z]` | 범위 내 문자 |

**셸의 "확장(expansion)"**: `ls *.c`를 치면, 셸이 프로그램에 넘기기 **전에 먼저** `*.c`를 실제 파일명들(`a.c b.c main.c`)로 바꿔치기함. 프로그램은 `*`를 본 적이 없고, 이미 확장된 결과만 받음.

```bash
$ echo *.c
a.c b.c main.c
$ echo "*.c"
*.c
```

**따옴표**(`"*.c"`)의 역할: 셸의 확장을 **막아서**, 프로그램 자신이 직접 패턴을 해석하게 함. `find`처럼 하위 디렉토리까지 직접 뒤져야 하는 프로그램에 필수.

### 리다이렉션

| 연산자 | 기능 |
|---|---|
| `> file` | 표준출력을 파일에 (덮어쓰기) |
| `>> file` | 파일 끝에 추가 |
| `2> file` | 에러 메시지를 파일에 |
| `< file` | 파일을 표준입력으로 |

### 파이프 (`|`)
한 명령의 출력을 다음 명령의 입력으로 직접 연결 (중간 파일 없음)

자주 쓰는 조합: `grep`(필터), `wc -l`(줄 수), `sort`(정렬), `uniq -c`(중복 개수), `cut`(필드 추출), `head`/`tail`

---

## 10. 핵심 텍스트 처리 도구

### `grep`
패턴이 포함된 줄만 필터링
- `-c`: 매칭된 줄 개수만 출력
- `-i`: **대소문자 구분 없이** 검색 (ignore case)

### `cut`
각 줄을 구분자로 나눠서 특정 필드만 추출
```bash
cut -d<구분자> -f<번호>
```
- `-d:` (콜론), `-d" "` (공백) 등
- `-f7`: 7번째 필드만
- 예: `cut -d: -f7 /etc/passwd` → 로그인 셸 경로만 추출
- **여러 번 이어서 쓰기 가능**: 시간 필드에서 "시"만 뽑으려면 공백으로 먼저 자르고, 그 결과를 콜론으로 다시 자름
  ```bash
  cut -d" " -f2 app.log | cut -d: -f1
  ```

### `sort`
줄 정렬. `-n`(숫자로), `-r`(역순), `-rn`(숫자+역순 조합)

### `uniq`
**연속으로 중복된 줄만** 하나로 합침 (떨어진 중복은 못 잡음 → `sort`와 함께 써야 전체 중복 처리)
- `-c`: 몇 번 나왔는지 개수 표시
- 전형적 조합: `sort | uniq -c | sort -rn`

### `find`
하위 디렉토리까지 포함해서 파일/디렉토리 검색
```bash
find <경로> -name "패턴"
```
- `-name "*.c"`: 이름으로 검색 (따옴표 필수 — 셸 확장 방지)
- `-type f` / `-type d`: 일반 파일만 / 디렉토리만
- `-size +1M`: 1MB 초과
- `2> /dev/null`: 권한 에러 등 버리기

### `ls -R` vs `find` 비교

| | `ls -R` | `find` |
|---|---|---|
| 결과 형태 | 폴더별로 나눠서 보여줌, 경로는 직접 조합해야 함 | 바로 전체 경로 형태로 출력 |
| 용도 | 구조를 눈으로 훑어보기 | 조건에 맞는 것만 콕 집어 찾기 |

---

## 11. 파이프라인 조립 방법론

**완성된 명령을 한 번에 치지 말고, 한 단계씩 추가하며 매번 결과 확인**

```bash
cut -d: -f7 /etc/passwd
cut -d: -f7 /etc/passwd | sort
cut -d: -f7 /etc/passwd | sort | uniq -c
cut -d: -f7 /etc/passwd | sort | uniq -c | sort -rn
```
→ 각 단계 역할을 정확히 이해하고, 문제 생겼을 때 원인 파악이 쉬워짐

### 실전 응용 예시
- **두 파일 합쳐서 세기**: `cat file1 file2 | grep -c ERROR`
- **여러 폴더 파일 중 최댓값 찾기**: `wc -l src/*.c tests/*.c | sort -n | tail -2`
- **시간대별 집계**: `cut -d" " -f2 log | cut -d: -f1 | sort | uniq -c`
- **레벨별 분포**: `cat file1 file2 | cut -d" " -f1 | sort | uniq -c`

---

## 12. GUI vs CLI

| | GUI | CLI |
|---|---|---|
| 조작 | 마우스, 아이콘, 창 | 텍스트 명령어 |
| 삭제 | 휴지통으로 이동(복구 가능) | `rm`은 즉시 영구 삭제 |
| 장점 | 직관적 | 빠름, 자동화 가능 |

---

## Week 02 핵심 요약

> 셸은 작은 명령어들(`ls`, `cd`, `cat`, `grep`, `cut`, `sort`, `uniq`, `find` 등)을 파이프(`|`)와 리다이렉션(`>`, `>>`)으로 조합해서 더 큰 작업을 수행하는 도구이며, 파일시스템은 절대/상대 경로와 권한 체계로 구성되고, 모든 것이 파일이라는 철학 위에 동작한다.

---

# 추가 정리: 복습하며 나온 질문들

## 절대경로 vs 상대경로

| | 출발점 | 적는 것 | 위치가 바뀌면 |
|---|---|---|---|
| 절대경로 | 항상 `/` | 루트부터 **전부** | 그대로 동작 |
| 상대경로 | **지금 위치** | 지금 위치부터 필요한 만큼 | 안 될 수 있음 |

- 상대경로도 방향은 자유: 아래(`a/b/x`), 위(`..`), 옆(`../../c`)
- 중간 폴더는 건너뛸 수 없음: 홈에서 `/home/kim/a/b/x`로 가려면 `cd a/b/x` (`cd x`, `cd b/x` ❌)
- `~/a/b` = `/home/kim/a/b` (셸이 `~`를 바꿔주므로 어디서든 같은 곳)
- 시험 팁: 현재 위치를 적어두고 `..` 하나마다 한 칸씩 따라가기

## `ls`

- 폴더 내용 목록. 파일 이름을 주면 있으면 이름을 stdout, 없으면 에러를 stderr로 출력
- `ls a.log nope > out.txt` → 화면에 에러, `out.txt`에는 `a.log`만
- `-h`는 크기를 K/M/G로 **바꾸기만** 함. 크기는 `-l`이 보여주므로 `ls -lh`로 같이 씀 (혼자 쓰면 효과 없음)
- `-i`: inode 번호 (하드링크는 원본과 같은 번호, 링크 수 2)

## 와일드카드 정밀하게

| 기호 | 글자 수 |
|---|---|
| `?` | 아무 글자 **딱 1개** |
| `[a-z]` | 괄호 안 글자 중 **딱 1개** |
| `*` | 아무 글자 0개 이상 |

- 패턴은 이름 **전체**와 맞아야 함 ("포함"이 아님): `[a-z].c` → `a.c` ✅, `ab.c` ❌, `main.c` ❌
- 소문자로 시작하는 .c 전부: `[a-z]*.c`
- 셸이 명령 실행 **전에** 바꿔치기 → `find . -name "*.log"`처럼 따옴표 필요
- 숨김 파일(`.`으로 시작)은 `*`에 안 잡힘. 매칭이 없으면 글자 그대로 남음

## `sort -rn`이 필요한 이유

- `-r`만: **글자 기준** 비교 후 뒤집기. `9`, `12`, `3` → `9`, `3`, `12` (틀림)
- `uniq -c` 출력은 숫자 앞을 공백으로 채워서 `-r`만 써도 **우연히** 맞을 때가 있음
- 숫자 크기순이 목적이면 항상 `-n`. 큰 것부터면 `-rn`

## 리다이렉션 `>`의 함정

- `> file` 단독: 빈 파일 만들기 / 비우기
- 셸은 명령 실행 **전에** `>` 대상 파일을 먼저 열어 비움 → 명령이 실패해도 기존 내용 사라짐 (4주차 summarize 함정)


---

# Week 02-B. Shell Practice Lab (실습)

## Practice Set 정답 (영상 문제)

```bash
cd ~/oss/w02
ls *.c | wc -l                    # 1. C 소스 개수 → 3
grep -c ERROR app.log             # 2. → 2
grep ERROR app.log | wc -l        #    같은 결과 2
grep ERROR app.log > errors.txt   # 3. 저장 → wc -l 하면 2
grep ERROR app.log >> errors.txt  # 4. 추가 → 2가 4로 늘어남 (> 덮어쓰기, >> 추가)
ls -lhS /var/log | head -2        # 5. 가장 큰 파일 (보통 dpkg.log)
```

## 왜 CLI인가
- 임베디드 보드, 서버, 컨테이너, CI에는 데스크톱이 없음 → ssh 셸이 전부
- 클릭은 기록이 안 남지만 명령은 복사·재실행·공유·스크립트화 가능
- 암기가 목표가 아님: `--help`, Tab 자동완성, history 활용

## grep 옵션

| 옵션 | 효과 |
|---|---|
| (없음) | 매칭 줄 출력 |
| `-c` | 매칭 줄 개수 |
| `-i` | 대소문자 무시 |
| `-n` | 줄 번호 붙이기 |
| `-v` | 패턴 **없는** 줄만 |
| `-r` | 디렉토리 재귀 검색 |
| `-l` | 매칭된 **파일 이름만** |

조합 가능: `grep -rn FIXME src/` → 파일:줄번호:내용

## find
grep은 파일 **내용**, find는 파일 **자체**를 찾음

```bash
find . -name "*.c"        # 하위 전체의 .c
find . -type d            # 디렉토리만
find . -type f            # 일반 파일만
find . -size +1M          # 1MB 초과
find /etc -name "*.conf" 2> /dev/null   # 에러 버리기
find . -name "*.c" | wc -l
```
- `ls *.c`는 한 디렉토리만, `find`는 트리 전체
- 패턴에 따옴표 → 셸이 아니라 find가 패턴을 받게 함

## sort / uniq
```bash
sort scores.txt       # 텍스트 순: "100"이 "21"보다 앞
sort -n scores.txt    # 숫자 순
sort fruits.txt | uniq -c | sort -rn   # 빈도 순위 관용구
```
- uniq는 **인접한** 중복만 합침 → 반드시 sort 먼저

## 하드 링크 vs 심볼릭 링크

| 하드 링크 `ln 대상 이름` | 심볼릭 링크 `ln -s 대상 이름` |
|---|---|
| 같은 데이터의 두 번째 이름 | 경로를 저장하는 작은 파일 |
| 둘 다 동등, 원본 구분 없음 | `ls -l`에 `이름 -> 대상` |
| 모든 이름을 지워야 데이터 삭제 | 대상 삭제 시 깨짐 |
| 같은 파일시스템, 파일만 | 파일시스템 넘나듦, 디렉토리 가능 |

## 셸 단축키

| 키 | 효과 |
|---|---|
| Tab | 자동완성 (두 번 → 후보 목록) |
| ↑ / ↓ | 이전 명령 불러오기 |
| Ctrl+R | 히스토리 검색 |
| Ctrl+C | 현재 줄/실행 중 명령 취소 |
| Ctrl+L | 화면 지우기 (= clear) |
| Ctrl+A / Ctrl+E | 줄 처음 / 끝으로 |

## 숨김 파일과 .bashrc
- `.`으로 시작하는 이름은 `ls`에 안 보임 → `ls -a`
- `~/.bashrc`: 셸이 시작할 때마다 실행하는 명령 목록 (`export LANG=C.UTF-8`, `alias ll='ls -l'`)

## 권한 읽기
```
-rw-r--r--  →  - | rw- | r-- | r--
            타입  소유자  그룹  기타
```
- 파일의 x = 실행 가능, 디렉토리의 x = cd로 진입 가능
- `chmod +x file`로 실행 권한 부여

## PATH와 ./
- 명령 이름은 PATH 디렉토리에서만 순서대로 찾음
- 현재 디렉토리는 PATH에 없음 → `./`가 필요
- 내 파일을 명령처럼 실행하려면 **`./` + x 권한** 두 가지가 필요

## 실습 준비
```bash
curl -O <setup_w02.sh URL>
bash setup_w02.sh
tar -xzf oss-w02.tar.gz     # -x 풀기, -z gzip, -f 이 파일
cd oss-w02                  # README.txt data docs logs src tests
```

## Lab 1. 탐색과 와일드카드
```bash
find . -type d                  # 트리 구조 먼저 파악
ls src/*.c                      # 3개 (한 단계만)
find . -name "*.c" | wc -l      # 7개 (전체 트리)
cat src/old/legacy.c            # Checkpoint 1: FIXME 줄
```
- 차이 4개 = src/old의 옛 소스 2개 + tests/의 테스트 2개

## Lab 2. 복사, 이동, 삭제
```bash
cp -r logs logs_backup                               # 디렉토리 복사는 -r
mv logs_backup/old.log logs_backup/archive.log       # 이름 변경 = 이동
touch tmp1.txt tmp2.txt draft.txt
ls tmp*.txt          # 같은 패턴으로 먼저 확인 (안전 습관)
rm tmp*.txt
rm -r logs_backup draft.txt                          # 디렉토리는 -r
```
Task: data/로 같은 과정 반복
```bash
cp -r data data_backup
mv data_backup/fruits.txt data_backup/fruits_old.txt
ls -d data_backup
rm -r data_backup
```

## Lab 3. grep과 리다이렉션
```bash
cd ~/oss-w02/logs
head -3 app.log                 # 측정 전에 데이터부터 보기
grep -c ERROR app.log           # 12
grep -c ERROR app.log > count.txt
cat count.txt                   # Checkpoint 2: 12
grep ERROR app.log > errors.txt; wc -l errors.txt    # Task A: 12
grep -in error app.log | grep -v ERROR               # Task B: 12 vs 14의 차이 2줄
```
- `grep -ci error`가 14인 이유: "0 error rows", "no error detected"처럼 **문제없다는 INFO 줄**에도 error라는 글자가 있음
- grep은 의미가 아니라 **글자**를 매칭함

## Lab 4. 파이프
```bash
grep ERROR app.log | wc -l      # 12, 중간 파일 없이
grep ERROR app.log | cut -d" " -f4 | sort | uniq -c | sort -rn
#   5 db / 3 net / 2 auth / 1 ui / 1 sys   ← Checkpoint 3
grep -c ERROR *.log             # 여러 파일 → 파일명:개수
# app.log:12
# old.log:3
```
- sort 없이 `uniq -c` → 인접한 것만 합쳐져서 개수가 쪼개짐

## Lab 5. find와 sort
```bash
grep -rn FIXME src                       # 알려진 문제 위치
find . -size +1M                         # ./data/big.bin
find . -name "*.md"                      # A: 3개
sort -n data/scores.txt | tail -3        # B: 87 kim / 100 choi / 100 lim
sort data/fruits.txt | uniq -c | sort -rn            # C: apple 5
cut -d" " -f4 logs/app.log | sort | uniq -c | sort -rn   # D: db 51 (200줄 전체라 grep 없이 cut부터)
```

## Lab 6. 링크 실습
```bash
echo "the original line" > original.txt
ln original.txt hard.txt
ls -li original.txt hard.txt    # 같은 inode 번호, 링크 수 2
ln -s original.txt soft.txt
ls -l soft.txt                  # soft.txt -> original.txt
rm original.txt
cat hard.txt                    # 데이터 살아있음
cat soft.txt                    # No such file or directory
```

## Lab 7. 파일을 명령으로 실행
```bash
echo "echo hello from my first command" > hello.sh
bash hello.sh       # OK (bash가 파일을 읽음)
./hello.sh          # Permission denied (x 권한 없음)
chmod +x hello.sh
./hello.sh          # OK → Checkpoint 4
hello.sh            # command not found (PATH에 . 없음)
```

## Lab 8. nano로 편집
- `nano hello.sh` → 저장 `Ctrl+O` + Enter, 종료 `Ctrl+X`
- 하단의 `^O`에서 `^`는 Ctrl

## Extra 문제 정답

| # | 답 | 명령 |
|---|---|---|
| 1 | 5 | `grep -c WARN logs/old.log` |
| 2 | 15 | `cat logs/*.log \| grep -c ERROR` 또는 `grep ERROR logs/*.log \| wc -l` |
| 3 | 4 | `grep -r TODO docs \| wc -l` |
| 4 | ERROR 12줄 | `grep -v INFO logs/app.log \| grep -v WARN` |
| 5 | 5개 | `find . -name "*.txt"` |
| 6 | 100 choi, 100 lim, 21 shin… | 텍스트 정렬이라 "100"이 "21" 앞 |

※ 2번: `grep -c ERROR logs/*.log`는 파일별로 따로 나와서(12, 3) 합계가 아님

## Challenge 정답
```bash
cut -d" " -f2 logs/app.log | cut -d: -f1 | sort | uniq -c   # 1. 시간대별: 176 09 / 24 10
grep ERROR logs/app.log | grep " db " > db_errors.txt        # 2. → 5줄
cat logs/*.log | cut -d" " -f3 | sort | uniq -c | sort -rn   # 3. 179 INFO / 36 WARN / 15 ERROR
wc -l src/*.c src/old/*.c tests/*.c | sort -n | tail -2      # 4. 최대: src/parser.c 17줄
echo "restored" > original.txt; cat soft.txt                 # 5. restored
```
- 5번: 심볼릭 링크는 **경로만** 저장 → 같은 경로를 다시 만들면 다시 연결됨

## 파이프라인 = 프로그램

| 단계 | 역할 |
|---|---|
| `grep ERROR` | 관련 레코드 선택 |
| `cut -d" " -f4` | 필드 하나 추출 |
| `sort` | 같은 값끼리 모으기 |
| `uniq -c` | 그룹별 개수 |
| `sort -rn` | 순위 매기기 |

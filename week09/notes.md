# Week 09-A. File I/O (측정 데이터 파일 입출력)

## 1. 이번 주 프로젝트: power-report

전력 측정 CSV를 읽어 **검사 → 시간별 집계 → CSV 보고서**로 쓰는 C 프로그램입니다.

```
data/power_20061221.csv → C 프로그램(검사 + 집계) → hourly.csv → viewer.html(그래프)
```

- 데이터: UCI 가정용 전력 데이터셋(CC BY 4.0) 중 **2006-12-21 하루치 = 1440분**
- C가 계산하고, JavaScript(viewer.html)는 CSV를 읽어 그림만 그림 → **파일이 프로그램 사이의 경계**
- 파일 경계의 장점: 역할 분리, 중간 결과를 눈으로 확인, 엑셀·파이썬·테스트가 재사용, 문제 위치 찾기 쉬움

### 입력 형식

```
minute,power_kw
0,1.814
683,?
```

| 항목 | 의미 |
|---|---|
| `minute` | 하루 안의 몇 번째 분 (0~1439). **시 = minute / 60** |
| `power_kw` | 1분 평균 유효전력 (kW) |
| `?` | **측정이 빠짐 (missing)** |
| `0.0` | **실제로 측정된 0 (valid)** |

### 출력 형식 (hourly.csv)

```
hour,valid,missing,mean_kw,peak_kw
11,58,2,....,....
```

- 보통 valid 60 / missing 0. 실제 데이터는 **11시만 58 / 2** (683, 684분이 `?`)
- 엑셀은 `00`→`0`, `0.430`→`0.43`으로 **표시만** 바꿈

### 폴더 구조

| 파일 | 역할 |
|---|---|
| `include/power.h` | `Hour` 구조체 + 함수 선언 |
| `src/main.c` | 전체 흐름 (열기 → 읽기 → 저장) |
| `src/parse.c` | 한 줄 검사 → `parse_row` |
| `src/load.c` | 시간별 집계 → `load_hours` (**실습**) |
| `src/report.c` | CSV 형식 출력 → `emit_csv` |
| `src/save.c` | 새 파일로 안전하게 저장 → `save_report` (**실습**) |
| `src/inspect.c` | open/read 바이트 단위 시연 |

```
main ─┬─ load_hours  → parse_row   (읽기·검사)
      └─ save_report → emit_csv    (쓰기)
```

## 2. main.c 흐름과 종료 상태

```c
if (argc < 2 || argc > 3) { fprintf(stderr, "usage: ..."); return 2; }
FILE *in = fopen(argv[1], "r");
if (!in) { perror(argv[1]); return 1; }
Hour hours[24] = {0};
int rc = load_hours(in, hours);
if (fclose(in) == EOF) { perror("close input"); rc = -1; }
if (rc < 0) return 1;                      // 읽기 실패면 보고서를 만들지 않음
if (argc == 3) rc = save_report(argv[2], hours);
else { rc = emit_csv(stdout, hours); if (fflush(stdout) == EOF) rc = -1; }
if (rc < 0) { perror("write report"); return 1; }
return 0;
```

| 반환 | 의미 |
|---|---|
| 0 | 성공 |
| 1 | 실행 중 실패 (입력 없음, 잘못된 데이터, 쓰기 실패) |
| 2 | 사용법 오류 |

- 에러 메시지는 전부 **stderr** → `> out.csv` 해도 CSV만 깔끔하게 저장

## 3. 파일을 다루는 두 층 (핵심!)

| | 저수준 (시스템 콜) | 고수준 (C 라이브러리) |
|---|---|---|
| 함수 | `open / read / write / close` | `fopen / getline / fprintf / fclose` |
| 가리키는 것 | **`int fd`** (정수 번호) | **`FILE *`** (스트림 포인터) |
| 버퍼 | 없음 (바로 커널) | **있음** (모았다가 한 번에) |
| 단위 | 바이트 | 줄, 형식 있는 텍스트 |

```
C 프로그램 → C 라이브러리(버퍼, wrapper) → 리눅스 커널
```

- `fopen` 안에서 결국 `open`이 불림. `FILE *` = fd + 버퍼 + 상태
- 버퍼를 쓰는 이유: 시스템 콜은 비쌈 → 모아서 한 번에
- 버퍼의 대가: **`fprintf` 성공 ≠ 파일에 써짐** → `fflush`/`fclose` 반환값까지 확인

## 4. 파일 디스크립터와 표준 스트림

| fd | 이름 | C에서 | 기본 |
|---|---|---|---|
| 0 | 표준 입력 | `stdin` | 키보드 |
| 1 | 표준 출력 | `stdout` | 화면 |
| 2 | 표준 에러 | `stderr` | 화면 |

- `open` 성공 → **0 이상** (보통 비어 있는 가장 작은 번호 = 3), 실패 → **-1**
- fd에는 **정수만** 저장됨. 파일 이름, 모드, 현재 위치 같은 진짜 정보는 **커널의 열린 파일 표**에 있음 (fd = 그 표의 칸 번호)

```bash
./bin/inspect data/power_20061221.csv > output/first64.txt 2> output/read.log
```
- `>` = `1>` (stdout), `2>` = stderr. 리다이렉션은 **실행 전에 셸이** fd를 바꿔 끼움

## 5. open / read (inspect.c)

```c
int fd = open(argv[1], O_RDONLY);
if (fd < 0) { perror("open"); return 1; }
char buf[64];
ssize_t n = read(fd, buf, sizeof buf);
if (n < 0) { perror("read"); close(fd); return 1; }
int rc = fwrite(buf, 1, (size_t)n, stdout) == (size_t)n ? 0 : 1;   // '\0' 없으니 개수로 출력
fprintf(stderr, "read returned %zd bytes\n", n);
if (fflush(stdout) == EOF) rc = 1;
if (close(fd) < 0) rc = 1;
return rc;
```

| open 접근 모드 | 뜻 |
|---|---|
| `O_RDONLY` | 읽기 (fopen `"r"`) |
| `O_WRONLY` | 쓰기 |
| `O_RDWR` | 읽기 + 쓰기 |

- 실패 원인은 **errno**에 저장 → `perror`가 문장으로 stderr에 출력 (`open: No such file or directory`)

### read 반환값

| 반환 | 의미 |
|---|---|
| `n > 0` | n바이트 읽음 (요청보다 **적을 수 있음**) |
| `0` | **EOF** |
| `-1` | 에러 (errno) |

- read로 읽은 건 문자열이 아니라 **바이트 덩어리** (`'\0'` 없음)

## 6. EOF와 빈 입력

- **read는 EOF에서 0, getline은 EOF에서 -1** → 함수마다 약속이 다름
- 빈 파일: read 입장에서는 정상 EOF, 프로그램 입장에서는 **헤더가 없으니 실패**
- 헤더만 있는 CSV: 성공, 24시간 모두 mean/peak **빈칸**

## 7. write와 부분 쓰기

- `write`는 요청보다 **적게** 받아들일 수 있음 → 남은 것을 다시 써야 함

```c
size_t done = 0;
while (done < length) {
    ssize_t n = write(fd, buf + done, length - done);
    if (n < 0 && errno == EINTR) continue;   // 시그널로 끊김 → 재시도
    if (n < 0) return -1;
    if (n == 0) { errno = EIO; return -1; }  // 무한 반복 방지
    done += (size_t)n;
}
return 0;
```

## 8. 바이트 덩어리 vs 줄

```
파일:    690,0.532\n691,?\n
read #1: 690,0.5
read #2: 32\n691,?\n       ← 한 줄이 두 번에 쪼개짐
```
- read는 **줄 경계를 모름** → 줄 단위 처리는 **getline**

## 9. getline

```c
char *line = NULL;
size_t capacity = 0;
while (getline(&line, &capacity, in) >= 0) {
    /* 한 줄 처리 (줄 끝 '\n' 포함) */
}
if (ferror(in)) { /* 읽기 에러 */ }
free(line);
```

- 주소를 넘김 → getline이 **메모리를 알아서 할당·확장**하고 line, capacity를 바꿔 줌
- 반환: 읽은 글자 수 / **-1 = EOF 또는 에러 → `ferror`로 구분**
- 끝나면 `free(line)` 한 번

## 10. 검사 규칙과 parse_row

| 통과 | 거부 (이유) |
|---|---|
| `0,1.536` | `1440,1.2` (minute 범위 밖) |
| `1,?` (missing) | `12,NaN` (유한한 수 아님) |
| `2,0.0` (측정된 0) | `12,abc` (숫자 아님) |
| | `12,-1` (음수 전력) |

`parse_row` 반환: **1 = 측정값, 0 = `?`, -1 = 잘못된 줄**

- `strtol(line, &end, 10)` → `end == line`(숫자 없음), errno(넘침), `*end != ','`, 범위 밖 → -1
- 값: `strtod(value, &end)` → `end == value`, errno, `!isfinite`, 음수, 숫자 뒤에 줄바꿈 말고 다른 글자 → -1
- **end 포인터 = 어디까지 숫자였는지**

## 11. 집계와 빠진 값

```c
typedef struct { unsigned valid, missing; double sum, peak; } Hour;
```

| 입력 (11시) | valid | missing | 평균 |
|---|---|---|---|
| 1.0, ?, 3.0 | 2 | 1 | **2.000** (3으로 안 나눔) |
| 유효값 없음 | 0 | 따로 기록 | **빈칸** |

- `?`를 0으로 치면 (1+0+3)/3 = 1.33 → **틀림**. "모른다" ≠ "0이다"

### emit_csv

```c
fprintf(out, "hour,valid,missing,mean_kw,peak_kw\n");
for (int h = 0; h < 24; h++)
    if (hours[h].valid) fprintf(out, "%02d,%u,%u,%.3f,%.3f\n", h, v, m, sum / v, peak);
    else                fprintf(out, "%02d,0,%u,,\n", h, m);
```

## 12. stdout vs 이름 있는 보고서

| 셸이 결정 | 프로그램이 결정 |
|---|---|
| `./bin/power-report data.csv > output/hourly.csv` | `./bin/power-report data.csv output/new-report.csv` |
| `>`는 셸이 **기존 내용을 먼저 지움** (프로그램이 실패해도) | 생성 정책(이미 있으면 거부)을 **프로그램이 직접** 정함 |

## 13. 안전하게 새 파일 만들기

```c
int fd = open(path, O_WRONLY | O_CREAT | O_EXCL, 0644);
```

| 부분 | 뜻 |
|---|---|
| `O_CREAT` | 없으면 새로 만듦 |
| `O_EXCL` | O_CREAT와 함께: **이미 있으면 실패** |
| `\|` | 옵션 합치기 |
| `0644` | 새 파일 권한 rw-r--r-- |

```
open(path, flags) → fd → fdopen(fd, "w") → FILE* → fprintf → fclose
```

- open은 생성 규칙(O_EXCL, 권한)을, FILE\*는 fprintf의 편리함을 → **두 장점을 합침**
- fdopen 실패 → `close(fd)` 직접 + unlink
- fdopen 성공 후에는 **fclose 한 번이 FILE\*와 fd를 모두 닫음** (close 또 하면 이중 닫기)
- fclose = 버퍼에 남은 걸 **write로 fd(→ 실제 파일)에 내보내고** 닫기 → **fclose까지 성공해야 진짜 성공**

## 14. 실패 시 파일 처리 (서술형 1순위)

| 상황 | 관찰 | 출력 파일 |
|---|---|---|
| 입력 파일 없음 | open 실패 | **만들지 않음** |
| 잘못된 레코드 | 검사 실패 | **만들지 않음** |
| 보고서 경로가 이미 있음 | O_EXCL 실패 | **기존 파일 보존** |
| 쓰기/닫기 오류 | 출력 실패 | **불완전한 새 파일 삭제** (unlink) |

- 이유: **잘못되거나 덜 써진 결과 파일이 남지 않게** + **기존 결과를 덮어써서 잃지 않게**

## 15. strace

```bash
strace -e trace=openat,read,write,close -o output/inspect.trace ./bin/inspect data.csv
```

| 추적 줄 | 해석 |
|---|---|
| `openat(..., O_RDONLY) = 3` | fd 3 받음 |
| `read(3, ..., 64) = 64` | 64 요청 → 64 읽음 |
| `write(2, "read returned...", 23)` | **stderr가 먼저** |
| `write(1, "minute,...", 64)` | stdout은 버퍼에 있다가 fflush 때 |

- 코드에선 fwrite가 먼저인데 stderr가 먼저 찍힘 → **stdout 버퍼링 증거**
- power-report는 getline으로 한 줄씩 읽지만 실제로는 `read(3, ..., 4096)` 몇 번 → **버퍼가 시스템 콜 횟수를 줄임**

## Week 09-A 핵심 요약

- fd(정수, 버퍼 없음) vs FILE\*(버퍼 있음). fd 0/1/2 = stdin/stdout/stderr
- 반환값: open -1, read 0=EOF, getline -1(→ferror), write 일부만, fopen NULL, fclose EOF
- `?` ≠ `0.0`, 평균 분모 = valid
- open(O_CREAT|O_EXCL) → fdopen → fprintf → **fclose 확인** → 실패면 unlink
- 반환값을 확인하고, 실패를 숨기지 않고, 잘못된 파일을 남기지 않는다

---

# Week 09-B. File I/O Lab (load.c, save.c)

## 준비

```bash
base=https://gist.githubusercontent.com/Kyungho-Won
id=f1b4aec11633fcb4a74d0d99b705a08e
curl -L "$base/$id/raw/setup_w09.sh" -o setup_w09.sh
bash setup_w09.sh && cd oss-w09
make && make check-base          # 3개만 통과 (시작 상태)
./bin/power-report data/power_20061221.csv   # load_hours: Function not implemented
```

- 셋업은 이미 있는 `oss-w09` 폴더를 덮어쓰지 않음 → 이름이 겹치면 새 폴더에서

## load_hours 구현

```c
int load_hours(FILE *in, Hour hours[24])
{
    char *line = NULL;
    size_t capacity = 0;
    int rc = -1, previous = -1;                       // 실패로 시작, 끝까지 가야 0

    if (getline(&line, &capacity, in) < 0) goto done; // 빈 파일
    line[strcspn(line, "\r\n")] = '\0';               // 줄 끝 \n, \r 제거
    if (strcmp(line, "minute,power_kw") != 0) goto done;

    while (getline(&line, &capacity, in) >= 0) {
        int minute; double power;
        int kind = parse_row(line, &minute, &power);
        if (kind < 0 || minute <= previous) goto done; // 잘못된 줄, 중복/역순
        previous = minute;
        Hour *s = &hours[minute / 60];
        if (kind == 0) { s->missing++; continue; }    // ? → missing만
        if (s->valid == 0 || power > s->peak) s->peak = power;
        s->valid++;
        s->sum += power;
    }
    if (ferror(in)) { perror("read input"); goto done; }
    rc = 0;
done:
    free(line);                                       // 성공이든 실패든 한 번
    return rc;
}
```

- 인자: `FILE *in` = main이 **이미 fopen한** 입력 (닫는 건 main) / `Hour hours[24]` = main이 0으로 초기화해서 넘긴 결과 칸 (주소로 넘어가서 채운 값이 main에 남음)
- `strcspn(line, "\r\n")` = 처음 나오는 줄바꿈 위치 → 거기에 `'\0'` → 문자열을 자름 (`\r`까지 지워서 Windows 파일도 통과)
- `ferror` 없으면 읽다가 에러가 나도 "끝까지 잘 읽었다"고 착각 → 반쪽 보고서
- `previous = -1`로 시작 → 첫 레코드 minute 0도 통과, `<=`라 중복도 거부

## save_report 구현

```c
int save_report(const char *path, const Hour hours[24])
{
    int fd = open(path, O_WRONLY | O_CREAT | O_EXCL, 0644);
    if (fd < 0) return -1;                    // 기존 파일 손대지 않음

    FILE *out = fdopen(fd, "w");
    if (!out) {
        int saved = errno;
        close(fd);                            // 스트림이 없으니 fd 직접
        unlink(path);                         // 빈 새 파일 삭제
        errno = saved;
        return -1;
    }

    int rc = emit_csv(out, hours);
    int saved = rc < 0 ? errno : 0;
    if (fclose(out) == EOF) {                 // 버퍼가 실제로 써지는 순간
        if (rc == 0) saved = errno;           // 첫 에러 유지
        rc = -1;
    }
    if (rc < 0) { unlink(path); errno = saved; }   // 덜 써진 새 파일 삭제
    return rc;
}
```

- errno 저장/복원: close·unlink가 errno를 바꿀 수 있어서 **원래 실패 이유**를 지킴
- 성공 = open ✓ → fdopen ✓ → emit_csv 0 → fclose 0

```
open(O_EXCL) ─실패→ return -1           (기존 파일 그대로)
fdopen ──────실패→ close + unlink, -1   (빈 새 파일 삭제)
emit_csv/fclose 실패→ unlink, -1        (덜 써진 새 파일 삭제)
전부 성공 → return 0                     (완성된 파일만 남음)
```

## 확인

```bash
mkdir -p output                                   # 폴더 없으면 open이 ENOENT로 실패
./bin/power-report data/power_20061221.csv output/hourly.csv; echo $?   # 0
cp output/hourly.csv output/before.csv
./bin/power-report data/power_20061221.csv output/hourly.csv; echo $?   # 1 (File exists)
cmp output/before.csv output/hourly.csv && echo same                    # same
printf 'minute,power_kw\n0,NaN\n' > output/bad.csv
./bin/power-report output/bad.csv output/bad-report.csv; echo $?        # 1
test ! -e output/bad-report.csv && echo "no file"                       # no file
make check                                                               # 19/19 passed
```

- `write report: No such file or directory` → **output 폴더가 없는 것** (O_CREAT는 폴더까지 만들지 않음)

## 추가 활동: check_extra.sh

```bash
curl -L "$base/$id/raw/setup_w09_activities.sh" -o setup_w09_activities.sh
bash setup_w09_activities.sh      # activities/{valid,invalid,missing}.csv + tests/check_extra.sh
```

| 경우 | 예측 |
|---|---|
| 정상 입력 → 새 경로 | 0, 파일 생성 |
| 잘못된 입력 → 새 경로 | 1, 파일 없음 |
| 정상 입력 → 이미 있는 경로 | 1, 기존 파일 그대로 |

```bash
#!/usr/bin/env bash
set -eu
work=$(mktemp -d)
trap 'rm -rf "$work"' EXIT

./bin/power-report activities/missing.csv "$work/missing.csv"
grep -qx '02,0,2,,' "$work/missing.csv"

./bin/power-report activities/valid.csv "$work/report.csv"
cp "$work/report.csv" "$work/before.csv"
if ./bin/power-report activities/valid.csv "$work/report.csv"; then
    echo "FAIL: existing report was replaced" >&2; exit 1
fi
cmp -s "$work/before.csv" "$work/report.csv"

if ./bin/power-report activities/invalid.csv "$work/bad.csv"; then
    echo "FAIL: invalid input accepted" >&2; exit 1
fi
test ! -e "$work/bad.csv"

echo "PASS extra checks"
```

| 부분 | 뜻 |
|---|---|
| `set -eu` | 명령 실패 시 즉시 종료 / 미정의 변수 에러 |
| `mktemp -d` + `trap ... EXIT` | 임시 폴더, 끝나면 자동 삭제 |
| `grep -qx` | 출력 없이 상태만 / 줄 전체 일치 |
| `if 명령; then FAIL` | 실패해야 할 명령이 성공하면 FAIL |
| `cmp -s` / `test ! -e` | 내용 그대로인지 / 파일이 없는지 |

- 테스트는 **종료 상태 + 파일 상태** 둘 다 확인
- 버그를 일부러 넣었을 때 FAIL이 나야 좋은 테스트 (회귀 테스트):
  ```bash
  sed -i 's/O_EXCL, 0644/O_TRUNC, 0644/' src/save.c && make && bash tests/check_extra.sh; echo $?   # FAIL, 1
  sed -i 's/O_TRUNC, 0644/O_EXCL, 0644/' src/save.c && make && bash tests/check_extra.sh           # PASS
  ```

## GitHub Actions

```yaml
# .github/workflows/check.yml
steps:
  - uses: actions/checkout@v4
  - run: make check
  - run: bash tests/check_extra.sh
```

- 로컬과 CI가 **같은 명령** → 결과 비교 쉬움
- `.gitignore`에 `bin/`, `output/` → 빌드 결과물은 커밋하지 않음
- 거부 테스트 중 나오는 에러 메시지는 **기대된 것**

## Week 09-B 핵심 요약

- load.c: 헤더 → getline 반복 → parse_row → `hours[minute/60]` → ferror → free
- save.c: open(O_EXCL) → fdopen → emit_csv → fclose → 실패면 unlink
- 테스트: 종료 상태 + 파일 상태, 실패해야 하는 경우도 테스트

---

# Week 09 과제: 시간 범위별 집계

- `./bin/power-range 입력 START END [보고서]` → **[START, END)**: 시작 포함, 끝 제외
- 예: `10 12` → 10시, 11시 / 600분·719분 포함, 599분·720분 제외 (경계값 테스트)

```c
/* src/load.c, TODO L1: previous = minute; 다음 */
if (minute / 60 < start || minute / 60 >= end) continue;

/* src/report.c, TODO R1 */
for (int h = start; h < end; h++) {
```

- `previous = minute` **다음**에 넣어야 범위 밖 줄도 순서 검사를 받음
- 시작본: `make check` 19/19, `make check-range` 3/10 → 수정 후 10/10
- 제출: 고치기 전에 `git commit --allow-empty -m "Check starter"` → push → Actions 실패 캡처 / 수정 후 push → 성공 캡처 → PPT 2장 (실패 이유, 변경 내용 각 1~2문장)

---

# 추가 정리: 복습하며 나온 질문들

## fd에는 뭐가 저장되나

정수 하나 (예: 3). 진짜 정보(파일, 모드, 위치)는 커널의 열린 파일 표에 있고 fd는 그 칸 번호. 은행 대기 번호표처럼 숫자만 들고 다님

## 왜 fd를 FILE 스트림으로 바꾸나

fd로는 fprintf를 못 씀 → snprintf + write + 부분 쓰기 반복을 직접 해야 함. FILE\*는 형식 출력, 버퍼, 부분 쓰기 처리를 라이브러리가 해 줌. `emit_csv`도 FILE\*를 받음

## fopen으로 바로 열면 안 되나

`fopen(path, "wx")`(C11)도 "이미 있으면 실패"가 되지만 권한(0644)을 지정할 수 없고, 수업 목표가 fd ↔ FILE\* 연결과 fd 소유권이라 실습은 open + fdopen

| mode | 없으면 | 있으면 |
|---|---|---|
| `"r"` | 실패 (NULL) | 처음부터 읽음 |
| `"w"` | 새로 만듦 | **내용을 지움** |
| `"a"` | 새로 만듦 | 끝에 덧붙임 |
| `"wx"` | 새로 만듦 | 실패 |

## 버퍼가 없으면

| | 버퍼 있음 (FILE\*, stdout) | 버퍼 없음 (fd, stderr) |
|---|---|---|
| 속도 | 빠름 (시스템 콜 적음) | 느림 (매번 커널) |
| 출력 시점 | flush/fclose 때 | 즉시 |
| 실패 확인 | fclose/fflush까지 | write 반환값 바로 |
| 부분 쓰기 | 라이브러리가 처리 | 직접 반복 |

## errno

방금 실패한 함수가 **왜** 실패했는지 번호로 적어 두는 전역 변수 (`<errno.h>`)

| errno | 뜻 |
|---|---|
| `ENOENT` | 파일/디렉터리 없음 |
| `EEXIST` | 이미 있음 (O_EXCL) |
| `EACCES` | 권한 없음 |
| `ENOSPC` | 디스크 공간 없음 |
| `EINTR` | 시그널로 끊김 |
| `ENOSYS` | 구현 안 됨 (시작본) |

- **반환값으로 실패를 확인한 뒤에** 봄 (성공하면 안 바뀜)
- 다른 함수가 덮어쓸 수 있으니 필요하면 저장했다가 복원

## 함수 전부 외워야 하나

이름, 역할, 반환값만. 인자 순서(`fwrite(buf, 1, n, stdout)` 등), `ssize_t`, `%zd`는 안 외워도 됨. 짝으로 기억: open ↔ close, fopen ↔ fclose, open → fdopen이면 fclose 한 번

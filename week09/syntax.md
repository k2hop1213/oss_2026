# 9주차 문법: File I/O (C)

명령어 설명 없이 문법 형태만 빠르게 찾아보는 용도.

## 헤더

```c
#include <stdio.h>    // FILE, fopen, getline, fprintf, fclose, perror, ferror, fflush
#include <stdlib.h>   // free
#include <string.h>   // strcmp, strcspn
#include <errno.h>    // errno, ENOENT, EEXIST, EINTR ...
#include <fcntl.h>    // open, O_RDONLY, O_WRONLY, O_CREAT, O_EXCL
#include <unistd.h>   // read, write, close, unlink
```

## 반환값 표

| 함수 | 성공 | 끝(EOF) | 실패 |
|---|---|---|---|
| `open` | fd (0 이상) | | -1 |
| `read` | 읽은 바이트 수 | 0 | -1 |
| `write` | 쓴 바이트 수 (일부만 가능) | | -1 |
| `close` | 0 | | -1 |
| `fopen` / `fdopen` | FILE \* | | NULL |
| `getline` | 읽은 글자 수 | -1 | -1 (→ `ferror`) |
| `fprintf` | 쓴 글자 수 | | 음수 |
| `fflush` / `fclose` | 0 | | EOF |

## fd 계열

```c
int fd = open(path, O_RDONLY);
int fd = open(path, O_WRONLY | O_CREAT | O_EXCL, 0644);
if (fd < 0) { perror("open"); return 1; }

ssize_t n = read(fd, buf, sizeof buf);       // n > 0 / 0 EOF / -1
ssize_t n = write(fd, buf + done, length - done);
close(fd);
unlink(path);
```

```
O_RDONLY  O_WRONLY  O_RDWR      접근 모드 (하나)
O_CREAT                          없으면 만듦
O_EXCL                           (O_CREAT와) 이미 있으면 실패
0644                             rw-r--r--
```

## FILE 계열

```c
FILE *in  = fopen(path, "r");                // "r" "w" "a" "wx"
FILE *out = fdopen(fd, "w");                 // fd → FILE*
if (!in) { perror(path); return 1; }

fprintf(out, "%02d,%u,%u,%.3f,%.3f\n", h, v, m, mean, peak);
fwrite(buf, 1, (size_t)n, stdout);
if (fflush(stdout) == EOF) rc = -1;
if (fclose(out) == EOF) rc = -1;             // fdopen 후엔 fd도 같이 닫힘
```

## getline 반복

```c
char *line = NULL;
size_t capacity = 0;
while (getline(&line, &capacity, in) >= 0) {
    /* ... */
}
if (ferror(in)) { perror("read input"); }
free(line);
```

## 줄 끝 제거 / 비교

```c
line[strcspn(line, "\r\n")] = '\0';
if (strcmp(line, "minute,power_kw") != 0) { /* bad header */ }
```

## 숫자 변환

```c
char *end;
errno = 0;
long m = strtol(line, &end, 10);   // end == line → 숫자 없음
double p = strtod(value, &end);    // end == value → 숫자 없음
isfinite(p)                        // NaN, inf 거름
```

## errno 저장/복원

```c
int saved = errno;
close(fd);
unlink(path);
errno = saved;
```

## goto 정리 패턴

```c
int rc = -1;
if (/* 실패 */) goto done;
rc = 0;
done:
    free(line);
    return rc;
```

## 부분 쓰기 반복

```c
while (done < length) {
    ssize_t n = write(fd, buf + done, length - done);
    if (n < 0 && errno == EINTR) continue;
    if (n < 0) return -1;
    done += (size_t)n;
}
```

## 시간 계산

```c
int hour = minute / 60;            // 0~1439 → 0~23
Hour *s = &hours[minute / 60];
mean = s->sum / s->valid;          // valid == 0이면 빈칸
```

## 셸 / 검증

```bash
./prog in.csv > out.csv 2> err.log      # 1> stdout, 2> stderr
mkdir -p output
cmp a.csv b.csv && echo same
test ! -e output/bad.csv
grep -qx '02,0,2,,' report.csv
strace -e trace=openat,read,write,close -o out.trace ./prog in.csv
```

```bash
set -eu
work=$(mktemp -d)
trap 'rm -rf "$work"' EXIT
if ./prog bad.csv "$work/x.csv"; then echo FAIL >&2; exit 1; fi
```

## 과제: 범위 [start, end)

```c
if (minute / 60 < start || minute / 60 >= end) continue;
for (int h = start; h < end; h++) { /* ... */ }
```

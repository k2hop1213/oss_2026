# 11주차 문법: 시그널

명령어 설명 없이 문법 형태만 빠르게 찾아보는 용도.

## 헤더

```c
#include <signal.h>     // sigaction, sigset_t, sigprocmask, kill, SIG*
#include <unistd.h>     // alarm, getpid
#include <sys/wait.h>   // waitpid, WNOHANG
#include <time.h>       // nanosleep
#include <errno.h>      // EINTR, ESRCH, ECHILD
```

## 시그널 번호 (자주 나오는 것)

```
SIGINT  2   Ctrl+C          SIGKILL 9   강제 종료 (못 막음)
SIGALRM 14  alarm 타이머    SIGTERM 15  종료 요청
SIGTSTP     Ctrl+Z          SIGSTOP     정지 (못 막음)
SIGCONT     재개            SIGCHLD     자식 상태 변화
SIGUSR1     앱 정의
```

## 플래그 + 핸들러

```c
static volatile sig_atomic_t stop = 0;
static void handler(int signo) { (void)signo; stop = 1; }
```

## sigaction 등록

```c
struct sigaction sa = {0};
sa.sa_handler = handler;
sigemptyset(&sa.sa_mask);
if (sigaction(SIGTERM, &sa, NULL) == -1) return 1;
```

```c
sa.sa_handler = SIG_IGN;    // 무시
sa.sa_handler = SIG_DFL;    // 기본 동작
```

## 시그널 보내기

```c
if (kill(pid, SIGTERM) == -1 && errno != ESRCH) return -1;
```

```bash
kill -TERM "$pid"
kill -INT "$pid"
kill -9 "$pid"
kill %1 %2
```

## 타이머

```c
alarm(seconds);    // seconds 뒤 SIGALRM
alarm(0);          // 취소
```

## 차단 / 복원

```c
sigset_t set, old;
sigemptyset(&set);
sigaddset(&set, SIGUSR1);
sigprocmask(SIG_BLOCK, &set, &old);
/* 플래그 복사 + 지우기 */
sigprocmask(SIG_SETMASK, &old, NULL);
```

## 기다리지 않고 회수

```c
pid_t r = waitpid(child, &status, WNOHANG);   // child / 0 / -1
pid_t r = waitpid(-1, &status, WNOHANG);      // 아무 자식
```

## 보내고 회수

```c
kill(pid, SIGTERM);
while (waitpid(pid, &status, 0) == -1)
    if (errno != EINTR) return 1;
if (WIFEXITED(status))   /* WEXITSTATUS(status) */;
if (WIFSIGNALED(status)) /* WTERMSIG(status)    */;
```

## 짧게 쉬기

```c
nanosleep(&(struct timespec){0, 100000000}, NULL);   // 0.1초
```

## limit 종료 코드

```
124   시간 초과 (SIGALRM)
130   SIGINT (128 + 2)
127   exec 실패
143   자식이 SIGTERM으로 종료된 status (128 + 15)
그 외 자식 결과 그대로
```

## 터미널 작업 제어

```bash
Ctrl+C          # SIGINT (포그라운드)
Ctrl+Z          # SIGTSTP (포그라운드)
jobs
bg  /  bg %1
fg  /  fg %1
명령 &
echo $!         # 마지막 백그라운드 PID
wait "$pid"; echo $?
```

## 관찰 명령

```bash
ping 127.0.0.1
tail -n 1 -f out/live.log
printf 'ERROR request=/new status=500\n' >> out/live.log
timeout -v 2s tail -n 1 -f data/events.log; echo $?   # 124
./bin/limit 2 tail -n 1 -f data/events.log; echo $?   # 124
```

## 제출

```bash
make check1 && make check2 && make check
git add src/control.c src/monitor.c
git commit -m "Implement signal handling"
git push
```

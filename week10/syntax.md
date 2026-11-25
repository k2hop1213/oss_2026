# 10주차 문법: 프로세스 (fork / exec / wait)

명령어 설명 없이 문법 형태만 빠르게 찾아보는 용도.

## 헤더

```c
#include <unistd.h>     // fork, execvp, _exit, chdir, getpid, getppid
#include <sys/wait.h>   // waitpid, WIFEXITED, WEXITSTATUS, WIFSIGNALED, WTERMSIG
#include <errno.h>      // errno, EINTR
```

## fork

```c
pid_t pid = fork();
if (pid < 0)  { perror("fork"); return -1; }   // 실패
if (pid == 0) { /* 자식 */ }
/* 부모: pid = 자식 PID */
```

## execvp

```c
char *argv[] = {"wc", "-l", "data/sample.csv", NULL};
execvp(argv[0], argv);       // 성공하면 돌아오지 않음
perror(argv[0]);
_exit(127);
```

## waitpid

```c
int status;
waitpid(pid, &status, 0);          // 특정 자식 기다림
waitpid(pid, NULL, 0);             // 상태 버림

while (waitpid(pid, &status, 0) < 0) {
    if (errno == EINTR) continue;
    perror("waitpid");
    return 125;
}
```

## 종료 상태 해석

```c
if (WIFEXITED(status))   return WEXITSTATUS(status);   // 정상 종료 코드
if (WIFSIGNALED(status)) return 128 + WTERMSIG(status); // 시그널 종료
```

## 종료 코드 규칙

```
0          성공
N          자식이 반환한 값 그대로 (예: 7)
127        exec 실패 (명령 없음)
125        runner 자체 오류 (fork/wait 실패)
128 + n    시그널 n으로 종료 (실습 wait.c)
2          사용법 오류
```

## runner 패턴

```c
pid_t pid = fork();
if (pid < 0) { perror("fork"); return 125; }
if (pid == 0) { execvp(argv[0], argv); perror(argv[0]); _exit(127); }
int status;
while (waitpid(pid, &status, 0) < 0) {
    if (errno == EINTR) continue;
    perror("waitpid"); return 125;
}
return WIFEXITED(status) ? WEXITSTATUS(status) : 125;
```

## 셸 builtin (cd)

```c
if (strcmp(argv[0], "cd") == 0) {
    if (chdir(argv[1]) != 0) perror("cd");
    return 1;                  // 처리함 → fork 안 함
}
return 0;                      // 외부 명령 → fork
```

## 입력 파싱

```c
fgets(line, sizeof line, stdin);
char *nl = strchr(line, '\n'); if (nl) *nl = '\0';
int i = 0;
for (char *t = strtok(line, " \t"); t; t = strtok(NULL, " \t")) argv[i++] = t;
argv[i] = NULL;
```

## 셸에서 확인

```bash
./bin/demo ./bin/probe 7; echo $?        # runner: exit=7 / 7
./bin/demo ./missing-command; echo $?    # 127
echo $$          # 현재 셸 PID
ps -o pid,ppid,cmd
```

## Makefile (실습)

```make
ARCH := $(shell uname -m)
bin/shell-start: src/start.c prebuilt/$(ARCH)/libsupport.a
	$(CC) $(CFLAGS) -o $@ $^
```

## 제출

```bash
make check
git add src
git commit -m "Implement command shell"
git push
```

# 5주차 문법: gcc / C 빌드

명령어 설명 없이 문법 형태만 빠르게 찾아보는 용도.

## 4단계

```
전처리(-E) → 컴파일(-S) → 어셈블(-c) → 링크
```

## 분리 컴파일

```bash
gcc -std=c11 -Wall -Wextra -I. -c file.c -o file.o   # 컴파일만 (.o 생성)
gcc a.o b.o -o 실행파일                                # 링크
```

## 선언 vs 정의

```c
// 선언 (.h)
int func(int x);

// 정의 (.c)
int func(int x) {
    return x + 1;
}
```

## 헤더 가드

```c
#ifndef NAME_H
#define NAME_H
...
#endif
```

## nm (심볼 확인)

```
nm file.o
T   정의됨 (Text, 실제 코드 있음)
U   미정의 (Undefined, 다른 곳에서 가져와야 함)
```

## 정적 라이브러리

```bash
ar rcs lib이름.a file1.o file2.o     # 아카이브 생성
ar t lib이름.a                        # 안의 목록 보기
gcc main.o lib이름.a -o 실행파일       # 링크 (호출하는 쪽 먼저, 라이브러리 나중)
```

## 문자열 함수 (logutil 예제)

```c
strncmp(s1, s2, n)     // 앞 n글자 비교, 같으면 0
strlen(s)                // 문자열 길이
strstr(s, sub)           // s 안에 sub가 "어디든" 있으면 위치 반환 (접두사 아님! 주의)
```

## gcc 옵션

```
-E            전처리까지만 (.i)
-S            컴파일까지만 (.s, 어셈블리)
-c            어셈블까지만 (.o), 링크 안 함
-o 이름        출력 파일 이름
-I경로         헤더 찾는 경로 추가        예: -Iinclude
-L경로         라이브러리 찾는 경로 추가
-l이름         라이브러리 링크 (libNAME)   예: -lm
-std=c11      C 표준 지정
-Wall -Wextra 경고 켜기
-g            디버그 정보 포함 (gdb용)
-O0           최적화 끄기 (gdb용)
```

6주차 단계별 예:

```bash
gcc -Iinclude -E src/filters.c -o build/filters.i
gcc -Iinclude -S src/filters.c -o build/filters.s
gcc -Iinclude -c src/filters.c -o build/filters.o
ar rcs lib/libfilters.a build/filters.o
gcc -o bin/image-lab build/main.o build/image_io.o lib/libfilters.a -lm
```

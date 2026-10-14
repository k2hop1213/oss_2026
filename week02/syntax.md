# 2주차 문법: 리눅스 셸

명령어 설명 없이 문법 형태만 빠르게 찾아보는 용도.

## 경로

```
/absolute/path      # 절대 경로 (/ 로 시작)
relative/path        # 상대 경로 (현재 위치 기준)
.                     # 현재 디렉토리
..                    # 부모 디렉토리
~                     # 홈 디렉토리
```

---

## ls

```
ls [옵션] [경로]

-l    긴 형식
-a    숨김 파일 포함
-h    사람이 읽기 쉬운 크기 (-l과 같이)
-t    수정시각순 정렬 (최신 먼저)
-r    정렬 뒤집기
-S    크기순 정렬 (큰 것 먼저)
-R    하위 디렉토리까지 재귀적으로
-d    디렉토리 자체만 (내용물 X)
```

조합 예: `ls -lh`, `ls -la`, `ls -ltr`, `ls -ld 폴더명`

---

## grep

```
grep [옵션] PATTERN [file ...]

(없음)   매치되는 줄 출력
-c       매치된 줄 개수만
-i       대소문자 무시
-n       줄 번호 표시
-v       패턴이 없는 줄만 (반전)
-r       디렉토리 재귀 검색
-l       매치된 파일명만
```

조합 예: `grep -rn FIXME src/`, `grep -c '^ERROR ' file`

---

## cut

```
cut -d"구분자" -f필드번호 file
```

예: `cut -d" " -f1 app.log` → 공백 기준 1번째 필드

---

## sort

```
sort [옵션] file

-n    숫자 기준 정렬
-r    역순
-u    중복 제거하며 정렬
-f    대소문자 무시
-k N,M   N~M번째 필드 기준 정렬
-t"구분자"  필드 구분자 지정
```

조합 예: `sort -n`, `sort -rn`, `sort -k1,1rn -k2,2`

---

## uniq

```
uniq -c file     # 연속 중복 줄을 세서 합침 (sort 먼저 해야 전체 중복 다 잡힘)
```

패턴: `sort file | uniq -c`

---

## wc

```
wc -l file    # 줄 수
wc -c file    # 바이트 수
wc -w file    # 단어 수
```

---

## find

```
find 경로 -name "패턴"      # 이름으로 찾기
find 경로 -type f/d          # 파일(f)/디렉토리(d) 타입으로
find 경로 -name "패턴" -exec 명령어 {} \;   # 찾은 것마다 명령 실행
```

예: `find data -name "*.log"`

---

## 리다이렉션 / 파이프

```
cmd > file      # 출력을 파일에 저장 (덮어쓰기)
cmd >> file     # 출력을 파일에 추가
cmd 2> file     # 에러만 파일로
cmd < file      # 파일을 입력으로 사용
cmd1 | cmd2     # cmd1 출력을 cmd2 입력으로 연결
```

---

## which / type / PATH

```
which 명령어     # 명령어의 실행파일 위치
type 명령어       # 명령어가 뭔지 (내장/별칭/실행파일)
echo $PATH       # 명령어 찾는 디렉토리 목록
```

---

## 하드링크 / 심볼릭 링크

```
ln 원본 링크이름        # 하드링크 (같은 inode)
ln -s 원본 링크이름     # 심볼릭링크 (경로만 저장)
```

---

## 와일드카드

```
*        아무 글자 0개 이상          *.log
?        아무 글자 딱 1개            file?.txt
[abc]    a, b, c 중 딱 1개           file[12].txt
[a-z]    범위 중 딱 1개              [a-z].c
[!0-9]   숫자가 아닌 1개
.*       숨김 파일
"*.log"  따옴표 → 셸이 안 바꿈 (find -name에 필요)
```

## 파일 복사 / 이동 / 삭제

```bash
cp 원본 대상            cp -r 폴더 대상      # 폴더는 -r
mv 원본 대상            # 이동 = 이름 바꾸기
rm 파일                 rm -r 폴더          # 되돌릴 수 없음
mkdir -p a/b/c          # 중간 폴더까지 생성
touch 파일              # 빈 파일 / 수정 시각 갱신
ls -d 폴더              # 폴더 자체 정보
ls -li                  # inode 번호
```

## 도움말

```bash
명령 --help
man 명령
```

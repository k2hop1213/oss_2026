# 3주차 문법: Git

명령어 설명 없이 문법 형태만 빠르게 찾아보는 용도.

## 초기 설정

```
git init -b main
git config user.name "이름"
git config user.email "이메일"
```

## 기본 흐름

```
git status [--short]
git add 파일1 파일2 ...
git diff              # 워킹트리 vs 스테이징
git diff --staged     # 스테이징 vs 마지막 커밋
git commit -m "메시지"
git log --oneline [--graph]
```

## status --short 코드

```
??   추적 안 됨 (untracked)
 M   수정됐지만 스테이징 안 됨
M    스테이징됨
MM   스테이징 후 추가로 또 수정됨
```

## 되돌리기

```
git restore 파일           # 워킹트리 변경 취소 (마지막 커밋으로)
git restore --staged 파일  # 스테이징만 취소 (워킹트리는 유지)
```

## 원격 저장소

```
git remote -v
git remote add origin <URL>
git push -u origin main
git pull --ff-only
```

## 브랜치

- 브랜치 = 커밋을 가리키는 이동 가능한 참조
- merge 커밋은 부모가 2개 (명령어는 수업 범위 밖)

## .gitignore

```
build/
reports/
*.log
```
→ 이미 추적 중인 파일엔 효과 없음. 추적 해제하려면:
```
git rm --cached 파일
```

---

## 확인 / 조회

```bash
git show HEAD:파일               # 현재 커밋의 파일 내용
git show HEAD~1:파일             # 하나 전 커밋
git show --stat HEAD            # 커밋에 들어간 파일
git log -2 --oneline --stat
git log --oneline -- 파일        # 그 파일이 바뀐 커밋만
git diff 커밋 HEAD -- 파일
git ls-files                    # 추적 중인 파일 목록
git status --short --ignored    # 무시된 파일도 표시 (!!)
```

## 추적 / 삭제 / 이름

```bash
git rm 파일                 # 디스크에서도 삭제 + 삭제 스테이징
git rm --cached 파일        # 디스크엔 남기고 추적만 중단
git mv 원래 새이름          # 이름 변경 + 스테이징
git restore --staged 파일   # add 취소
git restore 파일            # 작업 파일을 스테이징 버전으로
```

## 원격과 동기화

```bash
git clone URL [폴더]
git remote -v                 # 연결된 원격 확인
git remote add origin URL
git fetch                     # 받기만 (통합 X)
git pull --ff-only            # fetch + 통합, 갈라졌으면 중단
git push -u origin main       # 처음
git push                      # 이후
```

## GitHub 인증 (gh)

```bash
gh auth status
gh auth login --hostname github.com --git-protocol https --web
gh auth setup-git
```

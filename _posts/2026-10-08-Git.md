---
title: "[Git] Git 기초 - 로컬 저장소에서 GitHub까지"
date: 2026-10-08 10:00:00 +0900
categories:
  - Git
tags:
  - Git
  - GitHub
  - VersionControl
  - Basics
---

# Git 기초 - 로컬 저장소에서 GitHub까지

Git과 GitHub를 처음 사용하면서 진행한 과정을 정리한다.

이번 글에서는 Windows 환경에서 Git을 설치하고, 로컬 Repository를 생성한 뒤 GitHub Repository와 연결하고 파일을 Push하는 과정까지 직접 실습했다.

---

## 1. Git이란?

**Git은 분산 버전 관리 시스템(Distributed Version Control System)**&#xC774;다.

파일의 변경사항을 기록하고 버전별로 관리할 수 있기 때문에 개발 과정에서 코드가 어떻게 변경되었는지 확인할 수 있다.

예를 들어 하나의 프로젝트를 계속 수정하면 다음과 같이 여러 버전을 기록할 수 있다.

```text
버전 1
  ↓
버전 2
  ↓
버전 3
  ↓
현재 버전
```

문제가 발생했을 경우 과거의 변경사항을 확인하거나 이전 버전으로 돌아갈 수 있다는 장점이 있다.

---

## 2. GitHub란?

**GitHub는 Git Repository를 인터넷에 저장하고 관리할 수 있는 플랫폼**이다.

Git은 기본적으로 내 PC에서도 사용할 수 있지만, GitHub를 이용하면 Repository를 원격으로 저장할 수 있다.

```text
내 PC
  │
  │ Git
  ▼
로컬 Repository
  │
  │ git push
  ▼
GitHub
  │
  └── 원격 Repository
```

따라서 PC가 변경되더라도 GitHub에 저장된 Repository를 다시 내려받아 작업을 이어갈 수 있다.

---

## 3. Git과 GitHub의 관계

Git과 GitHub는 같은 것이 아니다.

| 구분     | 설명                                     |
| ------ | -------------------------------------- |
| Git    | 파일의 변경 이력을 관리하는 버전 관리 시스템              |
| GitHub | Git Repository를 원격으로 저장하고 협업할 수 있는 플랫폼 |

쉽게 표현하면:

```text
Git
└── 버전 관리

GitHub
└── Git Repository를 저장하고 공유하는 공간
```

---

## 4. Git 설치 확인

Windows CMD에서 다음 명령어를 실행했다.

```cmd
git --version
```

실행 결과:

```text
git version 2.53.0.windows.2
```

Git이 정상적으로 설치되어 있는 것을 확인했다.

---

## 5. 프로젝트 폴더 생성

블로그 프로젝트를 관리하기 위한 폴더를 생성했다.

```cmd
mkdir kws-blog
cd kws-blog
```

현재 작업 경로:

```text
C:\Users\USER\kws-blog
```

---

## 6. Git Repository 생성

프로젝트 폴더에서 다음 명령어를 실행했다.

```cmd
git init
```

`git init`은 현재 폴더를 Git Repository로 초기화하는 명령어다.

즉,

```text
kws-blog
```

폴더를 Git으로 관리할 수 있는 상태로 만든 것이다.

---

## 7. README.md 생성

첫 번째 파일로 `README.md`를 생성했다.

```cmd
echo # My Blog > README.md
```

파일이 생성되었는지 확인하기 위해:

```cmd
dir
```

명령어를 사용했다.

결과:

```text
README.md
```

파일이 생성된 것을 확인했다.

---

## 8. Git 상태 확인

다음 명령어를 사용하면 현재 Git Repository의 상태를 확인할 수 있다.

```cmd
git status
```

처음 생성한 `README.md`는 아직 Git이 관리 대상으로 등록하지 않았기 때문에 `Untracked files` 상태로 나타났다.

```text
Untracked files:
    README.md
```

### Untracked란?

Git Repository 안에 파일은 존재하지만 아직 Git이 변경사항을 추적하지 않는 상태다.

---

## 9. git add

다음 명령어를 실행했다.

```cmd
git add README.md
```

`git add`는 변경된 파일을 다음 Commit에 포함할 대상으로 지정하는 과정이다.

흐름으로 보면:

```text
파일 생성
   ↓
Untracked
   ↓
git add
   ↓
Commit 준비
```

다시 상태를 확인하면:

```cmd
git status
```

다음과 같이 나타난다.

```text
Changes to be committed:
    new file: README.md
```

---

## 10. Git 사용자 정보 설정

처음 Commit을 시도했을 때 다음과 같은 메시지가 발생했다.

```text
Author identity unknown
Please tell me who you are.
```

Git은 Commit을 생성할 때 누가 작업했는지 기록하기 때문에 사용자 이름과 이메일 설정이 필요하다.

다음 명령어를 사용했다.

```cmd
git config --global user.name "김우석"
git config --global user.email "GitHub에 사용하는 이메일"
```

설정된 값을 확인할 수도 있다.

```cmd
git config --global user.name
git config --global user.email
```

---

## 11. 첫 번째 Commit

사용자 정보를 설정한 후 첫 번째 Commit을 생성했다.

```cmd
git commit -m "첫 번째 커밋"
```

Commit 결과:

```text
[master (root-commit) ec55859] 첫 번째 커밋
1 file changed, 1 insertion(+)
create mode 100644 README.md
```

### Commit이란?

Commit은 현재 변경사항을 하나의 버전으로 기록하는 과정이다.

예를 들어:

```text
Commit 1
└── README.md 생성

Commit 2
└── README.md 수정

Commit 3
└── Git 문서 추가
```

와 같이 프로젝트의 변경 이력을 관리할 수 있다.

---

## 12. GitHub Repository 생성

GitHub에 `kws-blog`라는 Repository를 생성했다.

Repository:

```text
kws9507-ops/kws-blog
```

GitHub Repository는 로컬 Repository와 별도로 존재한다.

```text
로컬 Repository

C:\Users\USER\kws-blog


원격 Repository

GitHub
└── kws9507-ops/kws-blog
```

---

## 13. 로컬 Repository와 GitHub 연결

다음 명령어로 GitHub Repository를 원격 저장소로 등록했다.

```cmd
git remote add origin https://github.com/kws9507-ops/kws-blog.git
```

여기서 `origin`은 GitHub Repository를 가리키는 이름이다.

연결 상태는 다음 명령어로 확인할 수 있다.

```cmd
git remote -v
```

결과:

```text
origin  https://github.com/kws9507-ops/kws-blog.git (fetch)
origin  https://github.com/kws9507-ops/kws-blog.git (push)
```

---

## 14. main Branch 사용

처음 Git Repository를 생성했을 때 기본 Branch가 `master`였다.

GitHub에서 일반적으로 사용하는 `main` Branch를 사용하기 위해 이름을 변경했다.

```cmd
git branch -M main
```

결과적으로:

```text
master
   ↓
main
```

으로 변경했다.

---

## 15. GitHub에 첫 번째 Push

이제 로컬 Repository의 Commit을 GitHub에 업로드했다.

```cmd
git push -u origin main
```

실행 결과:

```text
To https://github.com/kws9507-ops/kws-blog.git
 * [new branch]      main -> main
branch 'main' set up to track 'origin/main'.
```

### git push

`git push`는 로컬 Repository의 Commit을 원격 Repository인 GitHub로 업로드하는 명령어다.

```text
로컬 PC
   │
   │ git push
   ▼
GitHub
```

---

## 16. README.md 수정

이후 `README.md`의 내용을 수정했다.

수정 후 Git의 상태를 확인했다.

```cmd
git status
```

그리고 실제 변경 내용을 확인하기 위해:

```cmd
git diff
```

명령어를 사용했다.

### git diff

`git diff`는 Commit되지 않은 변경사항을 확인할 때 사용하는 명령어다.

즉:

```text
현재 파일
   ↓
마지막 Commit
   ↓
무엇이 달라졌는지 확인
```

하는 역할이다.

---

## 17. 수정사항 Commit

수정된 README.md를 다시 Git에 추가했다.

```cmd
git add README.md
```

그리고 Commit을 생성했다.

```cmd
git commit -m "README 수정"
```

실행 결과:

```text
[main 53e7a0d] README 수정
1 file changed, 4 insertions(+), 1 deletion(-)
```

이제 Git에는 두 개의 Commit이 기록되어 있다.

```text
ec55859  첫 번째 커밋
    ↓
53e7a0d  README 수정
```

---

## 18. 수정사항을 GitHub에 Push

마지막으로 수정된 Commit을 GitHub에 올렸다.

```cmd
git push
```

실행 결과:

```text
ec55859..53e7a0d  main -> main
```

이제 로컬 Repository와 GitHub Repository의 내용이 동일해졌다.

```text
내 PC
   │
   │ git push
   ▼
GitHub

main
└── 53e7a0d README 수정
```

---

# 19. 지금까지 배운 Git 명령어

| 명령어                  | 설명                  |
| -------------------- | ------------------- |
| `git --version`      | Git 설치 및 버전 확인      |
| `git init`           | Git Repository 생성   |
| `git status`         | 현재 Repository 상태 확인 |
| `git add`            | Commit 대상 지정        |
| `git commit`         | 변경사항을 버전으로 기록       |
| `git log`            | Commit 이력 확인        |
| `git diff`           | 변경사항 확인             |
| `git remote add`     | 원격 Repository 연결    |
| `git remote -v`      | 원격 Repository 확인    |
| `git branch -M main` | Branch 이름 변경        |
| `git push`           | GitHub에 Commit 업로드  |

---

# 20. 전체 Git 작업 흐름

이번 실습에서 가장 중요한 부분이다.

```text
                Git
                 │
                 ▼
        ┌─────────────────┐
        │ 파일 수정/생성   │
        └────────┬────────┘
                 │
                 ▼
           git status
                 │
                 ▼
             git add
                 │
                 ▼
            git commit
                 │
                 ▼
             git push
                 │
                 ▼
        ┌─────────────────┐
        │     GitHub      │
        │  Remote Repo    │
        └─────────────────┘
```

앞으로 Git을 사용할 때 기본적으로 이 흐름을 반복하게 된다.

```text
수정
 ↓
status
 ↓
add
 ↓
commit
 ↓
push
```

---

# 21. Git과 GitHub를 사용하는 이유

Git을 사용하면 프로젝트의 변경 이력을 관리할 수 있다.

GitHub를 함께 사용하면 Repository를 원격으로 저장할 수 있기 때문에 PC가 변경되더라도 작업을 이어갈 수 있다.

예를 들어 새로운 PC에서:

```cmd
git clone https://github.com/kws9507-ops/kws-blog.git
```

명령어를 사용하면 GitHub에 저장된 Repository를 새로운 PC로 가져올 수 있다.

따라서:

```text
PC 1
  │
  │ git push
  ▼
GitHub
  ▲
  │ git clone / git pull
  │
PC 2
```

와 같은 방식으로 여러 PC에서 작업을 이어갈 수 있다.

---

# 마무리

이번 실습에서는 Git을 처음 사용하는 것부터 시작해서

* Git 설치 확인
* Repository 생성
* `git init`
* 파일 생성
* `git status`
* `git add`
* `git commit`
* GitHub Repository 생성
* `git remote add`
* Branch 설정
* `git push`
* 파일 수정
* `git diff`

까지 직접 실습했다.

다음 단계에서는 **Markdown을 이용해 기술 문서를 작성하고, Jekyll을 이용해 이 Markdown 파일을 실제 웹페이지로 변환하는 과정**을 진행할 예정이다.

최종적으로는 GitHub Pages를 이용해 이 Repository를 실제 개인 기술 블로그로 만들어보는 것을 목표로 한다.

# PARA 볼트 구조 (빠른 색인)

터미널에서 볼트 루트(`obsidian_note`) 기준으로 폴더만 훑을 때 이 파일과 아래 표를 기준으로 삼으면 됩니다.

## PARA 네 축

| 폴더 | Tiago Forte PARA | 이 볼트에서의 역할 |
|------|------------------|-------------------|
| `1-Projects` | Projects | 마감·산출물이 있는 진행 중 작업 (취업 준비, 사이드 프로젝트 등) |
| `2-Areas` | Areas | 기한 없이 유지하는 책임·관심 (투자, 성장, 일기, 노트장) |
| `3-Resources` | Resources | 주제별 참고 자료·학습 클립 (React, 블록체인, Blog 등) |
| `4-Archives` | Archives | 완료·휴면 자료 보관 |

## 루트 기타 폴더·파일

| 경로 | 용도 |
|------|------|
| `task` | Periodic Notes 일·주·월 노트 (플러그인 설정과 동일) |
| `template` | Templater 템플릿 |
| `Excalidraw` | Excalidraw 플러그인 기본 폴더 (경로 변경 금지) |
| `images` | 첨부·스크린샷 등 |
| `🧠 디지털 두뇌.md` | 전체 허브 (그래프·워크플로우) |

## 허브 노트로 들어가기

- [[🧠 디지털 두뇌]]
- [[1-Projects/🚀 프로젝트 허브]]
- [[2-Areas/🌱 개인 성장 영역]]
- [[3-Resources/📚 학습 자료실]]
- [[4-Archives/📁 아카이브 보관소]]
- [[task/📅 일일 작업 허브]]
- [[template/📝 템플릿 허브]]
- [[Excalidraw/🎨 시각화 허브]]
- [[images/📸 이미지 허브]]

## PowerShell에서 구조 보기

```powershell
Set-Location 'C:\Users\21nko\obsidian_note'
Get-ChildItem -Directory | Sort-Object Name | Format-Table Name
Get-ChildItem -Path '1-Projects','2-Areas','3-Resources','4-Archives' -Recurse -File -Filter '*.md' | Select-Object FullName
```

---

#PARA #허브 #구조

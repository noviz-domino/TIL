# 📚 Notion to GitHub TIL (Today I Learned)
## 멀티캠퍼스 A.IM 1회차 TIL

> 노션(Notion)에 작성한 학습 기록을 깃허브(GitHub)에 자동으로 동기화되게 구축되었습니다. 🚀

---

## 🛠️ Tech Stack & Architecture

- **Source**: Notion Database (Published check & Date properties)
- **Automation**: GitHub Actions (Daily Cron & Manual Workflow Dispatch)
- **Script**: Python (Notion API Parser & Markdown Converter)
- **Destination**: GitHub Repository (`/TIL` directory)

---

## ✨ Key Features

1. **자동 동기화 (Automated Sync)**
   - 매일 자정(한국 시간 기준)에 GitHub Actions가 실행되어 노션의 최신 글을 자동으로 가져옵니다.
   - 필요시 깃허브 Actions 탭에서 **`Run workflow`**를 통해 수동으로 즉시 동기화할 수 있습니다.
   - 다만 노드js 버전처리 오류가 발생하는데 해당부분은 깃헙 자체 버전문제로 보여 신경쓰지 않아도 될듯합니다.
  
2. **완벽한 본문 마크다운 변환**
   - 제목, 단락, 글머리/번호 기호 목록, 투두리스트(`[ ]`)
   - 토글 메뉴 및 들여쓰기된 하위 블록(`Child blocks`) 구조 보존
   - 코드 블록(Language Highlighting) 및 콜아웃(Callout) 지원
  
3. **날짜 기반 파일 정렬**
   - 노션의 `Date(날짜)` 속성(또는 생성일)을 자동으로 파싱하여 파일명 앞에 `YYYY-MM-DD_제목.md` 형태로 접두사를 붙입니다.
   - 깃허브 폴더에서 날짜순으로 깔끔하게 정렬됩니다.

---

## 📂 Repository Structure

```text
├── .github/
│   └── workflows/
│       └── notion-sync.yml   # GitHub Actions 워크플로우 설정
├── TIL/
│   ├── YYYY-MM-DD_제목1.md   # 노션에서 동기화된 TIL 파일들
│   └── YYYY-MM-DD_제목2.md
├── sync.py                   # 노션 API 연동 및 마크다운 변환 파이썬 스크립트
└── README.md
```


## 노션 구조
<img width="1608" height="593" alt="image" src="https://github.com/user-attachments/assets/9ac7cba5-d40e-4740-8aa5-dd69574d18fc" />

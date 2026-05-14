
## Symlink

讓 claude 讀取的 skill 指向你的目錄

```bash
# （管理員)
New-Item -ItemType SymbolicLink -Path "C:\Users\ariel\.claude\skills" -Target "C:\Users\ariel\OneDrive\桌面\Practice\claude-skills"
```

## Git Repo

```bash
git init
git add .
git commit -m "init: add claude skills"

```

## Device

```bash
# 1. Clone repo
git clone https://github.com/你的帳號/claude-skills.git "想放的路徑\claude-skills"

# 2. 建 symlink（管理員)
New-Item -ItemType SymbolicLink -Path "C:\Users\你的帳號\.claude\skills" -Target "想放的路徑\claude-skills"
```

## Symlink

讓 claude 讀取的 skill 指向你的目錄
```bash
New-Item -ItemType SymbolicLink -Path "C:\Users\ariel\.claude\skills" -Target "C:\Users\ariel\OneDrive\桌面\Practice\claude-skills"
```

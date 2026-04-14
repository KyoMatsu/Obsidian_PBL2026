### uv（uvx）の再インストール

以下を1行ずつpowershellで実行

```powershell
uv cache clean
Remove-Item -Recurse -Force "$env:USERPROFILE\.local\bin\uv.exe", "$env:USERPROFILE\.local\bin\uvx.exe" -ErrorAction SilentlyContinue
Remove-Item -Recurse -Force "$env:USERPROFILE\.local\share\uv" -ErrorAction SilentlyContinue
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```
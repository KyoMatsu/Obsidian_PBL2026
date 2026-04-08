# Obsidian × GitHub 連携手順

作成日: 2026-04-08

---

## 概要

ObsidianのVaultをGitHubリポジトリで管理することで、バージョン管理・バックアップ・複数デバイス間の同期が可能になる。

---

## 前提条件

- Gitがローカルにインストール済み
- GitHubアカウント取得済み
- Obsidianインストール済み

---

## 手順

### 1. GitHubリポジトリの作成

1. GitHub（https://github.com）にログイン
2. 右上の「+」→「New repository」を選択
3. 以下を設定して「Create repository」
   - Repository name: 任意（例: `obsidian-vault`）
   - Visibility: **Private** を推奨（ノートの内容が公開されないよう）
   - README: 追加しない（後でVaultと競合する可能性あり）

---

### 2. ローカルVaultをGitリポジトリとして初期化

ターミナルでObsidianのVaultフォルダに移動して実行する。

```bash
cd /path/to/your/vault

git init
git remote add origin https://github.com/<ユーザー名>/<リポジトリ名>.git
```

---

### 3. .gitignoreの設定

Vault内の不要ファイル（キャッシュ・プラグインの設定など）をGit管理から除外する。

```bash
# .gitignoreファイルを作成
cat > .gitignore << 'EOF'
.obsidian/workspace
.obsidian/workspace.json
.obsidian/workspaces.json
.obsidian/plugins/
.trash/
.DS_Store
EOF
```

> **注意**: `.obsidian/` フォルダ全体を除外するか、テーマ・プラグイン設定ごと管理するかは運用方針による。  
> 複数デバイスで設定も共有したい場合は `.obsidian/` ごと管理する選択肢もある。

---

### 4. 初回コミット＆プッシュ

```bash
git add .
git commit -m "initial commit"
git branch -M main
git push -u origin main
```

---

### 5. Obsidian Git プラグインの導入（推奨）

CLIを使わずObsidian内でGit操作を行えるコミュニティプラグイン。

#### インストール手順

1. Obsidian → 設定 → コミュニティプラグイン → 「閲覧」
2. 「Obsidian Git」を検索してインストール
3. 有効化

#### 主な設定（設定 → Obsidian Git）

| 設定項目 | 推奨値 | 説明 |
|---|---|---|
| Vault backup interval | 10〜30（分） | 自動コミット間隔 |
| Auto pull interval | 10〜30（分） | 自動プル間隔 |
| Commit message | `vault backup: {{date}}` | コミットメッセージのテンプレート |
| Pull updates on startup | ON | 起動時に最新を取得 |

---

### 6. SSH認証の設定（オプション・推奨）

HTTPSではなくSSHを使うと、毎回の認証入力が不要になる。

```bash
# SSHキー生成（既存のキーがある場合は不要）
ssh-keygen -t ed25519 -C "your_email@example.com"

# 公開鍵を表示してGitHubに登録
cat ~/.ssh/id_ed25519.pub
```

GitHubの Settings → SSH and GPG keys → New SSH key に貼り付けて登録。

リモートURLをSSHに変更:
```bash
git remote set-url origin git@github.com:<ユーザー名>/<リポジトリ名>.git
```

---

### 7. モバイル（iOS/Android）での利用

ObsidianのモバイルアプリでもObsidian Gitプラグインは利用可能だが、**iOSではGitの動作に制限がある**ため、以下の代替手段も考慮する。

- **Working Copy**（iOS）: GitクライアントアプリとObsidianをiCloudやショートカット経由で連携
- **MGit**（Android）: AndroidのGitクライアント
- **iCloud / Dropbox 経由の間接同期**: Gitではなくクラウドストレージ経由でVaultを共有する方法

> モバイルでのGit連携は環境依存の問題が多く、安定性に差があることに注意。

---

## 注意事項

- **コンフリクト**: 複数デバイスで同時編集するとマージコンフリクトが発生しうる。自動コミット機能を使う際は注意。
- **センシティブ情報**: パスワードや個人情報をノートに含む場合、Privateリポジトリであっても過信しないこと。
- **大容量ファイル**: 画像・動画など大きなファイルを多用する場合はGit LFSの検討が必要。

---

## 参考

- [Obsidian Git プラグイン（GitHub）](https://github.com/denolehov/obsidian-git)
- [GitHub SSH設定ドキュメント](https://docs.github.com/ja/authentication/connecting-to-github-with-ssh)

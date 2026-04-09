# Obsidian × GitHub 連携手順

作成日: 2026-04-09  
更新日: 2026-04-09

---

## 概要

ObsidianのVaultをGitHubリポジトリで管理することで、バージョン管理・バックアップ・複数デバイス間の同期が可能になる。  
本手順では **Obsidian Git プラグインによる自動同期設定まで** を対象とする。

---

## 前提条件

- Gitがローカルにインストール済み（`git --version` で確認）
- GitHubアカウント取得済み
- Obsidianインストール済み

---

## STEP 1: GitHubリポジトリの作成

1. [GitHub](https://github.com) にログイン
2. 右上の「+」→「New repository」を選択
3. 以下を設定して「Create repository」をクリック

| 項目 | 設定値 | 備考 |
|---|---|---|
| Repository name | 任意（例: `obsidian-vault`） | |
| Visibility | **Private** | ノート内容の公開防止のため強く推奨 |
| Initialize with README | **チェックしない** | Vaultと競合するため |
| .gitignore / License | **追加しない** | 同上 |

---

## STEP 2: SSH認証の設定（推奨）

HTTPSよりSSHが推奨。Obsidian Gitプラグインはパスワード入力UIを持たないため、**HTTPSの場合は認証エラーになるケースが多い**。

### 2-1. SSHキーの生成

```bash
# 既存のキーを確認（あれば新規生成不要）
ls ~/.ssh/id_ed25519.pub

# キーがなければ生成
ssh-keygen -t ed25519 -C "your_email@example.com"
# パスフレーズは空にするか設定する（空推奨: プラグインの自動実行と相性が良い）
```

### 2-2. 公開鍵をGitHubに登録

```bash
# 公開鍵を表示
cat ~/.ssh/id_ed25519.pub
```

1. GitHub → 右上アバター → **Settings**
2. 左メニュー → **SSH and GPG keys**
3. **New SSH key** をクリック
4. Title: 任意（例: `MacBook`）、Key: 上記で表示した内容を貼り付け → **Add SSH key**

### 2-3. 接続確認

```bash
ssh -T git@github.com
# "Hi <username>! You've successfully authenticated..." と表示されればOK
```

---

## STEP 3: ローカルVaultをGitリポジトリとして初期化

```bash
# VaultのフォルダへCD（実際のパスに置き換える）
cd /path/to/your/vault

# Git初期化
git init

# リモートをSSHで登録
git remote add origin git@github.com:<ユーザー名>/<リポジトリ名>.git
```

---

## STEP 4: .gitignoreの設定

```bash
cat > .gitignore << 'EOF'
# Obsidianのワークスペースキャッシュ（デバイスごとに異なるため除外）
.obsidian/workspace
.obsidian/workspace.json
.obsidian/workspaces.json

# ゴミ箱
.trash/

# macOS
.DS_Store

# Windowsサムネイルキャッシュ
Thumbs.db
EOF
```

> **補足**: `.obsidian/plugins/` はコメントアウトしている。  
> プラグイン設定を複数デバイスで共有したい場合は除外不要。共有不要なら追記して除外する。

---

## STEP 5: 初回コミット＆プッシュ

```bash
git add .
git commit -m "initial commit"
git branch -M main
git push -u origin main
```

GitHubのリポジトリページでファイルが表示されれば成功。

---

## STEP 6: Obsidian Git プラグインのインストール

1. Obsidian → **設定（歯車アイコン）**
2. 左メニュー → **コミュニティプラグイン**
3. 「制限モードをオフにする」→ **オフにする** をクリック
4. **閲覧** ボタンをクリック
5. 検索欄に「`Obsidian Git`」と入力
6. 表示された「Obsidian Git」の **インストール** をクリック
7. インストール後 **有効化** をクリック

---

## STEP 7: Obsidian Git プラグインの自動同期設定

設定 → **Obsidian Git** を開き、以下を設定する。

### 自動バックアップ（コミット＆プッシュ）

| 設定項目 | 推奨値 | 説明 |
|---|---|---|
| Vault backup interval (minutes) | `10` ～ `30` | X分ごとに自動コミット＆プッシュ。`0` で無効 |
| Auto backup after file change | **ON** | ファイル変更後に自動バックアップをトリガー |
| Auto backup after latest commit | **OFF** | 最後のコミットからの経過時間でカウント（好みで） |
| Commit message on auto backup | `vault backup: {{date}}` | `{{date}}` は自動で日時に置換される |

### 自動プル（Pull）

| 設定項目 | 推奨値 | 説明 |
|---|---|---|
| Auto pull interval (minutes) | `10` ～ `30` | X分ごとに自動プル。`0` で無効 |
| Pull updates on startup | **ON** | Obsidian起動時に最新をプル |

### プッシュ・プル挙動

| 設定項目 | 推奨値 | 説明 |
|---|---|---|
| Push on backup | **ON** | バックアップ時にpushまで行う（これをOFFにするとcommitのみ） |
| Pull before push | **ON** | push前にpullしてコンフリクトを減らす |
| Rebase on pull | **OFF** ※ | コンフリクト時の挙動。慣れていなければOFF（merge） |

### 通知・表示

| 設定項目 | 推奨値 | 説明 |
|---|---|---|
| Show status bar | **ON** | 画面右下にGit状態を表示。同期確認に便利 |
| Show notifications | **ON** | コミット・プル完了時に通知を表示 |

---

## STEP 8: 動作確認

1. Obsidianで適当なノートを編集・保存
2. 設定したインターバルが経過するか、コマンドパレット（`Ctrl/Cmd + P`）から `Obsidian Git: Create backup` を実行
3. GitHubリポジトリをブラウザでリロードし、変更が反映されているか確認

### コマンドパレットから使える主なコマンド

| コマンド | 説明 |
|---|---|
| `Obsidian Git: Create backup` | 手動でコミット＆プッシュ |
| `Obsidian Git: Pull` | 手動でプル |
| `Obsidian Git: Open source control view` | 変更ファイルの一覧を表示 |
| `Obsidian Git: Open history view` | コミット履歴を表示 |

---

## 注意事項

- **コンフリクト**: 複数デバイスで同時編集するとマージコンフリクトが発生しうる。発生した場合はCLIでの手動解決が必要になる。
- **SSHパスフレーズ**: パスフレーズを設定したSSHキーを使う場合、プラグインが自動実行できないことがある。ssh-agentの設定が必要。
- **センシティブ情報**: Privateリポジトリであっても、パスワード等の機密情報はノートに書かないことを推奨。
- **大容量ファイル**: 画像・動画を多用する場合はGit LFSの検討が必要。GitHubの無料プランはLFS容量1GBまで。
- **モバイル（iOS）**: iOSのObsidian GitはGitの動作制限があり不安定。Working Copyとの連携が現実的。

---

## 参考リンク

- [Obsidian Git プラグイン（GitHub）](https://github.com/denolehov/obsidian-git)
- [GitHub SSH設定ドキュメント（日本語）](https://docs.github.com/ja/authentication/connecting-to-github-with-ssh)

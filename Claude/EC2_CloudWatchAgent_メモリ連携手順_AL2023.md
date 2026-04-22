# EC2 (Amazon Linux 2023) に CloudWatch Agent をインストールしてメモリ使用量を連携する手順

## 前提条件

- OS: Amazon Linux 2023 (AL2023)
- 認証方式: EC2インスタンスプロファイル (IAMロール)
- 設定ファイル: JSON を直接記述
- EC2からCloudWatchエンドポイントへの**アウトバウンド通信が可能**であること (インターネット経由、またはVPCエンドポイント経由)
- ※メモリやディスクは「カスタムメトリクス」として課金対象になる点に留意

---

## 手順全体像

1. IAMロールを作成し、EC2にアタッチする
	→ **※誰かが追加してくれているため、操作不要。**
2. CloudWatch Agent パッケージを `dnf` でインストールする
3. 設定ファイル (JSON) を作成する
4. 設定を読み込んでエージェントを起動する
5. CloudWatch コンソールでメトリクスを確認する

---

## 1. IAMロールの作成とアタッチ

### 1-1. IAMロールを作成

- マネジメントコンソール → IAM → 「ロール」 → 「ロールを作成」
- 信頼されたエンティティ: **AWSのサービス** → **EC2**
- アタッチするポリシー: **`CloudWatchAgentServerPolicy`** (AWS管理ポリシー)
  - エージェントがメトリクス/ログを送信するために必要な権限が揃っている
  - SSMパラメータストアから設定を取得する場合は別途 `AmazonSSMManagedInstanceCore` などを追加
- ロール名の例: `EC2-CloudWatchAgent-Role`

### 1-2. EC2にアタッチ

- EC2コンソール → 対象インスタンスを選択
- 「アクション」 → 「セキュリティ」 → 「IAMロールを変更」
- 作成したロールを選択して更新

> **注意**: 既存のインスタンスプロファイルを置き換えると、元々付与していた権限が失われる可能性がある。既存ロールがあれば、そのロールに `CloudWatchAgentServerPolicy` を追加する方が安全。

---

## 2. CloudWatch Agent のインストール

Amazon Linux 2023 では、CloudWatch Agent が公式リポジトリで配布されている。

```bash
# リポジトリ情報を更新
sudo dnf update -y

# CloudWatch Agent をインストール
sudo dnf install -y amazon-cloudwatch-agent
```

インストール後の主要ディレクトリ:

| パス | 用途 |
| --- | --- |
| `/opt/aws/amazon-cloudwatch-agent/bin/` | 実行バイナリ・制御スクリプト |
| `/opt/aws/amazon-cloudwatch-agent/etc/` | 設定ファイル配置場所 |
| `/opt/aws/amazon-cloudwatch-agent/logs/` | エージェント自身のログ |

---

## 3. 設定ファイル (JSON) の作成

既定の読み込み対象パスに配置する。

```bash
sudo vi /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json
```

### 3-1. メモリ使用率のみを取得する最小構成

```json
{
  "agent": {
    "metrics_collection_interval": 60,
    "run_as_user": "root"
  },
  "metrics": {
    "namespace": "CWAgent",
    "append_dimensions": {
      "InstanceId": "${aws:InstanceId}"
    },
    "metrics_collected": {
      "mem": {
        "measurement": [
          "mem_used_percent"
        ],
        "metrics_collection_interval": 60
      }
    }
  }
}
```

### 3-2. メモリ + スワップ + ディスクまで広げる構成例

```json
{
  "agent": {
    "metrics_collection_interval": 60,
    "run_as_user": "root"
  },
  "metrics": {
    "namespace": "CWAgent",
    "append_dimensions": {
      "InstanceId": "${aws:InstanceId}",
      "ImageId": "${aws:ImageId}",
      "InstanceType": "${aws:InstanceType}"
    },
    "metrics_collected": {
      "mem": {
        "measurement": [
          "mem_used_percent",
          "mem_available_percent",
          "mem_total",
          "mem_used"
        ],
        "metrics_collection_interval": 60
      },
      "swap": {
        "measurement": [
          "swap_used_percent"
        ],
        "metrics_collection_interval": 60
      },
      "disk": {
        "measurement": [
          "used_percent"
        ],
        "resources": [
          "*"
        ],
        "ignore_file_system_types": [
          "sysfs",
          "devtmpfs",
          "tmpfs"
        ],
        "metrics_collection_interval": 60
      }
    }
  }
}
```

### ポイント

- `metrics_collection_interval`: 収集間隔(秒)。60秒未満にすると高解像度メトリクス扱いで追加コスト。
- `namespace`: CloudWatch上の名前空間。デフォルトで `CWAgent`。
- `append_dimensions` の `${aws:InstanceId}` 等は、EC2メタデータから自動で埋まる予約変数。
- `run_as_user`: ディスクや一部ファイルを読むためrootを指定するのが無難。

---

## 4. エージェントの起動

設定ファイルを読み込ませ、サービスとして起動する。

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config \
  -m ec2 \
  -c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json \
  -s
```

| オプション | 意味 |
| --- | --- |
| `-a fetch-config` | 設定を取り込む |
| `-m ec2` | EC2モード |
| `-c file:<path>` | ローカルファイルから読み込み (SSMパラメータの場合は `ssm:<name>`) |
| `-s` | 読み込み後にサービスを起動 |

### 状態確認

```bash
# ステータス
sudo systemctl status amazon-cloudwatch-agent

# または専用コマンド
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a status

# エージェント自身のログ
sudo tail -f /opt/aws/amazon-cloudwatch-agent/logs/amazon-cloudwatch-agent.log
```

### 自動起動の有効化

インストール時点で有効化されていることが多いが、念のため。

```bash
sudo systemctl enable amazon-cloudwatch-agent
```

---

## 5. CloudWatchコンソールでの確認

1. CloudWatch コンソール → 左メニュー「メトリクス」 → 「すべてのメトリクス」
2. 「カスタム名前空間」に **`CWAgent`** が表示される
3. `CWAgent` → `InstanceId` などのディメンション
4. `mem_used_percent` を選択するとグラフ表示される

> 反映まで数十秒〜数分かかることがある。表示されない場合はエージェントログとIAM権限を確認。

---

## トラブルシューティング

| 症状 | 確認ポイント |
| --- | --- |
| `CWAgent` が名前空間に出てこない | IAMロール(`CloudWatchAgentServerPolicy`)のアタッチ漏れ / アウトバウンド通信 / リージョン設定 |
| エージェントは起動するがメトリクスが来ない | `amazon-cloudwatch-agent.log` で `AccessDenied` や `NoCredentialProviders` を確認 |
| プロキシ経由で送る必要がある | `/opt/aws/amazon-cloudwatch-agent/etc/common-config.toml` でプロキシ/リージョンを設定 |
| ディスクメトリクスに不要なFSが出る | `ignore_file_system_types` や `drop_device` で除外 |

---

## 補足: JSON直書き以外の選択肢との比較

| 方式 | メリット | デメリット |
| --- | --- | --- |
| **JSON直書き (本手順)** | シンプル、Gitで管理しやすい、CI/CD向き | 台数が増えると配布が面倒 |
| ウィザード (`amazon-cloudwatch-agent-config-wizard`) | 対話形式で初心者向け | 差分管理しづらい |
| SSMパラメータストア | 複数台への一元配信が可能 | SSM側の権限・運用設計が別途必要 |

---

## 参考(裏付け元)

- AWS公式: [Install and run the CloudWatch agent on your servers](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/install-CloudWatch-Agent-commandline-fleet.html)
- AWS公式: [Metrics collected by the CloudWatch agent](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/metrics-collected-by-CloudWatch-agent.html)
- AWS re:Post: [Send memory metrics from EC2 instance to CloudWatch](https://repost.aws/knowledge-center/cloudwatch-memory-metrics-ec2)

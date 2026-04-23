# AWS構築演習: ALB + ASG + プライベートEC2 (自動復旧構成)

## 演習概要

| 項目 | 内容 |
|---|---|
| 参加者 | 24名(8名ずつ3グループ) |
| アカウント構成 | 単一AWSアカウント共有 |
| 操作方法 | マネジメントコンソール |
| 学習目標 | ALB経由でプライベートEC2にアクセス、ASGのヘルスチェック連動による自動復旧の理解 |
| 想定リージョン | ap-northeast-1(東京) |

## 全体構成図(1グループ分)

```
                    Internet
                       │
                  ┌────┴────┐
                  │   IGW   │
                  └────┬────┘
                       │
      ┌────────────────┴────────────────┐
      │           VPC (10.x.0.0/16)     │
      │                                 │
      │  ┌──────────────┐  ┌──────────┐ │
      │  │ Public-a     │  │Public-c  │ │
      │  │  - ALB       │  │  - ALB   │ │
      │  │  - NAT GW    │  │          │ │
      │  └──────┬───────┘  └────┬─────┘ │
      │         │               │       │
      │  ┌──────┴───────┐  ┌────┴─────┐ │
      │  │ Private-a    │  │Private-c │ │
      │  │  - EC2(ASG)  │  │          │ │
      │  └──────────────┘  └──────────┘ │
      └─────────────────────────────────┘
```

- NAT Gatewayは AZ-a のみに1個配置(コスト優先)
- ASGはマルチAZサブネット指定だが、desired=1 なので実稼働は1AZ
- AZ-a障害時にAZ-cで起動する動作確認も可能

## 事前に講師側で整備しておくもの

### 1. VPC・サブネット・NAT Gatewayは事前作成

受講者の学習フォーカスは「ALB + ASG + EC2」なので、ネットワーク土台は完成状態で提供することを推奨。

### 2. CIDR設計(3VPC)

| VPC | グループ | CIDR | Public-a | Public-c | Private-a | Private-c |
|---|---|---|---|---|---|---|
| vpc-group1 | Group1 | 10.0.0.0/16 | 10.0.1.0/24 | 10.0.2.0/24 | 10.0.11.0/24 | 10.0.12.0/24 |
| vpc-group2 | Group2 | 10.1.0.0/16 | 10.1.1.0/24 | 10.1.2.0/24 | 10.1.11.0/24 | 10.1.12.0/24 |
| vpc-group3 | Group3 | 10.2.0.0/16 | 10.2.1.0/24 | 10.2.2.0/24 | 10.2.11.0/24 | 10.2.12.0/24 |

### 3. ルートテーブル

- **Public用**: 0.0.0.0/0 → IGW
- **Private用**: 0.0.0.0/0 → NAT Gateway

### 4. IAMロール(EC2用)事前作成

名前: `EC2-SSM-Role`(3グループ共通で1個でOK)

- 信頼ポリシー: ec2.amazonaws.com
- アタッチするマネージドポリシー: `AmazonSSMManagedInstanceCore`

### 5. EIPクォータ確認

**重要**: デフォルトでは1アカウントあたりEIP 5個までの制限があります。
- NAT Gateway 3個 = EIP 3個消費
- 受講者がEIPを触る演習があれば追加で必要
- 事前にサービスクォータで緩和申請(10個程度)を推奨

---

## 受講者向け命名規則(必須)

**24名が同一アカウントで作業するため、命名規則を徹底しないとリソースが識別できません。**

全リソースに個人識別prefixを付与します。

例: 受講者番号が `user01` の場合

| リソース | 命名例 |
|---|---|
| Launch Template | `user01-lt` |
| Target Group | `user01-tg` |
| ALB | `user01-alb` |
| Auto Scaling Group | `user01-asg` |
| Security Group (ALB用) | `user01-sg-alb` |
| Security Group (EC2用) | `user01-sg-ec2` |

**Name タグも同様にprefixを付けること。**

---

## 受講者の作業手順

### ステップ1: セキュリティグループの作成

#### 1-1. ALB用セキュリティグループ

- 名前: `user01-sg-alb`
- VPC: 自グループのVPC
- インバウンドルール:
  - HTTP (80) / Source: 0.0.0.0/0
- アウトバウンドルール: デフォルト(すべて許可)

#### 1-2. EC2用セキュリティグループ

- 名前: `user01-sg-ec2`
- VPC: 自グループのVPC
- インバウンドルール:
  - HTTP (80) / Source: `user01-sg-alb`(セキュリティグループ参照)
- アウトバウンドルール: デフォルト(すべて許可)

**ポイント**: EC2用SGのインバウンドは「ALB用SGのID」を指定することで、ALB経由のトラフィックのみ許可します。IPアドレス指定ではない点が重要な学習ポイントです。

### ステップ2: Launch Template(起動テンプレート)の作成

EC2 > Launch Templates > Create launch template

| 項目              | 値                                           |
| --------------- | ------------------------------------------- |
| 名前              | `user01-lt`                                 |
| AMI             | 自分のAMI                                      |
| インスタンスタイプ       | t3.micro                                    |
| キーペア            | 指定なし(SSM使用のため)                              |
| ネットワーク設定        | セキュリティグループ `user01-sg-ec2` を指定(サブネットは指定しない) |
| IAMインスタンスプロファイル | `ssmAcceptRole`                             |

### ステップ3: Target Group の作成

EC2 > Target Groups > Create target group

| 項目 | 値 |
|---|---|
| ターゲットタイプ | Instances |
| 名前 | `user01-tg` |
| プロトコル / ポート | HTTP / 80 |
| VPC | 自グループのVPC |
| プロトコルバージョン | HTTP1 |
| ヘルスチェックパス | `/` |
| ヘルスチェック間隔 | 30秒(デフォルト) |
| 正常しきい値 | 2(デフォルト) |
| 異常しきい値 | 2(デフォルト) |

**ターゲットの登録はしない**(ASGが自動登録するため)

### ステップ4: Application Load Balancer の作成

EC2 > Load Balancers > Create Load Balancer > Application Load Balancer

| 項目              | 値                                   |
| --------------- | ----------------------------------- |
| 名前              | `user01-alb`                        |
| Scheme          | Internet-facing                     |
| IP address type | IPv4                                |
| VPC             | 自グループのVPC                           |
| Mappings        | AZ-a の Public-a、AZ-c の Public-c を選択 |
| Security Group  | `user01-sg-alb`                     |
| Listener        | HTTP:80 → Forward to `user01-tg`    |

作成後、**ALBのDNS名をメモ**。

### ステップ5: Auto Scaling Group の作成

EC2 > Auto Scaling Groups > Create Auto Scaling group

| 項目                             | 値                                                     |
| ------------------------------ | ----------------------------------------------------- |
| 名前                             | `user01-asg`                                          |
| Launch template                | `user01-lt`                                           |
| VPC                            | 自グループのVPC                                             |
| Availability Zones and subnets | Private-a と Private-c の両方を選択                          |
| Load balancing                 | Attach to an existing load balancer → `user01-tg` を選択 |
| **Health checks**              | **ELB を有効化**(重要)                                      |
| Health check grace period      | 120秒                                                  |
| Desired capacity               | 1                                                     |
| Min capacity                   | 1                                                     |
| Max capacity                   | 1                                                     |

**重要ポイント**: 
- Health check type は **EC2 + ELB の両方** を有効化することで、ALBのヘルスチェック異常もASGが検知します
- これを有効にしないと、Apacheが停止してもASGは異常を検知せず、EC2を置き換えません

### ステップ6: 疎通確認

1. ASGによりEC2が1台起動することを確認
2. Target Groupで該当EC2が `healthy` になることを確認(起動から2〜3分)
3. ブラウザでALBのDNS名にアクセス
4. `Hello from ip-xxx.ap-northeast-1.compute.internal` が表示されることを確認

---

## 動作確認演習: ASG自動復旧の検証

### 演習A: 手動Terminate による復旧確認

1. Session Manager で起動中のEC2に接続
2. EC2一覧から対象インスタンスを選択し、Instance state > Terminate instance
3. ASGが新しいEC2を自動起動することを確認(数分)
4. 新しいEC2がTarget Groupでhealthyになり、ALB経由でアクセス可能になることを確認

### 演習B: ヘルスチェック連動による復旧確認(本命)

1. Session Manager で起動中のEC2に接続
2. Apacheを停止:
   ```bash
   sudo systemctl stop httpd
   ```
3. **観察する挙動**(時系列):
   - 約30秒〜1分: Target Groupで該当EC2が `unhealthy` になる
   - ALB経由のアクセスが 503 Service Unavailable になる
   - ASGが `unhealthy` を検知し、該当EC2を Terminate する
   - ASGが新しいEC2を起動する
   - User dataで再度Apacheがインストール・起動される
   - 新しいEC2がhealthyになり、アクセス復旧
4. 全工程で所要時間およそ 5〜7分

### 演習Bの観察ポイント

- CloudWatch Logs や ASG の Activity history で一連の動きを確認
- 「**EC2内のデータは失われる**」点を体感する(新しいEC2にSessionManagerで入るとホスト名が変わっている)
- 永続化が必要なデータは EBS / S3 / RDS 等に退避すべき、という設計思想を理解する

---

## トラブルシューティング

### ケース1: EC2がTarget Groupでunhealthyのまま

- セキュリティグループ設定を確認(EC2のSGインバウンドでALBのSGを許可しているか)
- User dataが正常終了したか確認: `/var/log/cloud-init-output.log` を Session Manager 経由で確認
- Target Groupのヘルスチェックパスが `/` になっているか確認

### ケース2: Session Manager で接続できない

- EC2のIAMロールに `AmazonSSMManagedInstanceCore` がアタッチされているか
- NAT Gatewayが正常稼働しているか(プライベートサブネットからインターネットに出られるか)
- EC2のステータスチェックが2/2 passedか

### ケース3: ALBにアクセスしても応答なし

- ALBのセキュリティグループでHTTP(80)が0.0.0.0/0から許可されているか
- ALBのスキームが Internet-facing になっているか
- ALBがパブリックサブネットに配置されているか

---

## 想定コスト(1グループ・24時間あたり概算)

| リソース | 単価 | 個数 | 小計 |
|---|---|---|---|
| NAT Gateway | $0.062/時 | 1 | $1.49 |
| NAT Gateway データ処理 | $0.062/GB | - | 少量 |
| ALB | $0.0243/時 | 1 | $0.58 |
| ALB LCU | $0.01/LCU時 | - | 少量 |
| EC2 t3.micro | $0.0136/時 | 1 | $0.33 |
| EBS gp3 8GB | $0.096/GB月 | 1 | 微少 |
| **合計(1グループ)** | | | **約 $2.4/日** |
| **3グループ合計** | | | **約 $7.2/日** |

演習終了後、不要リソースは削除してください。特に **NAT Gateway と ALB は起動しているだけで課金** されます。

## 片付け手順(演習終了時)

受講者作成分:

1. Auto Scaling Group を削除(desired=0にすればEC2が終了)
2. Launch Template を削除
3. Load Balancer を削除
4. Target Group を削除
5. Security Group を削除(ALB用 → EC2用の順、依存関係のため)

講師側で整備したVPC・NAT Gateway・EIPは必要に応じて削除。

---

## 設計上の注意事項(講師向けメモ)

### 相互干渉の許容について

24名が同一アカウント・同一VPC(8名ずつ)で作業するため、以下の事故が起きる可能性があります。事前に受講者に周知してください。

- **他人のリソースの誤削除**: 命名規則を徹底
- **セキュリティグループの誤設定で他人のEC2に影響**: 自分のprefixのSGのみ操作するよう周知
- **ALB/TG名の重複エラー**: 先に作った人が勝つので、命名規則を事前配布

### クォータの事前確認

- **EIP**: デフォルト5個 → 10個程度に緩和申請推奨
- **ALB**: デフォルト50個/リージョン → 24個なのでOK
- **ASG**: デフォルト200個/リージョン → OK
- **Launch Template**: 1万個/リージョン → OK

### なぜ desired=1 の ASG にするのか(受講者への説明用)

一般的にASGは複数台のスケールに使われますが、**1台構成でもASGを使う理由**は「自己修復(Self-healing)」です。

- ASGなしの単体EC2: 障害時は手動復旧
- ASG (min=max=desired=1): 障害時に自動でEC2を置き換え

「スケーリング」ではなく「可用性」のためにASGを使うパターンとして重要な学習ポイントです。

なお、「1台をそのまま修復したい」場合は **EC2 Auto Recovery**(CloudWatchアラーム + recoverアクション)という別機能が存在します。今回の演習ではASG方式を採用しますが、両者の違いを紹介すると学習効果が高まります。


---

## 【訂正】EIPクォータに関する記述について

本ドキュメント内の以下2箇所の記述を訂正します。

### 訂正1: 「事前に講師側で整備しておくもの > 5. EIPクォータ確認」

#### 修正前(誤)

> **重要**: デフォルトでは1アカウントあたりEIP 5個までの制限があります。
> - NAT Gateway 3個 = EIP 3個消費
> - 受講者がEIPを触る演習があれば追加で必要
> - 事前にサービスクォータで緩和申請(10個程度)を推奨

#### 修正後(正)

今回の構成で EIP を消費するのは **NAT Gateway 3個分(=EIP 3個)** のみです。ALB・プライベートEC2はEIPを使いません。

デフォルトクォータは 5個/リージョン なので、**今回の構成単独であれば緩和申請は不要**です。

ただし以下のケースでは事前確認・緩和申請を検討してください:

- 同アカウント・同リージョンで他の用途(別演習、検証環境等)でEIPを既に使用している
- NAT Gateway を各AZ配置に変更する(1VPCあたり2個、合計6個)
- 受講者がEIP関連の演習を別途行う予定がある

### 訂正2: 「設計上の注意事項(講師向けメモ) > クォータの事前確認」

#### 修正前(誤)

> - **EIP**: デフォルト5個 → 10個程度に緩和申請推奨

#### 修正後(正)

- **EIP**: デフォルト5個、本構成で消費するのは3個のためクォータ内に収まる。他用途との併用時のみ緩和申請を検討

### 訂正の理由

受講者の演習手順ではEIPを扱わず、ALBもEIP不要のためです。NAT Gateway 3個で消費するEIP 3個のみが関係しますが、デフォルトクォータ5個の範囲内に収まります。当初「緩和申請推奨」としたのは過剰な注意喚起でした。

**以後、最新情報は本「訂正」セクションを優先してください。**

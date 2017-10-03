# サービス一覧

知らなかったこと・重要そうなこと・忘れそうなことを中心にメモしていく

## Amazon EC2 Container Registry

## VM Import
- 既存のVMイメージをデータセンターからAmazon EC2に移行できる
- VMイメージのディザスタリカバリとして使える

## VM Export
- VM ImportしたインスタンスをAWS外にエクスポートするためのコマンド
- EC2上で作成したインスタンスはExport不可

## Elastic Load Balancing（ELB）
- Classic Load Balancer（CLB）とALBを含んだ総称
    - CLBは旧ELBのこと
- スケーラブル(負荷分散)かつ高可用性のサービスの提供が可能になる
    - ELBがスケールするときには、IPアドレスが変化するので必ずDNS名でアクセスすること
- ELB自体がスケーラブル、従量課金
- ヘルスチェック(ping等)を行ってくれる
- 独自ドメイン名
    - Route 53を使用する場合は、Route 53エイリアスレコード(Zone ApexでなければCNAMEも可)
- クライアントのIPアドレスはHTTPヘッダ上のX-Forwarded-Forで参照可
- 急激なELBへの負荷が予想される場合は、Pre-Warming（暖機運転）の申請を行う
- Internet-Facing ELB
    - インターネットからアクセスできるELB
- Internal ELB
    - VPC内やオンプレミス環境からのみアクセスできればよいELB
- スティッキー セッション
    - 同じユーザから来たリクエストを全て同じEC2インスタンスに送信
    - EC2にセッション情報等を保持する場合に使用（DB等に持たせるほうが望ましい）
- Connection Draining
    - 登録解除やヘルスチェックがFailしたEC2への割り振りを中止するが、処理中リクエストは終わるように一定時間待つ機能
- WebSocketとLB
    - セッション確立時だけLBを使うのが上手な付き合い方

### Application Load Balancer (ALB)
- レイヤー７のコンテントベースのロードバランサー
- コンテントベースのルーティングとは(http://itpro.nikkeibp.co.jp/atcl/column/16/121200298/121200001/?rt=nocnt)
    - 例えば「http://xxxx.yyyy.co.jp/AAA/」というリクエストはAサーバーに、「http://xxxx.yyyy.co.jp/BBB/」というリクエストはBサーバーに振り分けれる
    - 単一のロードバランサーで異なるアプリケーションへリクエストをルーティング可能
- 複数ポート対応と動的ポートマッピングによって、ECSと組み合わせることが可能
- 特徴は以下
    - コンテナベースアプリケーション・WebSocket・HTTP/2のサポート、複数AZ対応のため高耐障害性、ALB自体が自動スケールする
- 料金は利用時間とLoad Balancer Capacity Units（LCU）で計算
    - LCUは新規接続数・アクティブ接続数・帯域幅の内で最も高いディメンションでのみ請求する仕組み

### Classic Load Balancer（CLB）
- TCP/SSL（セキュアTCP）による負荷分散も可能
- バックエンドインスタンスは、全て同一の機能を持ったインスタンスが必要、異なる機能に対してコンテントベースルーティングは出来ない

## Auto Scaling
- 必要な分だけのサーバリソースを確保し、高い可用性とコスト最適化を実現する

### Auto Scaling Group(ASG)
- 情報管理単位。最小/最大台数等を設定する

### Launch Configuration
- 起動するEC2インスタンスの情報。通常のEC2インスタンスを起動する際の入力情報とほぼ同じ

### Scaling Plan
- どのようにスケールするかを定義する。1つのASGに複数のPlanを設定可能

###  動的スケーリングのスケーリング調整タイプ
- ChangeInCapacity
    - 指定した数だけASGの現在の容量を増減
- ExactCapacity
    - 指定した数にASGの現在の容量を変更
- PercentChangeInCapacity
    - 指定した%分だけASGの現在の容量を増減

### Application Auto Scaling
- Auto Scalingと似たUXでAWSリソースの自動スケールを実現
- サポートサービスはECS,SpotFleet,EMR

### その他
- Scale Out
    - EC2インスタンスを増やすこと
- Scale In
    - EC2インスタンスを減らすこと
- クールダウン
    - インスタンス初期化中の無駄なスケーリングを回避するために利用
- ターミネーションポリシー
    - OldestInstance/NewestInstance, OldestLaunchConfiguration, ClosestToNextInstanceHour
    - デフォルトはOldestLaunchConfiguration -> ClosestToNextInstanceHourの順に適用し、複数の場合はランダム
- インスタンス保護
    - Scale Inの対象から除外する用に指定可能
- スケールアウト時の初期化処理をどうするか
    - 設定済みのAMI（ゴールデンイメージ）を用いる
    - user-dataで初期化スクリプトを実行
    - Life Cycle Hooksを利用して初期化する
- Auto Scalingは突発的なスパイクには向いていない
- スパイクに対応するには、CF等の利用・スロットリング機能の設置・一定以上を超えた場合の静的ページへの切替を検討する

##  EC2 Container Service

### Container技術について
- 4つのキーワード。OS仮想化、プロセス隔離、イメージ、自動化
- コンテナは、リソース隔離が元々の由来

#### The Twelve-Factor App
- ベストプラクティスの1つ。アプリをコンテナに対応させる。
    - コードベース: バージョン管理される1つのコードベースと複数デプロイ
    - 依存関係: 依存関係を明示的に宣言し分離する
    - 設定: 設定を環境変数に格納する
    - バックエンドサービス: バックエンドサービスをアタッチされたリソースとして扱う
    - ビルド、リリース、実行: ビルド、リリース、実行の3つのステージを厳密に分離する
    - プロセス: アプリを1つ又は複数のステートレスなプロセスとして実行
    - ポートバインディング: ポートバインディングを通してサービスを公開する
    - 並行性: プロセスモデルによってスケールアウトする
    - 廃棄容易性: 高速な起動とグレースフル停止で堅牢性を最大化する
    - 開発/本番一致: 開発、ステージング、本番環境をできるだけ一致させた状態を保つ
    - ログ: ログをイベントストリームとして扱う
    - 管理プロセス: 管理タスクを1回限りのプロセスとして実行する
    
### Amazon EC2 Container Serviceについて
- コンテナ管理をあらゆるスケールで
- 柔軟なコンテナの配置
- AWSの基盤との連携
- Task: Instance上で実行されている実際のContainer
- Task Definition: Taskで使うContainer、及び周辺設定の定義
- Cluster: Taskが実行されるEC2Instance群
- Manager: ClusterのリソースとTaskの状態を管理
- Scheduler: Clusterの状態をみて、Taskを配置
- Agent: EC2 InstanceとManagerの連携を司る

#### Service
- ロングランニングアプリケーションを希望する状態に保ち続ける
- ALBとの動的ポートマッピング
- Application Auto ScalingのAmazon ECS Serviceサポート

#### Security
- 各TaskにIAMロールを指定することができる
    - 同一のContainer Instance上で異なるIAMロールのTaskが動く
    - Container InstanceのIAMロールは必要最低限にできる

#### Simple
- マスターサーバ群が不要。クラスタ管理ソフト自体を管理する必要がなくなる
- AWS SDK/CLI/CloudFormationで期待通りの動作
- AWSとネイティブな連携

## AWS Elastic Beanstalk(ビーンストーク)
- 定番構成の構築・アプリデプロイの自動化サービス
- アプリケーション
    - トップレベルの論理単位。バージョン、環境、環境設定が含まれている入れ物
- バージョン(Application Version)
    - デプロイ可能なコード。Amazon S3 上でのバージョン管理
- EB Command Line Interface(EB CLI) 
    - ハイレベルな操作が可能なコマンドラインツール
- デプロイメントに関する用語(EBではどちらも可能)
    - In Place Deployment(Rolling Deploy)
        - インスタンスは現行環境のものをそのまま利用し、新しいリビジョンのコードをその場で反映させる
    - Blue/Green Deployment(Red/Black Deployment)
        - 新しいリビジョンのコードを、新しいインスタンスに反映させ、インスタンスごと入れ替える
- デプロイに関する設定
    - バッチタイプ: 一度にデプロイを反映させる台数(バッチ)をどう決めるかを設定
    - バッチサイズ: 各バッチにデプロイするインスタンスの数または割合

## AWS Lambda
- クラウドネイティブとは
    - クラウドで提供されるサービス利用を前提に構築するシステムおよびアプリケーション
    - 仮想サーバ上で1から全てを作り込むのではなく効率的にアプリケーションを実装
    - ビジネスの差別化ポイントへの集中
- インフラを一切気にすることなくアプリケーションコードを実行できるコンピュートサービス
- やりたいこと・ビジネスロジックだけに集中できる
- ファンクション
    - それぞれが隔離されたコンテナ内で実行される
- イベントソース
    - イベントの発生元となるAWSリソース
        - Amazon S3
        – Amazon Kinesis
        – Amazon DynamoDB Streams
        – Amazon Cognito(Sync)
        – Amazon SNS
        – Alexa Skills Kit
        – Amazon SWF
    - Pushモデル(S3, Cognito, SNS)
        - 順不同
        - 3回までリトライ
    - Pull(DynamoDB, Kinesis)
        - 順序性あり。ストリームに入ってきた順に処理される
        - イベントごとに複数のレコードを取得可能
        - データが期限切れになるまでリトライ
- パーミッションモデル
    - ExecutionパーミッションとInvocationパーミッションの2種類
    - ExecutionRole
        - 必要なAWSリソースへのアクセスを許可するIAMロール
- クロスアカウントアクセス
    - 異なるAWSアカウントのアプリケーションからのアクセスが可能
- Blueprint
    - Lambdaファンクションの新規作成時に利用可能なサンプル集
- スケジュール実行
    - CloudWatch Events – Scheduleを使用する
    - Cron形式の指定もサポート
- VPC内のリソースへインターネットを経由せずにアクセス可能
    - Elasticache, RDS, EC2 ...
    - Elastic Network Interface(ENI)を利用
    - ENIには指定したサブネットのIPがDHCPで動的に割り当てられる



### 特徴
- インフラの管理が不要
- オートスケール
- Bring your own code
- 細やかな料金体系
    - アイドル時の支払いは一切なし

### ユースケース
- 監査と通知
    - ex.S3に保管されるCloudTrailのログを分析し、怪しい行動や異常を検知したら通知する
- 写真共有モバイルアプリ
- モーションセンサーを利用した観客参加型ゲーム
- APIサーバの代わりとして

### Lambdaファンクション作成時の注意事項
- ロールを正しくセットアップする
- 再帰処理は慎重に
    - 無限ループに陥る可能性があるので











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
    - セッション確立時だけLBを使う䛾が上手な付き合い方

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

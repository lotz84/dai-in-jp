```
このドキュメントはMakerDAOパープルペーパーの邦訳です。
REFERENCE IMPLEMENTATION of the decentralized DAI STABLECOIN issuance system
[draft; 2018-02-06] https://makerdao.com/purple/
```

## 1. 導入
Dai 安定コインシステムは、担保に裏付けられた（Daiと呼ばれる）トークンを発行し、その価格が分散型の安定性メカニズムに従うように設計されたスマートコントラクトの集まりです。

このドキュメントはシステムの実行可能な技術的な仕様です。これはドラフトでありローンチ後に変わる可能性があります。

システムの概要に関しては[ホワイトペーパー](https://github.com/makerdao/docs/blob/master/Dai.md)を読んで下さい[^1]。

システムの仕組みについて疑問が出てきた場合は、インタラクティブなFAQをお待ちください。

私たちは新しく来た人がシステムを深く理解するための資料を提供することに全力を尽くしています。 これはプロジェクトのガバナンスが将来的に成功するためには重要です。

何か質問がある場合は[チャット](https://chat.makerdao.com/)や[subreddit](https://www.reddit.com/r/MakerDAO/)で聞いて下さい。聞いていただけることは説明資料の作成に役立つので、大変ありがたいです。

### 1.1 なぜリファレンス実装を作ったのか
EthereumブロックチェーンにデプロイされるコントラクトはSolidityで記述されます。このペーパーにはHaskellのプログラムとして書かれたシステムのモデルが記述されています。この動機は次のようなものです:

**比較** 2つの独立した実装を互いに照合して確認することは、両方が意図した通りに動作することを保証するためのよく知られた方法です。

**検証** Haskellでは、QuickCheckなどの強力なテストツールを使用して、主要な性質を包括的に検証することができます。 これはテストと形式検証の中間に位置するものです。

**形式** 純粋関数型プログラムに翻訳する作業は形式検証の機会を与えてくれます。この文書は、Isabellaのような証明補助のシステムの観点でのモデリングに役立つでしょう。

**明白さ** 静的型付け関数型言語であるHaskellでコントラクトの振る舞いをコーディングすることは、Solidityが暗黙的に残してしまう側面を明示的に記述することを強制します。

**明快さ** Blockchainにデプロイする目的ではない実装であれば、Gasのコストの最適化などSolidityの実装を理解し難くするような本質とは関係のない要因がありません。

**シミュレーション** SolidityはBlockchainの環境に特化したもので、ファイルや他のプログラムとやりとりする機能がありません。リファレンス実装は、システムの経済的、ゲーム理論的、統計的な側面のシミュレーションを行うのに便利です。

### 1.2 形式検証とそれまでの道のり
我々は性質を検証するためのインタラクティブなシーケンスを生成する自動テストスイートを開発しています。

検証する性質の一つに、リファレンス実装がオンチェーンでの実装のように動作するかという性質があります。これを検証するために、全ての状態について値の等しさをチェックするSolidityのテストケースを生成しています。

その他の重要な性質は次のようなものです。同じような条件や不変性の下で

* 目標価格が目標レートにしたがって変化する
* Daiの総供給量が完全に計上されている
* アクションはCDPのステージに従って制限されている

これらの性質の正式な記述は、この文章の今後の改訂に含まれる予定です。

### 1.3 ジャーゴンに関する注記
リファレンス実装ではシステム変数とアクションに対して簡潔な単語を使用します。

この文書には強調表示された単語の上にマウスをホバーするとその場で意味を確認できる用語集があります[^2]。

ジャーゴンを使うことの動機は以下のようなものです。

* 言葉の使い方についての議論を避けます。例えば"目標価格の変化率"と呼ぶべきか"目標レート"と呼ぶべきかなどです。
* 財務的や技術的な専門用語から切り離された言葉を使うことで他のものに影響をあたえることなく柔軟に改善することができます。
* 部分的に意味を欠いた財務的な解釈を使ってシステムを形式的に論じることによって、通常の言語の中では考えるのが難しいような洞察も得ることができます。
* 正確で示唆的な言葉は実装の構造とロジックをより明白に表現し、より簡単に形式化することを可能にします。
* 簡潔な名前はコードをあまり冗長にせず、紙やホワイトボードなどで扱いやすいコンセプトにします。

## 2. Dai の仕組み
> Note: このセクションは不完全です。ここでは関連する定義へのリンクを用意してシステムの明示的な仕組みを簡潔にかつ技術的に説明しようとしています。

Dai安定コインシステムでは、ユーザーの担保資産をロックし、担保の市場価値に比例したDaiを発行することができます。 したがって一定量の安定コインを引き出すために、価値のあるトークンを預けることができます。 このような預金口座は"担保付債務ポジション"またはCDPと呼ばれます。

参考: [lock](https://makerdao.com/purple/#dfn-lock), [draw](https://makerdao.com/purple/#dfn-draw), [Urn](https://makerdao.com/purple/#dfn-Urn)

そのような預金が十分な市場価値を保持している限り、ユーザーはDaiを払い戻すことで部分的または全体的に預金を回収することができます。 CDPが必要な比率を超えて担保されている限り、ユーザーはDaiを払い戻さずに預金の一部を回収することによって担保を減らすこともできます。

参考: [free](https://makerdao.com/purple/#dfn-free), [wipe](https://makerdao.com/purple/#dfn-wipe)

ガバナンスはどの外部トークンが担保として有効であるかを決め、最大Dai発行額や最低担保比率などのパラメータが違う異なる預金クラス、すなわち"CDPタイプ"を作成します。

参考: [Ilk](https://makerdao.com/purple/#dfn-Ilk)

担保の必要要件を確認するために、システムは市場価格ではなく安定性メカニズムによって調整された独自の目標価格でDaiを評価します。

参考: [feel](https://makerdao.com/purple/#dfn-feel), これはCDPのライフサイクルのステージを決定します

目標価格の調整は二次的な効果です。 一次的には、安定性メカニズムは目標レートを調整することによって市場価格の変化に反応します。

参考: [prod](https://makerdao.com/purple/#dfn-prod), これは安定性メカニズムを更新します。

## 3. 前置きとデータ型
このペーパーに記載されているプログラムは、外部ライブラリで定義されたいくつかのシンボルを使用します。 ほとんどのシンボルは文脈的に意味が明確だと思いますが、私たちの"Prelude"を見ればインポートされたそれぞれの型と関数が簡潔な解説とともに一覧して理解することができるでしょう。

`[TODO: Preludeを表示する]`

```hs
module Maker where
import Prelude (); import Maker.Prelude; import Maker.Decimal
```

### 3.1 数値型
システムは小数を2つの精度で使用しており、これらに対して短い名前を与えています。

1つは `wad`と呼ばれ18桁の精度を持ちます。これはETH, DAI, MKRなどのトークンの量を表すために使用されます。

もう1つは`ray`と呼ばれ36桁の精度を持ちます。これは安定化手数料パラメータなどの正確なレートと比率に使用されます。

これらは異なる型として定義されるので、型システムによって明示的な変換を用いずにそれらを組み合わせることはできません。

```hs
newtype Wad = Wad (Decimal E18)
  deriving (Ord, Eq, Num, Real, Fractional, RealFrac)

newtype Ray = Ray (Decimal E36)
  deriving (Ord, Eq, Num, Real, Fractional, RealFrac)
```

これらの型のうち一方をもう片方の型に変換する汎用関数を定義します。

```hs
cast x = fromRational (toRational x) -- [Via fractional n/m form]
```

また、時間の長さの型を1秒単位で定義します。

```hs
newtype Sec = Sec Int
  deriving (Eq, Ord, Enum, Num, Real, Integral)
```

### 3.2 識別子とアドレス
以下のよくあるHaskellでの実装によって `Id Ilk`, `Id Urn`などを異なる識別子の型として使用することができます。

```hs
newtype Id a = Id String
  deriving (Eq, Ord, Show)
```

Ethereumアカウントのアドレスを表す別の型も定義しておきます。

```hs
newtype Address = Address String
  deriving (Eq, Ord, Show)
```

### 3.3 Gem, SIN, DAI, MKR: トークン識別子
システムは、4つのトークンの基本的な型を使います。

```hs
data Token
  -- システムガバナンスによって承認された担保トークン
  = Gem (Id Tag)
  -- CDP所有者によって発行された、トレード可能で価値の代替物となる安定コイン
  | DAI
  -- 発行されたDaiの総額と常に等しい額となる、システム内部で使用される反対コイン
  | SIN
  -- 価格が安定していないカウンターコインであり投票用のトークン
  | MKR
  deriving (Eq, Ord, Show)
```

システムが承認した担保トークンは、"gems"と呼ばれます。 `Id Tag`型を使用して、担保トークンの識別情報を表します。

このモデルでは、全ての担保トークンがシンボルのみが異なる単純なERC20トークンとして扱われます。 実際には、投票者はトークンが承認される前にそのトークンが適切な動作をすることを確認する必要があります。

### 3.4 Tag: 担保トークン価格のレコード
価格フィードから受け取ったデータは、`Tag`レコードに保存されます。

```hs
data Tag = Tag {
  -- 最新のトークンの市場価格（SDR建て）
  tag :: Wad,
  -- 価格が失効したとみなされる時間のタイムスタンプ
  zzz :: Sec
} deriving (Eq, Show)
```

### 3.5 Urn: CDPのレコード

`Urn`レコードは1つのCDPを表します。

```hs
data Urn = Urn {
  -- CDPタイプの識別子
  ilk  :: Id Ilk,
  -- CDPの所有者
  lad  :: Address,
  -- このCDPによって発行された未払いDaiの総額（負債単位建てで表示）
  art  :: Wad,
  -- このCDPによって現在ロックされている担保の総額
  ink  :: Wad,
  -- 清算を発動したActor
  cat  :: Maybe Actor
} deriving (Eq, Show)
```

### 3.6 Ilk: CDPタイプのレコード
`Ilk`レコードは1つのCDPタイプを表します。

```hs
data Ilk = Ilk {
  -- このタイプのCDPの担保として使用されるトークン
  gem :: Id Tag,
  -- このタイプのCDPに保持できる負債総額（負債単位建てで表示）
  rum :: Wad,
  -- 現在の負債単位のDai建ての価値。安定化手数料に伴って増加する
  chi :: Ray,
  -- 負債の上限: このCDPタイプによって発行できる未払いDaiの最大総額
  hat :: Wad,
  -- 清算率（Daiあたりの担保価格）
  mat :: Ray,
  -- 清算ペナルティ（Dai建て）
  axe :: Ray,
  -- 手数料（１秒あたり、Dai建て）
  tax :: Ray,
  -- 価格フィードの利用不可能期間の猶予期間
  lax :: Sec,
  -- 負債単位が調整された最新のタイムスタンプ
  rho :: Sec
} deriving (Eq, Show)
```

### 3.7 Vox: フィードバックメカニズムのレコード
フィードバックメカニズムは、市場価格に基づいてDaiの目標価格を調整する仕組みです。 そのデータは`Vox`というレコードにまとめられています。

```hs
data Vox = Vox {
  -- SDR建てのDaiの市場価格
  wut :: Wad,
  -- SDR建てのDaiの目標価格
  par :: Wad,
  -- 現在の目標価格の1秒あたりの変化量
  way :: Ray,
  -- 感度パラメータ（ガバナンスによって設定される）
  how :: Ray,
  -- 最新のフィードバックが実行された時のタイムスタンプ
  tau :: Sec
} deriving (Eq, Show)
```

### 3.8 Actor: アカウント識別子
トークンの残高を持ちアクションを呼び出すことができる様々なエンティティを明示的に区別するためにデータ型を使用します。

```hs
data Actor
  -- 外部アドレス（CDPの所有者）
  = Account Address
  -- 担保の金庫。清算まで全てのロックされた担保を保持する
  | Jar
  -- DAIとSINは"Jug"によって生成され焼却される
  | Jug
  -- 精算者として機能するもの
  | Vow
  -- Daiを調達して流動性を保つ担保の競売人
  | Flipper
  -- MKRを購入する際に手数料収入を支払う、 購入と焼却を行う競売人
  | Flapper
  -- 流動性を保つためにMKRを生成する、値段の引き上げと売却を行う競売人
  | Flopper
  -- テスト用（実際のシステムには存在しない）
  | Toy
  -- 全ての機能を持ったActor（一時的な対応）
  | God
  deriving (Eq, Ord, Show)
```

### 3.9 システムモデル
最後にモデルの持つ全ての状態を定義します。

```hs
data System = System {
  -- フィードバックメカニズムのデータ
  vox      :: Vox,
  -- CDPのデータ
  urns     :: Map (Id Urn) Urn,
  -- CDPタイプのデータ
  ilks     :: Map (Id Ilk) Ilk,
  -- 担保トークンの価格タグ
  tags     :: Map (Id Tag) Tag,
  -- 各Actorの各トークンのトークン残高
  balances :: Map (Actor, Token) Wad,
  -- 現在のタイムスタンプ
  era      :: Sec,
  -- 精算者の操作モード
  mode     :: Mode,
  -- 現在のアクションの送信者
  sender   :: Actor,
  -- 全てのユーザーのアカウント（テスト用）
  accounts :: [Address]
} deriving (Eq, Show)
```

```hs
-- 精算者に関連する Work In Progress
data Mode = Dummy
  deriving (Eq, Show)
```

## 4. アクション
アクションはシステムの基本的な状態遷移です。

**内部的なもの** と指定されていない限り、アクションはブロックチェーン上のパブリックな機能として使うことができます。

`auth`修飾子は、システムが権限を付与したアドレスからのみ呼び出されるアクションを示します。

アクションの定義が状態とロールバックに関してどのように振る舞うかの根底にある"Actionモナド"の詳細については、 `6.1 Action モナド` の章を参照してください。

### 4.1 発行
ユーザーは`open`を使ってシステムの中で1つまたは複数のアカウントを作り、自分で選択したアカウント識別子とilkを指定することができます。

```hs
open id_urn id_ilk = do
  -- アカウント識別子が既に使われていたら失敗する
  none (urns . ix id_urn)

  -- ilkの示すタイプが存在しなければ失敗する
  _ <- look (ilks . ix id_ilk)

  -- 送信者を所有者としてCDPを作成する
  Account id_lad <- use sender
  initialize (urns . at id_urn) (emptyUrn id_ilk id_lad)
```

Urnの所有者は`give`を使って所有権をいつでも移転することができます。

```hs
give id_urn id_lad = do
  -- 送信者がCDPの所有者でなければ失敗する
  id_sender <- use sender
  owns id_urn id_sender

 -- CDPの所有権を移転する
  assign (urns . ix id_urn . lad) id_lad
```

CDPが精算されない限り、所有者は`lock`を使ってさらに担保をロックすることができます。

```hs
lock id_urn wad_gem = do
  -- 送信者がCDPの所有者でなければ失敗する
  id_lad <- use sender
  owns id_urn id_lad

  -- 清算中であれば失敗する
  want (feel id_urn) (not . oneOf [Grief, Dread])

  -- 担保トークンを特定する
  id_ilk  <- look (urns . ix id_urn . ilk)
  id_tag  <- look (ilks . ix id_ilk . gem)

  -- 担保を拘束する
  transfer (Gem id_tag) wad_gem id_lad Jar

  -- 担保トークンの残高が増加したことを記録する
  increase (urns . ix id_urn . ink) wad_gem
```

CDPにリスク上の問題がなければ（ilkの上限を超えている場合を除く）、その所有者はCDPが清算率を下回らない限り `free` を使用することができます。

```hs
free id_urn wad_gem = do
  -- 送信者がCDPの所有者でなければ失敗する
  id_lad <- use sender
  owns id_urn idl_ad

  -- 担保トークンの残高が減少したことを記録する
  decrease (urns . ix id_urn . ink) wad_gem

  -- ilkの上限を超えている場合を以外のリスク上の問題があればロールバックする
  want (feel id_urn) (oneOf [Pride, Anger])

  -- 担保を開放する
  id_ilk <- look (urns . ix id_urn . ilk)
  id_tag <- look (ilks . ix id_ilk . gem)
  transfer (Gem id_tag) wad_gem Jar id_lad
```

CDPにリスク上の問題がなければその所有者は、ilkの上限を超えておらず発行によってCDPが精算率を下回らない限り、 `draw`を使用して新たな安定コインを発行することができます。

```hs
draw id_urn wad_dai = do
  -- 送信者がCDPの所有者でなければ失敗する
  id_lad <- use sender
  owns id_urn id_lad

  -- 負債単位と未処理の手数料収益を更新する
  id_ilk <- look (urns . ix id_urn . ilk)
  chi1   <- drip id_ilk

  -- 発行額を負債単位で計算する
  let wad_chi = wad_dai / cast chi1

  -- CDPの安定コインの発行量が増加したことを記録する
  increase (urns . ix id_urn . art) wad_chi

  -- ilkの安定コインの発行量が増加したことを記録する
  increase (ilks . ix id_ilk . rum) wad_chi

  -- リスク上の問題があればロールバックする
  want (feel id_urn) (== Pride)

  -- 安定コインと反対コインの両方を新しく作る
  lend wad_dai

  -- 安定コインをCDP所有者に送る
  transfer DAI waddai Jug idlad
```

安定コインを発行したCDP所有者は`wipe`を使用して、Daiを送り戻しCDPの発行額を減らすことができます。

```hs
wipe id_urn wad_dai = do
  -- 送信者がCDPの所有者でなければ失敗する
  id_lad <- use sender
  owns id_urn id_lad

  -- CDPが精算中であれば失敗する
  want (feel id_urn) (not . oneOf [Grief, Dread])

  -- 負債単位と未処理の手数料収益を更新する
  id_ilk <- look (urns . ix id_urn . ilk)
  chi1   <- drip id_ilk

  -- 発行額を負債単位で計算する
  let wad_chi = wad_dai / cast chi1

  -- CDPの発行量が減少したことを記録する
  decrease (urns . ix id_urn . art) wad_chi

  -- ilkの発行量が減少したことを記録する
  decrease (ilks . ix id_ilk . rum) wad_chi

  -- CDP所有者から安定コインを拘束する
  transfer DAI wad_dai id_lad Jar

  -- 安定コインと反対コインを消滅させる
  mend wad_dai
```

価格フィードが最新でCDPが精算されていない場合、CDP所有者は `shut` を使ってアカウントを閉じることができます。これにより全ての担保が回収され全ての発行と手数料がキャンセルされます。

```hs
shut id_urn = do
  -- 負債単位と未処理の手数料収益を更新する
  id_ilk <- look (urns . ix id_urn . ilk)
  chi1   <- drip id_ilk

  -- 発行された全ての安定コインと手数料を予約する
  art0 <- look (urns . ix id_urn . art)
  wipe id_urn (art0 * cast chi1)

  -- ロックされた担保を全て回収する
  ink0 <- look (urns . ix id_urn . ink)
  free id_urn ink0

  -- CDPを無効にする
  assign (urns . at id_urn) Nothing
```

### 4.2 評価
CDPのライフサイクルの6個のステージを定義します。

```hs
data Stage
  -- 超過担保, 負債上限を下回るCDPタイプ, 新しい価格タグ, 清算が発動されていない状態
  = Pride

  -- CDPタイプの負債上限に達した状態
  | Anger

  -- CDPタイプの担保価格が不安定な状態
  | Worry

  -- 担保が過小なCDPもしくは タイプの価格が不安定な状態の猶予期間
  | Panic

  -- 清算が発動された状態
  | Grief

  -- 清算が発動され、開始された状態
  | Dread
  deriving (Eq, Show)
```

#### 4.2.1  ステージ効果のライフサイクル
以下の表はCDPライフサイクルの各ステージでどのCDPアクションが許可もしくは禁止されているかを示しています。

```
                          担保を減少させる
                           ╭┈┈┈┈┈┈┈┈┈╮
        give shut lock wipe free draw bite grab plop
Pride    ■    ■    ■    ■    ■    ■
Anger    ■    ■    ■    ■    ■
Worry    ■    ■    ■    ■
Panic    ■    ■    ■    ■              ■
Grief    ■                                  ■
Dread    ■                                       ■
        give shut lock wipe free draw bite grab plop
             ╰┈┈┈┈┈┈┈┈┈┈┈┈┈╯          ╰┈┈┈┈┈┈┈┈┈┈┈┈┈╯
           　　　　　　　　　担保を増加させる 　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　精算
```

この表から以下のようなことが分かります。

* 担保を増加させるアクションは `Grief` までしか認められていない。
* `free` は `Anger` の間まで許可されているが、 `draw` は `Pride` の間までしか許可されていない
* `give` は精算の間も含めていつでも許可されている
* 精算のアクションはそれぞれ決められたステージでしか許可されていない

#### 4.2.2 CDPステージの分析

CDPのライフサイクルのステージを決定する `analyze` を定義します。

```hs
analyze era0 par0 urn0 ilk0 tag0 =
  let

    -- ロックされたurnの担保のSDR建ての値
    pro = view ink urn0 * view tag tag0

    -- SDR建てのCDPの発行量
    con = view art urn0 * cast (view chi ilk0) * par0

    -- 精算率毎に必要な担保の値
    min = con * cast (view mat ilk0)

    -- CDPタイプのDAIの総発行量
    cap = view rum ilk0 * cast (view chi ilk0)

  in if

     -- 順番に条件をチェックする
    | has cat urn0 && view ink urn0 == 0    -> Dread
    | has cat urn0                          -> Grief
    | pro < min                             -> Panic
    | view zzz tag0 + view lax ilk0 < era0  -> Panic
    | view zzz tag0 < era0                  -> Worry
    | cap > view hat ilk0                   -> Anger
    | otherwise                             -> Pride
```

次に、システムの状態を更新した後に `analyze` の値を返す内部的なアクション `feel` を定義します。

```hs
feel id_urn = do
  -- 目標価格と目標レートを調整する
  prod

  -- 負債単位と未処理の手数料収益を更新する
  id_ilk <- look (urns . ix id_urn . ilk)
  drip id_ilk

  -- analyze のためのパラメータを読み込む
  era0 <- use era
  par0 <- use (vox . par)
  urn0 <- look (urns . ix id_urn)
  ilk0 <- look (ilks . ix (view ilk urn0))
  tag0 <- look (tags . ix (view gem ilk0))

  -- CDPのライフサイクルのステージを返す
  return (analyze era0 par0 urn0 ilk0 tag0)
```

CDPアクションはすでに危険な状態である時にリスクが増加することを防ぐために `feel` を使い、安定コインと担保を精算の間凍結します。

### 4.3 調整

フィードバックメカニズムは`prod` によって更新され、Keeperはこれをいつでも呼び出すことができます。さらに `feel` を使ってリスクを評価するCDPのアクションの副作用としても呼び出されます。

```hs
prod = do
  -- フィードバックメカニズムに関係する全てのパラメータを読み込む
  era0 <- use era
  tau0 <- use (vox . tau)
  wut0 <- use (vox . wut)
  par0 <- use (vox . par)
  how0 <- use (vox . how)
  way0 <- use (vox . way)

  let
    -- 時間差（秒）
    age  = era0 - tau0

    -- 目標価格に適用されている現在の目標レート
    par1  = par0 * cast (way0 ^^ age)

    -- 時間の経過とともに適用される感度パラメータ
    wag  = how0 * fromIntegral age

    -- 目標レートの上昇もしくは下降
    way1  = inj (prj way0 +
                 if wut0 < par0 then wag else -wag)

  -- 目標価格の更新
  assign (vox . par) par1

  -- 価格変化率の更新
  assign (vox . way) way1

  -- 更新時間を記録
  assign (vox . tau) era0

  where
    -- 乗算形式と加算形式の変換
    prj x  = if x >= 1  then x - 1  else 1 - 1 / x
    inj x  = if x >= 0  then x + 1  else 1 / (1 - x)
```

ilkの安定化手数料はガバナンスによって変更されます。アクションを一定時間内に行わないといけない制約があるため全てのCDPにこのような変更の効果を与えることができません。代わりにそれぞれのilkは安定化手数料を累積した1つの"負債単位"を持っています。 `drip`アクションはこの単位を更新します。これはKeeperによっていつでも呼び出されることができますが、`feel` を使ってCDPのリスクを評価する全てのアクションの副作用としても呼び出されます。

```hs
drip id_ilk = do
  rho0  <- look (ilks . ix id_ilk . rho)
  tax0  <- look (ilks . ix id_ilk . tax)
  chi0  <- look (ilks . ix id_ilk . chi)
  rum0  <- look (ilks . ix id_ilk . rum)
  era0  <- use era

  let
    -- 時間差（秒）
    age   = era0 - rho0
    -- 安定化手数料によって増加した負債単位の値
    chi1 = chi0 * tax0 ^^ age
    -- 新しい単位による安定化手数料収益
    dew   = (cast (chi1 - chi0) :: Wad) * rum0

  -- 少し増加した手数料のために安定コインと反対コインを新しく作る
  lend dew

  -- 更新時間を記録
  assign (ilks . ix id_ilk . rho) era0

  -- 新しい負債単位を記録
  assign (ilks . ix id_ilk . chi) chi1

  -- 新しい負債単位を返す
  return chi1
```

### 4.4 価格フィードの入力

`mark` アクションは価格の有効期限と一緒に新しい担保の市場価格を記録します。

```hs
mark id_gem tag_1 zzz_1 = auth $ do
  initialize (tags . at id_gem) Tag {
    · tag  = tag_1,
    · zzz  = zzz_1
  }
```

`tell` アクションは価格の有効期限と一緒に新しいDaiの市場価格を記録します。

```hs
tell wad = auth $ do
  assign (vox . wut) wad
```

### 4.5 精算
CDPが精算が必要なステージになったら、精算プロセスを開始するために任意のアカウントが `bite` を呼び出すことができます。これによって精算者コントラクトはオークションのために担保を手に入れ、反対コインを引き継ぐことができます。

```hs
bite id_urn = do
  -- CDPが適切なステージでなければ失敗する
  want (feel id_urn) (== Panic)

  -- 送信者を精算開始者として記録する
  id_cat <- use sender
  assign (urns . ix id_urn . cat) (Just id_cat)

  -- CDPの発行に精算ペナルティを適用する
  id_ilk <- look (urns . ix id_urn . ilk)
  axe0   <- look (ilks . ix id_ilk . axe)
  art0   <- look (urns . ix id_urn . art)
  let art1 = art0 * cast axe0

  -- ペナルティを含めるためにCDPの発行を更新する
  assign (urns . ix id_urn . art) art1
```

精算が発動されたら、指定された精算者コントラクトがCDPの担保とCDPの発行に対応する反対コインを受け取るために`grab`を呼び出す。

```hs
grab id_urn = auth $ do
  -- CDPが精算が発動された状態でなければ失敗する
  want (feel id_urn) (== Grief)

  ink0   <- look (urns . ix id_urn . ink)
  art0   <- look (urns . ix id_urn . art)
  id_ilk <- look (urns . ix id_urn . ilk)
  id_tag <- look (ilks . ix id_ilk . gem)

  -- 負債単位と未処理の手数料収益を更新する
  chi1 <- drip id_ilk

  -- 発行額をDai建てで計算する
  let con = art0 * cast chi1

  -- 担保と反対コインを精算者に送信する
  transfer (Gem id_tag) ink0 Jar Vow
  transfer SIN con Jug Vow

  -- CDPの担保と反対コインの量を0にする
  assign (urns . ix id_urn . ink) 0
  assign (urns . ix id_urn . art) 0

  -- ilkの総発行額を減らす
  decrease (ilks . ix id_ilk . rum) art0
```

精算者がCDPの精算を終えたら、 CDPを売却したり復元したりする必要のない担保を返却するために `plop` を呼び出します。

```hs
plop id_urn wad_dai = auth $ do
  -- CDPが適切なステージになければ失敗する
  want (feel id_urn) (== Dread)

  -- CDPの精算の開始者の記録を消す
  assign (urns . ix id_urn . cat) Nothing

  -- 過剰な担保を精算者から金庫に移す
  id_vow <- use sender
  id_ilk <- look (urns . ix id_urn . ilk)
  id_tag <- look (ilks . ix id_ilk . gem)
  transfer (Gem id_tag) wad_dai id_vow Jar

  -- 超過担保をCDPに属するものとして記録する
  assign (urns . ix id_urn . ink) wad_dai
```

精算者はいつでもカウンターコインの購入焼却オークションで使うために未回収の安定化手数料の収益を `loot` を使って請求することができます。

```hs
loot = auth $ do
  -- Daiの金庫の残高は未回収の安定化手数料の収益
  wad <- look (balance DAI Jug)

  -- 全てのDaiの金庫の残高を精算者に送る
  transfer DAI wad Jug Vow
```

### 4.6 オークション

> Note: このセクションは未完成で、全てのオークションはダミーです

```hs
flip id_gem wad_jam wad_tab id_urn = do
  vow <- look mode
  case vow of
    Dummy -> return ()
```

```hs
flap = do
  vow <- look mode
  case vow of
    Dummy -> return ()

flop = do
  vow <- look mode
  case vow of
    Dummy -> return ()
```

## 4.7 セトルメント

```hs
tidy who = auth $ do
  -- エンティティの安定コインと反対コインの残高を調べる
  awe <- look (balance DAI who)
  woe <- look (balance SIN who)

  -- 二つのバランスの少ない残高まで焼却することができる
  let x = min awe woe

  -- 安定コインと反対コインを精算者に送る
  transfer DAI x who Vow
  transfer SIN x who Vow

  -- 安定コインと反対コインの両方を焼却する
  burn DAI x Vow
  burn SIN x Vow
```

```hs
kick = do
  -- Vowアカウントに未処理の安定化手数料の収益を送る
  loot

  -- 反対コインに対して安定コインをキャンセルする
  tidy Vow

  -- 全ての残っている安定コインをcointercoinの下落オークションに割り当てる
  transferAll DAI Vow Flapper
  flap

  -- 全ての残っている反対コインをcointercoinの上昇オークションに割り当てる
  transferAll SIN Vow Flopper
  flop
```

## 4.8 ガバナンス

ガバナンスは`form` を使って新しいilkを作ることができます。新しいタイプは0で初期化されるため、発行が行われる前に別のトランザクションで安全にリスクパラメータを設定することができます。

```hs
form id_ilk id_gem = auth $ do
  initialize (ilks . at id_ilk) (defaultIlk id_gem)
```

ガバナンスはフィードバックメカニズムの唯一変更可能なパラメータである感度ファクターを `frob` を使って変更することができます。

```hs
frob how1 = auth $ do
  assign (vox . how) how1
```

ガバナンスはilkの5つのリスクパラメータを変更できます。 `cuff` は精算比率を、 `chop` は精算ペナルティを、 `cork` はilkの上限を、 `calm` は価格が不安定な状態の期間を、 `crop` は安定化手数料を変更します。

```hs
cuff id_ilk mat1 = auth $ do
  assign (ilks . ix id_ilk . mat) mat1

chop id_ilk axe1 = auth $ do
  assign (ilks . ix id_ilk . axe) axe1

cork id_ilk hat1 = auth $ do
  assign (ilks . ix id_ilk . hat) hat1

calm id_ilk lax1 = auth $ do
  assign (ilks . ix id_ilk . lax) lax1
```

`crop` で安定化手数料を変更する時は変更前の安定化手数料が内部的な負債単位を考慮にいれるようにします。

```hs
crop id_ilk tax1 = auth $ do
  -- 現在の安定化手数料を内部的な負債単位に適用する
  drip id_ilk

  -- 安定化手数料を変更する
  assign (ilks . ix id_ilk . tax) tax1
```

## 4.9 トークン操作

ERC20の transfer 関数を（"allowance"のコンセプトを除いて）単純な形でモデル化します。

```hs
transfer id_gem wad src dst =
  -- トークンの残高テーブルの中で操作を行う
  zoom balances $ do

　　  -- 元の残高が不十分なら失敗する
 　　 balance <- look (ix (src, id_gem))
 　　 aver (balance >= wad)

　　  -- 残高を更新する
　　  decrease    (ix (src, id_gem)) wad
　　  initialize  (at (dst, id_gem)) 0
　　  increase    (ix (dst, id_gem)) wad
```

```hs
transferAll id_gem src dst = do
  wad <- look (balance id_gem src)
  transfer id_gem wad src dst
```

内部的なアクションである `mint` はトークンの供給量を増やします。これは `lend` によって安定コインと反対コインを新しく生成するに使われ、精算者によって新しいカウンターコインを生成するのに使われます。　

```hs
mint id_gem wad dst = do
  initialize (balances . at (dst, id_gem)) 0
  increase   (balances . ix (dst, id_gem)) wad
```

内部的なアクションである　`burn` はトークンの供給量を減らします。これは`mend` によって安定コインと反対コインを消滅させるのに使われ、精算者によってカウンターコインを消滅されるのに使われます。

```hs
burn id_gem wad src =
  decrease (balances . ix (src, id_gem)) wad
```

内部的なアクションである `lend` は安定コインと反対コインの両方を同じ量だけ新しく作ります。これは `draw` によって安定コインを発行するのに使われ、 `drip` によって回収されるまで金庫の中に置かれる安定化手数料からの収益を表す安定コインを発行するのにも使われます。

```hs
lend wad_dai = do
  mint DAI wad_dai Jug
  mint SIN wad_dai Jug
```

内部的なアクションである `mend` はDaiと内部的な負債トークンを同じ量だけ消滅させます。これは `wipe` を通して使用されることで安定コインの供給を減らします。

```hs
mend wad_dai = do
  burn DAI wad_dai Jug 
  burn SIN wad_dai Jug
```

## 5 デフォルトのデータ

```hs
defaultIlk :: Id Tag -> Ilk
defaultIlk id_tag = Ilk {
  gem = id_tag,
  axe = Ray 1,
  mat = Ray 1,
  tax = Ray 1,
  hat = Wad 0,
  lax = Sec 0,
  chi = Ray 1,
  rum = Wad 0,
  rho = Sec 0
}
```

```hs
emptyUrn :: Id Ilk -> Address -> Urn
emptyUrn id_ilk id_lad = Urn {
  cat = Nothing,
  lad = id_lad,
  ilk = id_ilk,
  art = Wad 0,
  ink = Wad 0
}
```

```hs
initialTag :: Tag
initialTag = Tag {
  tag = Wad 0,
  zzz = 0
}
```

```hs
initialSystem :: Ray -> System
initialSystem how0 = System {
  balances = empty,
  ilks     = empty,
  urns     = empty,
  tags     = empty,
  era      = 0,
  sender   = God,
  accounts = mempty,
  mode     = Dummy,
  vox      = Vox {
    tau = 0,
    wut = Wad 1,
    par = Wad 1,
    how = how0,
    way = Ray 1
  }
}
```

## 6 アクションの枠組み

読者はコードを理解するためにモナドの抽象的な理解は一切必要ありません。それは純粋な関数のまま例外や状態を表現するためのシンタックス（`do`記法）を提供しています。そのようなブロックの各行は必要なセマンティクスを提供するモナドによって解釈されます。

### 6.1 Actionモナド
`Action`モナドを単純なStateモナドとErrorモナドの組み合わせとして定義します。

```hs
type Action a = StateT System (Except Error) a
```

アクションの失敗を一般的な検証の失敗と認証の失敗に分けます。

```hs
data Error = AssertError Act | AuthError
  deriving (Show, Eq)
```

アクションは `exec` を使うことで与えられた最初のシステムの状態の上で実行されます。この結果はエラーもしくは新しい状態です。`exec` 関数はアクションの列も受け取ることができ、これは一つのトランザクションとして解釈されます。

```hs
exec :: System -> Action () -> Either Error System
exec sys m = runExcept (execStateT m sys)
```

### 6.2 検証
ある条件が満たされないと失敗する関数をいくつか定義します。

```hs
-- | 一般的な検証
aver x = unless x (throwError (AssertError ?act))

-- | 値が存在しないことの検証
none x = preuse x >>= \case
  Nothing -> return ()
  Just _  -> throwError (AssertError ?act)

-- | 値が存在することの検証
look f = preuse f >>= \case
  Nothing -> throwError (AssertError ?act)
  Just x  -> return x

-- | アクションを実行しその結果に対してある条件が満たされているかの検証
want m p = m >>= (aver . p)

has p x = view p x /= Nothing
```

`owns id_urn id_lad` を与えられたCDPが与えられたアカウントが所有しているかの検証として定義します。

```hs
owns id_urn id_lad =
  want (look (urns . ix id_urn . lad)) ((== id_lad) . Account)
```

`auth k` を送信者が認証されている場合のみ`k`を実行するようなアクションの修飾子として定義します。

```hs
auth continue = do
  s <- use sender
  unless (s == God) (throwError AuthError)
  continue
```

## 7 用語集
`Id a` は（任意の`a`に対して）別の`Id b`と混ぜて使えないような文字列の識別子。

`Urn` はCDPを表すレコード。

`Ilk` はCDPタイプを表すレコード。

`Urn`の`lad`(型は`Actor`)はそのCDPの所有者のアカウント識別子。

`Urn`の`art`（型は`Wad`）はCDPによって発行された未払いDaiの総額。

`Urn`の`ink`（型は`Wad`）はそのCDPによってロックされている担保の総量。

`Urn`の`cat`は可能な場合にCDPの精算を発動するActor。

`Ilk`の`gem`は対応するCDPタイプの担保に使われる担保トークン。

`Ilk`の`tax`（型は`Ray`）は対応するCDPタイプのCDPに課された安定化手数料であり、1秒あたりのCDPの未払いDaiを表す。

`Ilk`の`lax`（型は`Sec`）はCDPの対応するタイプに適用される失効された担保の価格タグの猶予期間。

`Ilk`の`hat`（型は`Wad`）は対応するCDPタイプから発行できるDaiの最大総額（切り上げ）。

`Ilk`の`rum`（型は`Wad`）は対応するCDPタイプの現在の総発行額で、CDPの内部的な負債単位で表される。

`Ilk`の`chi`（型は`Ray`）は対応するCDPタイプのDai建ての内部的な負債単位であり、CDPタイプの安定化手数料に従って時間の経過とともに大きくなる。

`Ilk`の`mat`（型は`Ray`）は対応するCDPタイプのCDPにおいて要求される最小の担保化比率（担保の値を発行されたDaiの値で割ったもの）。

`Ilk`の`axe`（型は`Ray`）は対応するCDPタイプの精算されたCDPに課されるペナルティであり、CDPの未払いDaiとして表される。

`Ilk`の`rho`（型は`Sec`）は負債単位が調整された時点の最新のタイムスタンプ。

`Tag`の`tag`（型は`Wad`）は対応する担保トークンのSDR建てで記録された市場価格。

`Tag`の`zzz`（型は`Sec`）は対応する担保の価格のタグが失効するタイムスタンプ。

`Price`はリスクの無いCDPのリスクステージ。

`Anger`は負債上限に達したタイプのCDPのステージであり、担保化され過ぎた状態で精算がまだ発動していないもの。

`Worry`は担保の価格フィードが既に失効していてまだCDPタイプの猶予期間にあるCDPのステージ。清算の発動はまだしていない状態（CDPタイプは債務上限に達している可能性もある）。

`Panic`は担保化されているか価格フィードがCDPタイプの猶予期間を過ぎて失効しているCDPのステージ。清算の発動はまだしていない状態（CDPタイプは債務上限に達している可能性もある）。

`Grief`は清算が発動されたCDPのステージ。

`Dread`は清算を受けているCDPのステージ。

`has k x`はレコードxのフィールドxがNothingでなければtrueとなる。

`Wad`は18桁の精度を持つ小数値の型。トークンの量を表すのに使われる。

`Ray`は36桁の精度を持つ小数値の型。厳密なレートや比率を表すのに使われる。

`Sec`はタイムスタンプや秒で表される期間の型。

`case x`はxを式のコンテクストの中で求められている何らかの数値型に変換する。その過程で精度を失う可能性もある。

`Address`は任意のEthereumアカウントのアドレスを表現する。

`Token`はシステムによって利用されるERC20トークンの識別子。`Gem`（担保トークン）, `SIN`, `DAI`, `MKR`の中のいずれか。

`Gem`は`Token`のコンストラクタで担保トークンを表す。

`DAI`はDai安定コイントークンの識別子。

`MKR`はMKRトークン（カウンターコインとガバナンストークン）の識別子。

`SIN`は常にDaiと同じ量だけ生成されて焼却される内部で使われる”反対コイン”トークン。会計のための量としてシステムの中だけで使われる。

`wut`(型は`Wad`)はフィードバックメカニズムでのDaiの最新の市場価格。SDR建て。

`par`（型は`Wad`）はフィードバックメカニズムでのDaiの最新の目標価格。SDR建て。

`way`（型は`Ray`）は現在の1秒あたりの目標価格の変化量。感度パラメータに従ってフィードバックメカニズムにより連続的に変化している。

`how`（型は`Ray`）はフィードバックメカニズムの感度パラメータ。ガバナンスによって設定されDaiの目標価格の変化率をコントロールする。

`tau`（型は`Sec`）は最新のフィードバックメカニズムが実行されたタイムスタンプ。

`Tag`は担保の価格フィードの更新のレコード。型`Id Tag`は担保トークン(`Gem`)を特定するために使われる。

`Vox`はフィードバックメカニズムのデータのレコード。

`Actor`はトークン残高を持ちシステムアクションを起こすことができるエンティティの識別子を表す。

`Account`（型は `Address -> Actor`）は外部のEthererumアカウントを表す`Actor`識別子を構築する。

`Jar`（型は`Actor`）はシステムの担保の金庫。

`Jug`（型は`Actor`）はDAI/SINを作りSINを保有するActor。

`Vow`（型は`Actor`）はシステムの精算者として機能するもの。

`Flipper`（型は`Actor`）は担保の競売人として機能するもの。

`Flapper`（型は`Actor`）はDAI安定コインの競売人として機能するもの。

`Flopper`（型は`Actor`）はMKRカウンターコインの競売人として機能するもの。

`Toy`（型は`Actor`）はシステムのテストドライバー（本番には存在しない）。

`God`（型は`Actor`）は全ての権限を持つActor（プロトタイピングのために作られたもので削除される予定）。

`lock`は担保をCDP所有者からシステムのトークン金庫に送金し、`ink`の増加をCDPの`Urn`に記録する。

`draw`は超過して担保化されているCDPの所有者のために新しく`DAI`を作成する。

`give`はCDPの所有権を移転する。

`free`は超過して担保化されているCDPに担保を請求する。

`prod`は安定化フィードバックメカニズムを更新する。目標価格(`par`)を目標レート(`way`)に従って調整し、目標レートを現在の市場価格（`wut`）と感度パラメータ(`how`)に従って調整する。

`feel`はCDPのステージを計算する。これには担保化の要件の決定と価格フィードの状態と清算の進捗を確認することも含まれる。

----

訳注

[^1]: 最新のホワイトペーパーはこちらです <https://makerdao.com/whitepaper/DaiDec17WP.pdf>
[^2]: こちらの邦訳にはありません

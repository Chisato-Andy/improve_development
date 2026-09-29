# 良いアーキテクチャの本質（初心者向けまとめ）

この資料で言っている「良いアーキテクチャ」は、ざっくりいうと次の3つを大事にしています。

1. ビジネスの核心（ドメイン）に集中できること  
2. 技術的な詳細から分離できること  
3. 柔軟に組み合わせられる小さなコンポーネントで作ること（継承より委譲）

順番に、初心者向けにかみ砕いてまとめます。

---

## 1. ビジネスの核心（ドメイン）に集中できること

ソフトウェアの本来の目的は「ビジネス上の課題を解決すること」です。

- ECなら「商品を検索して購入する」「在庫を管理する」「配送を手配する」
- 倉庫なら「出庫する」「入庫する」「在庫差異を管理する」

こういう「業務の流れ」そのものが**ドメイン（ビジネスロジック）**です。

一方で、実装にはたくさんの「技術的な話」が出てきます。

- DB接続、SQL実行
- HTTPリクエストとレスポンス
- 外部APIの呼び出しとエラー処理
- ログ、モニタリング、キャッシュ など

### 悪い状態：ドメインに技術がべったりくっついている

よくある悪い例：

- 出庫処理のメソッドの中で
  - JDBCで接続
  - SQL書く
  - ログ出す
  - 例外ハンドリング  
  …を**全部1メソッドに書いてしまう**

すると、本当に確認したい「出庫の業務ルール」がコードの中に埋もれます。

### 良い状態：技術は別クラスに任せ、ビジネスだけを書く

良い例では、出庫処理はこんな感じになります（イメージ）：

```java
public class ShipmentService {
    private final ItemRepository repository;

    public void processShipment(String itemId, int quantity) {
        Item item = repository.findById(itemId)
            .orElseThrow(() -> new ItemNotFoundException(itemId));

        if (!item.hasEnoughStock(quantity)) {
            throw new InsufficientStockException(...);
        }

        item.ship(quantity);
        repository.save(item);

        if (item.needsReplenishment()) {
            replenishmentService.createOrder(item);
        }
    }
}
```

ここでは：

- DB接続やSQLは **`ItemRepository` に任せている**
- このクラスには **「出庫のビジネスルールだけ」が書かれている**

こうして**関心事を分離**すると、このクラス自体が「仕様書」に近い読みやすさになります。

---

## 2. 技術的な詳細から分離できること

UIフレームワークやDB・外部サービスなどの「技術」は、時間とともに変わります。

- jQuery → React → Next.js…
- Oracle → PostgreSQL → クラウドマネージドDB…
- 自前の決済 → Stripe → 別のサービス…

### 悪い設計だと

DBを Oracle から PostgreSQL に変えようとすると：

- ほぼ全てのビジネスクラスに埋め込まれた SQL を修正
- テストコードも全部DB前提で書かれていて大修正
- Oracle 固有のエラーコード処理など、あちこち書き換え

という「地獄の全面改修」になります。

### 良い設計だと

- DBに依存する実装は **`adapters/repositories` の中だけ** に閉じ込める
- ビジネスクラスは DB の種類を意識しない（リポジトリのインターフェースだけ知っている）

結果として、DBを変えるときの影響範囲は：

- Repository の実装クラスの差し替え
- 設定ファイルの変更

くらいで済みます。

**ポイント**

> 「ビジネスロジック」と「技術的な詳細」を物理的（パッケージ）にも、論理的（依存関係）にも分ける  
> → 技術が変わっても、ビジネスのコードがほとんど揺れない

---

## 3. 柔軟なコンポーネントの組み合わせ（継承より委譲）

### 継承に頼りすぎる問題

「再利用したいから」といって、なんでも親クラスを作って継承していくと：

- 親クラスを変えると全部の子クラスに影響する（脆弱な基底クラス）
- 階層が増えると追いきれない
- DB接続クラスを継承しているせいで、DBを変えたくなった時に継承関係ごと変更…

ということが起きやすいです。

### 委譲（Composition）で小さい部品を組み合わせる

良い設計では「必要な機能を**持つ**」形にします。

```java
public class UserService {
    private final UserRepository repository;      // DBはここに「委譲」
    private final NotificationService notifier;   // 通知はここに「委譲」

    public void registerUser(User user) {
        repository.save(user);
        notifier.sendWelcomeEmail(user);
    }
}
```

- DBを変えたければ `UserRepository` の実装を差し替えるだけ
- 通知方法をメール → Slack に変えたければ `NotificationService` の実装だけ変えればよい

典型的な流れはこうなります：

```text
Controller  →  Application  →  Usecase  →  Service  →  Repository
（入口）        （調整役）       （業務フロー）      （計算）        （DB）
```

**効果**

- クラスごとの責務が小さくなる（単一責任）
- 差し替えやテストがしやすい（モックを入れ替えるだけ）
- 再利用もしやすい（小さな部品単位で使い回せる）

---

# 依存性逆転の原則（DIP）入門

ここまでの話を支える中心的な考え方が **DIP（Dependency Inversion Principle）** です。

### DIP の2つのルール（ざっくり）

1. **上位レベル（ビジネス側）のモジュールは、下位レベル（技術側）に直接依存してはいけない。両方とも「抽象」に依存すべき。**
2. **抽象は詳細に依存してはならない。詳細（実装クラス）が抽象（インターフェース）に依存すべき。**

言い換えると：

- ビジネスロジックは、DB の具体クラスに依存しない
- どちらも「リポジトリのインターフェース」に依存する
- そのインターフェースは、ドメイン側が定義する

### よくある悪い例

```java
public class RetrievalFlow {
    private final ItemRepository repository; // 具体クラスに依存

    public void execute(String itemCode, int quantity) {
        Item item = repository.getItem(itemCode);
        ...
    }
}
```

- `ItemRepository` の中身が JDBC でも JPA でも Mongo でも、**RetrievalFlow がそれに引きずられる**

### DIP を適用した良い例

```java
// ドメイン側に IItemRepository を定義（ポート）
public interface IItemRepository {
    Item getItem(String itemCode);
    void save(Item item);
}

// ドメインのビジネスロジックはインターフェースにだけ依存
public class RetrievalFlow {
    private final IItemRepository itemRepository;

    public RetrievalFlow(IItemRepository itemRepository) {
        this.itemRepository = itemRepository;
    }

    public void execute(String itemCode, int quantity) {
        Item item = itemRepository.getItem(itemCode);
        ...
    }
}

// 技術側（インフラ）はこのインターフェースを実装する
public class OracleItemRepository implements IItemRepository { ... }
public class PostgresItemRepository implements IItemRepository { ... }
public class MongoItemRepository   implements IItemRepository { ... }
```

**依存の向きが「逆転」しているのがポイント**

- 前：`ドメイン → 具体DBクラス`
- 後：`具体DBクラス → ドメインが定義したインターフェース`

---

## DIP がもたらすメリット

### 1. 認知負荷の軽減

- 具体クラスに依存：  
  「このクラスは中で何してるんだろう？Oracle？接続どうなってる？」と、実装を見に行きたくなる
- インターフェースに依存：  
  「`getItem` で取れて、`save` で保存できる。それ以上は知らなくていい」と割り切れる

→ 開発者の頭は**「ビジネスの流れ」に集中できる**

### 2. テストのしやすさ

DIP のおかげで、テスト時に本物の DB を使う必要がなくなります。

- 本番：`RetrievalFlow(new OracleItemRepository(...))`
- テスト：`RetrievalFlow(new MockItemRepository(...))`

モック実装や Mockito などで簡単に差し替えられます。

**結果**

- テストが速い
- 外部環境に左右されない
- CI でどこでも動かせる

---

# 依存性の注入（Dependency Injection, DI）

DIP を現実のコードで実現するテクニックが **DI（依存性の注入）** です。

ポイントは：

> 「使う側のクラスの中で `new` しない。必要なものは外から渡してもらう」

### 代表的な3パターン

1. **コンストラクタインジェクション（推奨）**

```java
public class RetrievalFlow {
    private final IItemRepository repository;

    public RetrievalFlow(IItemRepository repository) {
        this.repository = repository;
    }
}
```

2. セッターインジェクション  
3. フィールドインジェクション（フレームワークのアノテーションで注入）

初心者はまず「**コンストラクタで受け取る**」と覚えておけばOKです。

### 手動 DI と DI コンテナ

小さなアプリなら、自分でこう書いてもよいです。

```java
DataSource ds = createDataSource();
IItemRepository repo = new OracleItemRepository(ds);
RetrievalFlow flow = new RetrievalFlow(repo);
```

大きくなってくると、これを全部手で書くのはつらいので、

- Spring, Guice, Dagger などの **DIコンテナ** に
  - 「このインターフェースにはこの実装を使って」と登録しておくと
  - フレームワークが自動で `new` と組み立てをやってくれる

こういう仕組みをよく使います（Spring Boot がまさにそれ）。

---

# Port / Adapter（ポート・アダプタ）のイメージ

最後に、DIP を直感的にイメージするための比喩です。

## コンセントの話

現実世界でいうと：

- **電化製品（ドメイン）**
  - ドライヤー、掃除機、PC など
- **プラグの形・電圧の仕様（ポート = 契約）**
  - 「100V / 2ピン」といった仕様
- **変換プラグ・変圧器（アダプタ）**
  - 日本のプラグをヨーロッパのコンセントに挿すための変換
- **各国のコンセント・電源（インフラ）**

大事なのは：

- **電化製品本体はどこの国でも同じ**  
- 国ごとの差は「変換プラグ（アダプタ）」で吸収する

## ソフトウェアに対応させると

- **ドメイン（電化製品）**
  - `InventoryService`, `ShipmentUseCase` など
- **ポート（プラグの仕様）**
  - `IItemRepository`, `INotificationService` などのインターフェース
- **アダプタ**
  - `OracleItemRepository`, `PostgresItemRepository`, `EmailNotificationService`
- **インフラ**
  - Oracle / PostgreSQL / MongoDB / SMTP サーバー / 外部API など

ドメイン側は「こういうインターフェースが欲しい」とだけ決めておき、  
インフラ側がそれに合わせて実装する＝**プラグに合わせて変換プラグを作る**イメージです。

## 得られるメリット

- DB や外部サービスを**差し替えやすい**
- ドメインのテストに**本物のインフラが要らない**
- ドメインチームとインフラチームが**並行開発しやすい**
- 責任範囲がはっきり分かれる  
  - ドメイン＝ビジネスロジック  
  - アダプタ＝技術的な詳細

---

## まとめ

この資料が伝えたいことを一言でいうと：

> **「ドメイン（ビジネス）を中心において、技術は後からいくらでも差し替えられるように設計しよう」**

そのための具体的な武器が：

- 関心の分離（ドメインと技術の分離）
- 委譲（小さいコンポーネントの組み合わせ）
- DIP（依存性逆転の原則）
- DI（依存性の注入）
- Port / Adapter（ポートとアダプタ）の考え方

です。

もし「このプロジェクトで、どの部分を Port / Adapter にすればいいのか？」みたいな具体的な相談があれば、  
実際のクラスや機能例を出してもらえれば、一緒に「ここがポートで、これがアダプタ」と分解していくこともできます。

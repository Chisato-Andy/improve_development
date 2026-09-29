# パッケージ構成ガイドライン（初心者向け）

このガイドは、以下の AsciiDoc をベースにした **「実際のプロジェクトで使いやすいパッケージ構成」** の入門版です。

---

## 1. 全体の考え方

- **domain**：ビジネスルールの本体（できるだけ純粋な Java）
- **application**：ユースケースの実行・調整・トランザクション
- **adapters**：Web/API・DB・外部サービスなど「技術的な話」

そして **「外側の層は、内側の層に依存してよいが、逆はダメ」** というルールで構成します。

依存の矢印イメージ：

```text
adapters  →  application  →  domain
（外の世界）      （調整役）        （ビジネスの中心）
```

---

## 2. パッケージ構成の例

```text
src/
├── domain/          # ビジネスロジックの中心（フレームワーク非依存）
│   ├── entities/    # エンティティ・値オブジェクト
│   ├── services/    # ドメインサービス（複数エンティティにまたがる処理）
│   ├── usecases/    # ユースケース（アプリケーションサービス）
│   └── ports/       # 外界との「契約」（インターフェース）
│       ├── presenters/
│       ├── repositories/
│       └── external/
│
├── application/     # ユースケースの実行調整・DI・トランザクション境界
│
└── adapters/        # 外部との接続（実装の詳細）
    ├── controllers/ # Web API、UI などの入り口
    ├── presenters/  # レスポンス生成（JSON 等）
    ├── repositories/# DB アクセス実装
    └── external/    # その他外部サービス実装
```

---

## 3. まず覚えるべきルール3つ

1. **domain はどこからも参照されるが、自分からは何にも依存しない（ほぼ純粋 Java）**
2. **外部世界（Web/API、DB、外部サービス）のコードはすべて adapters 配下に置く**
3. **技術詳細（Spring, JPA など）は domain に入れない**

細かいルールは後述しますが、まずはこの3つだけ意識すると失敗しにくいです。

---

## 4. 各パッケージの役割とルール

### 4.1 `domain` パッケージ

#### 4.1.1 `domain/entities`（エンティティ・値オブジェクト）

**何を書くところか**

- ビジネスの中心となるモデル
  - 例：`Order`, `Item`, `Money`, `Quantity` など
- 不変条件・ビジネスルールをメソッドとして持つ

**ここから参照してよいもの**

- Java標準クラス（`String`, `List`, `LocalDateTime` など）
- 同じ `domain/entities` 内のクラス
- 副作用のないユーティリティ（必要なら）

**ダメなもの（初心者はここだけ意識しておけばOK）**

- Spring（`@Component`, `@Service`, `@Autowired` など）
- JPA（`@Entity`, `@Id`, `@GeneratedValue` など）
- DB・HTTP・ファイルI/O など外部アクセス系

**良い例（ビジネスルールだけ）**

```java
public class Item {
    private final String id;
    private String name;
    private Quantity quantity;

    public void ship(Quantity shippingQuantity) {
        if (this.quantity.isLessThan(shippingQuantity)) {
            throw new InsufficientStockException();
        }
        this.quantity = this.quantity.subtract(shippingQuantity);
    }
}
```

**悪い例（技術が混ざっている）**

```java
public class Item {

    @Id           // ← JPA: NG
    @GeneratedValue
    private Long id;

    @Autowired    // ← Spring: NG
    private ItemRepository repository;

    public void save() {    // ← DBアクセスを直接呼ぶ: NG
        repository.save(this);
    }
}
```

---

#### 4.1.2 `domain/services`（ドメインサービス）

**何を書くところか**

- 1つのエンティティに閉じないビジネスロジック  
  例：`PricingService`, `InventoryService` など
- 複数エンティティを横断した計算や判断

**参照してよいもの**

- `domain/entities`
- `domain/ports/repositories`（リポジトリポート）
- `domain/ports/external`（外部サービス用ポート）

**ダメなもの**

- `adapters/*` の実装クラス（DB実装、HTTP実装など）
- Spring の `JdbcTemplate` や `RestTemplate` など具体的技術

**良い例**

```java
public class PricingService {
    private final IItemRepository itemRepository;
    private final ITaxCalculator taxCalculator;

    public Money calculateTotalPrice(List<OrderLine> orderLines) {
        Money subtotal = Money.zero();
        for (OrderLine line : orderLines) {
            Item item = itemRepository.findById(line.getItemId());
            subtotal = subtotal.add(item.getPrice().multiply(line.getQuantity()));
        }
        return taxCalculator.addTax(subtotal);
    }
}
```

---

#### 4.1.3 `domain/usecases`（ユースケース）

**何を書くところか**

- 「○○を行う」という**業務フロー全体**  
  例：`PlaceOrderUseCase`, `ShipmentUseCase`
- 入力を受け取り → ドメインを呼び出し → 結果を出力ポートに流す

**参照してよいもの**

- `domain/entities`
- `domain/services`
- `domain/ports/repositories`
- `domain/ports/presenters`
- `domain/ports/external`

**ポイント**

- Web や DB のことは知らない
- 「誰にどう見せるか」は `presenters` に任せる

**イメージ例**

```java
public class ShipmentUseCase {
    private final InventoryService inventoryService;
    private final IItemRepository itemRepository;
    private final INotificationService notificationService;

    public void execute(ShipmentRequest request, IShipmentPresenter presenter) {
        try {
            Item item = itemRepository.findById(request.getItemId());
            inventoryService.processShipment(item, request.getQuantity());
            itemRepository.save(item);
            notificationService.notifyShipment(item, request.getQuantity());

            presenter.presentSuccess(new ShipmentResult(item, request.getQuantity()));
        } catch (InsufficientStockException e) {
            presenter.presentError("在庫不足です: " + e.getMessage());
        }
    }
}
```

---

#### 4.1.4 `domain/ports/*`（ポート）

**何を書くところか**

- ドメインが外部に対して期待する「契約（インターフェース）」  
  - `repositories/`：DBアクセスのインターフェース
  - `external/`：メール通知・外部APIなど
  - `presenters/`：ユースケース結果の出力方法

**重要ポイント**

- **インターフェースだけ置く**
- 戻り値・引数には **domain の型（エンティティやVO）だけ** を使う  
  → `ResultSet` や `HttpResponse` など技術の型は出さない

**良い例（repositories）**

```java
public interface IItemRepository {
    Optional<Item> findById(String id);
    void save(Item item);
}
```

---

### 4.2 `application` パッケージ

**何を書くところか**

- ユースケースの実行を外部から呼びやすくする「窓口」
- トランザクション境界（`@Transactional`）の指定
- DI コンテナによる組み立て（Springの設定）

**参照してよいもの**

- `domain/usecases`
- `domain/ports/*`

**良いイメージ**

```java
@Component
public class ShipmentApplicationService {
    private final ShipmentUseCase shipmentUseCase;

    @Transactional
    public void processShipment(ShipmentRequest request, IShipmentPresenter presenter) {
        shipmentUseCase.execute(request, presenter);
    }
}
```

**ポイント**

- コントローラ（Web層）は、この `application` のサービスを呼ぶ
- 直接 `ShipmentUseCase` をコントローラから呼ばない、というルールにしておくと分かりやすい

---

### 4.3 `adapters` パッケージ

#### 4.3.1 `adapters/controllers`（入力アダプター）

**何を書くところか**

- Web API / UI / バッチ起動 など**外部からの入り口**
  - 例：`@RestController` のクラス
- リクエストを DTO に変換し、`application` のサービスに渡す

**参照してよいもの**

- `application`（基本ここだけ呼ぶ）
- 場合によっては domain の Enum を参照（DTO変換用）  
  ※ただし、直接ドメインをいじり始めると責務が濁るので注意

**良い例**

```java
@RestController
@RequestMapping("/items")
public class ItemController {

    private final ShipmentApplicationService applicationService;
    private final JsonShipmentPresenter presenter;

    @PostMapping("/ship")
    public ResponseEntity<ShipmentResponse> ship(@RequestBody ShipmentRequestDto request) {
        applicationService.processShipment(request.toDomainRequest(), presenter);
        return ResponseEntity.ok(presenter.getResponse());
    }
}
```

**悪い例**

```java
@RestController
public class ItemController {

    @Autowired
    private ShipmentUseCase useCase;          // UseCase を直接持つ: NG

    @Autowired
    private JpaItemRepository repository;     // 具体的リポジトリを直接持つ: NG
}
```

---

#### 4.3.2 `adapters/presenters`（出力アダプター）

**何を書くところか**

- ユースケースの結果を JSON / HTML / CSV などに変換するクラス
- `domain/ports/presenters` を実装する

**参照してよいもの**

- `domain/ports/presenters`
- `domain/entities`（結果変換のため）

---

#### 4.3.3 `adapters/repositories`（DBアダプター）

**何を書くところか**

- 実際のDBアクセス（JPA, MyBatis, JDBC など）
- `domain/ports/repositories` の実装クラス

**参照してよいもの**

- `domain/ports/repositories`
- `domain/entities`（マッピング対象）
- JPA / JDBC / PostgreSQL ドライバ など技術詳細

**良いイメージ**

```java
@Repository
public class JpaItemRepository implements IItemRepository {

    @PersistenceContext
    private EntityManager em;

    @Override
    public Optional<Item> findById(String id) {
        ItemEntity entity = em.find(ItemEntity.class, id);
        return Optional.ofNullable(entity).map(this::toDomainModel);
    }

    private Item toDomainModel(ItemEntity e) {
        return new Item(e.getId(), e.getName(), new Quantity(e.getQuantity()));
    }
}
```

---

#### 4.3.4 `adapters/external`（外部サービスアダプター）

**何を書くところか**

- メール、外部REST API、メッセージキューなど
- `domain/ports/external` の実装クラス

---

## 5. 初心者がまず守ればいいチェックリスト

画面やAPI・DBコードを書くときに、これだけチェックすると事故りにくいです。

1. **エンティティ（`domain/entities`）に Spring / JPA のアノテーションを付けていないか？**
2. **コントローラ（`adapters/controllers`）で、具体的なリポジトリ実装（JPAクラスなど）を `new` したり `@Autowired` していないか？**
3. **`domain/services` が JDBC や Web クライアントを直接使っていないか？**
4. **DB アクセスのコードは `adapters/repositories` に集中しているか？**
5. **外部サービスとの通信は `adapters/external` に閉じているか？**

迷ったら、

> 「これはビジネスルールの話か？」→ domain  
> 「これは技術の話か？」→ adapters  
> 「その2つをどう繋ぐか？」→ application  

と考えると、置き場を判断しやすくなります。

---

## 6. まとめ

- **domain**：ビジネスロジックの中心。できるだけ純粋な Java コード。
- **application**：ユースケースの実行・トランザクション・DI の調整役。
- **adapters**：Web / DB / 外部サービスなど技術的な接続部分。

この構成にしておくと、

- Spring やDBを変えたくなったとき
- バッチやCLIなど別の入口を追加したくなったとき

にも、影響範囲を局所化しやすくなります。



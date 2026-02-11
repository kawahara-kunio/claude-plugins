# Kotlin/Spring ベストプラクティスチェックリスト

Framework Best Practicesサブエージェントが使用するKotlin/Spring固有のレビュー基準。
neuhaus-backendプロジェクトのDDD/Clean Architectureパターンに基づく。

---

## Clean Architecture レイヤー規約

### レイヤー依存方向
```
Presentation → Application → Domain ← Infrastructure
```
- Domain層は他のレイヤーに依存しない
- Infrastructure層はDomain層のインターフェース/抽象クラスを実装する
- レイヤーをまたぐ依存方向の逆転がないか確認

### Domain層
- **Value Object**: `data class`で不変、`init`ブロックでバリデーション、`DomainValidateException`をスロー
- **DomainObject（Entity）**: 一意識別子を持つ、ビジネスロジックを内包
- **Aggregate**: DomainObject + ValueObjectで整合性境界を形成
- **ファクトリメソッド**: インスタンス生成は`of()`メソッドで行う
- **companion object**: クラス先頭に配置、`UBIQUITOUS_NAME`定数を定義
- **ID型**: `NeuhausId`を実装した型安全なID（`TenantId`, `CustomerId`等）
- **Sealed class**: ポリモーフィックなドメイン概念に使用（`when`で`else`を避け全パターン明示）
- **バリデーション**: `check`/`require`ではなく`if`文を使用
- **First Class Collection**: ラップ対象のドメインモデルと同じファイルに定義
- **外部依存禁止**: Domain層からInfrastructure/Presentation層への依存がないこと

### Application層
- **UseCase**: `@Service`、公開メソッド名は`handle`のみ
- **命名**: `{Verb}{Subject}UseCase`（例: `CreateCustomerUseCase`）
- **QueryService**: 集約をまたぐ複雑なクエリ用。インターフェースをApplication層に、実装をInfrastructure層に定義
- **EventListener**: ドメインイベント処理、メソッド名は`execute`
- **Input**: プリミティブ型のみ（String, Int, Long, Boolean）またはドメインのenum
- **Output**: ドメインのValue Objectを使用可能

### Infrastructure層
- **Repository実装**: 抽象クラスを継承、ドメインイベントの発行を含む
- **DAO**: Doma Criteria API（KQueryDsl）を使用、SQLファイルではなくCriteria APIで記述
- **Entity**: Domaアノテーション（`@Entity(immutable = true)`, `@Table`, `@Id`, `@Version`）
- **マッピング**: `toXxx()`メソッドでEntity↔ドメインモデル変換
- **テナント分離**: DAOクエリで`tenantIdEq()`等のユーティリティ関数を使用

### Presentation層
- **Controller**: 薄く保つ、UseCaseに委譲
- **パス**: 単数形名詞（`/customer/{id}`、`/customers/{id}`ではない）
- **一覧取得**: POST + body（GETではない）、サフィックス`_list`
- **Enum**: コード値（文字列）で受け渡し
- **認可**: admin APIのみ`@RoleBasedAccess`
- **Request/Response**: Controllerごとに専用のDTO

---

## Kotlin

### Null安全
- `!!`演算子の使用を避ける（安全呼び出し`?.`やElvis`?:`を優先）
- プラットフォーム型（Java連携時）のnull安全な処理
- `lateinit`の適切な使用（初期化保証があるか）
- `?.let { }`の過度なネストを避ける

### Data Class
- `var`ではなく`val`で不変性を保証
- 1パラメータ1行、名前付き引数、末尾カンマ
- `copy()`メソッドの適切な活用

### Sealed Class / Enum
- 分岐処理には`sealed class`/`sealed interface`で網羅性を保証
- `when`式で`else`を避け、全パターンを明示
- ステータス系enumはサフィックス`Type`（例: `CustomerStatusType`）

### コレクション操作
- シーケンス（`asSequence()`）の活用（大きなコレクションのチェーン操作）
- 不変コレクション（`List`, `Map`）をデフォルトで使用
- `var`を避け`val`を使用

---

## Spring Framework

### DI（依存性注入）
- コンストラクタインジェクションを優先（`@Autowired`フィールドインジェクションを避ける）
- `@Component`/`@Service`/`@Repository`の適切なステレオタイプ使用
- 循環依存の回避

### @Transactional
- **UseCase層**で管理（Repository/Domain Serviceではない）
- 書き込み操作: `@Transactional`
- 読み取り操作: `@Transactional(readOnly = true)`
- DB未使用: アノテーション不要
- `@Transactional`はpublicメソッドに配置（プロキシの制約）
- 自クラス内呼び出しでは`@Transactional`が効かない問題の認識

### 例外処理
- ドメイン例外階層の活用（`NotFoundException`, `DomainValidateException`等）
- Controllerでドメイン例外をcatchしHTTP例外に変換
- Failure enumによる網羅的なエラーハンドリング（`when`で全パターン明示）
- `@ControllerAdvice`でのグローバルエラーハンドリング

### Spring Security
- admin APIのみ`@RoleBasedAccess`で認可制御
- Read操作: 全ロール許可（ADMINISTRATOR, EDITOR, ACCOUNT_READER）
- Write操作: Admin/Editorのみ

### REST API設計
- パス: 単数形名詞
- 一覧: POST + body、サフィックス`_list`
- ケース変換: グローバルObjectMapper使用（個別`@JsonProperty`不要）
- バリデーション: `@Validated`をControllerに付与

---

## Doma ORM

### DAO
- Criteria API（KQueryDsl）を使用、SQLファイルは使わない
- テーブル/ビュー定数: companion objectに`{ENTITY_NAME}_TABLE`/`{ENTITY_NAME}_VIEW`
- `limit == 0`の場合は`emptyList()`を返す
- オプショナルフィルタはnullableパラメータ（空リストではない）
- デフォルトパラメータ値は使わない

### Entity
- `@Entity(immutable = true, metamodel = Metamodel())`
- 楽観ロック: `@Version`フィールド
- 監査データ: `AuditData`埋め込み型
- ID: String型で保存、ドメイン型との相互変換メソッド

---

## テスト

### フレームワーク
- **Mockito Kotlin**を使用（MockKではない）
- JUnit 5（Jupiter）
- テスト名は**日本語バッククォート**（例: `` `正常にインスタンス生成できること` ``）

### テスト構造
- `@Nested`内部クラスで関連テストをグループ化
- テストクラスは`private class`
- セットアップ→実行→検証の3フェーズ

### レイヤー別テスト要件
- **Domain**: Value Objectの`of()`とバリデーション、ドメインロジックのテスト
- **Application**: UseCaseテストは原則不要（`ExecuteAsyncTaskUseCase`サブクラスは除く）
- **Infrastructure**: Repository/QueryServiceテストはTestContainers + DbSetupで実施。DAOの個別テストは不要
- **Presentation**: ユニットテスト（`@WebMvcTest` + パッケージ別アノテーション）**と**インテグレーションテスト**の両方が必須

### インテグレーションテスト
- TestContainers（PostgreSQL）使用
- `DbTestHelper.setUpDbWithFkCheck()`でデータセットアップ
- Fixtureクラスでテストデータ生成
- 外部サービスのモック禁止
- 固定時刻: エポックミリ秒0（1970-01-01T00:00:00.000Z）

### カスタムアノテーション
- Admin Controller: `@WebMvcTest` + `@AdminControllerTest` / `@AdminIntegrationTest` + `@ControllerIntegrationTest`
- API/Frontend Controller: `@WebMvcTest` + `@ControllerTest` / `@ControllerIntegrationTest`
- Public Controller: `@WebMvcTest` + `@PublicControllerTest` / `@PublicIntegrationTest` + `@ControllerIntegrationTest`
- Repository: `RepositoryTest`を継承
- EventListener: `@EventListenerTest`

---

## KDoc / コメント

- public/protectedメソッドにKDocを記述
- ただし`of()`メソッド、Controller、Entity変換メソッドには不要
- ユビキタス名による自己文書化を優先（ドメイン層では原則KDoc不要）
- 例外は`@exception`でドキュメント化、アルファベット順に列挙

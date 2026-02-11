# TypeScript/Next.js ベストプラクティスチェックリスト

Framework Best Practicesサブエージェントが使用するTypeScript/Next.js固有のレビュー基準。

---

## TypeScript

### 型安全
- `any`型の使用を避ける（`unknown` + 型ガードを優先）
- 型アサーション（`as`）の最小化（型ガードやnarrowingを優先）
- strict modeの準拠（`strictNullChecks`, `noImplicitAny`）
- `!`（non-null assertion）の濫用を避ける

### 型設計
- 判別共用体（Discriminated Unions）の活用
- ユーティリティ型の活用（`Partial`, `Pick`, `Omit`, `Record`等）
- ジェネリクスの適切な制約（`extends`）
- `interface` vs `type`の一貫した使用
- `enum`よりもunion型またはconst objectを検討

### 関数・モジュール
- 戻り値型の明示（推論に頼りすぎない、特にパブリックAPI）
- オプショナルパラメータの適切な使用
- `import type`の活用（型のみのインポート）
- バレルファイル（index.ts）の適切な使用

---

## Next.js (App Router)

### Server Components vs Client Components
- デフォルトでServer Components（`'use client'`は必要最小限）
- `'use client'`の配置（コンポーネントツリーのできるだけ下位に）
- Client Componentからのサーバー専用コードのインポートを避ける
- Server Componentでの非シリアライズ可能なpropsの使用

### データフェッチング
- Server Componentsでの直接的なデータ取得を優先
- `fetch`のキャッシュ/再検証戦略（`revalidate`, `cache`）
- 不要なクライアントサイドフェッチを避ける
- Route Handlersの適切な使用場面

### Server Actions
- `'use server'`の適切な配置
- Server Actions内の入力バリデーション（セキュリティ上重要）
- revalidation（`revalidatePath`, `revalidateTag`）の呼び出し
- エラーハンドリングとユーザーへのフィードバック

### ルーティング・レイアウト
- レイアウトの適切な活用（共通UI、データフェッチ）
- `loading.tsx`, `error.tsx`, `not-found.tsx`の配置
- Route Segment Config（`dynamic`, `revalidate`）の適切な設定
- パラレルルート・インターセプティングルートの適切な使用

### Middleware
- Middlewareの軽量性（Edge Runtimeの制約）
- Middlewareでの認証/認可チェック
- `matcher`設定の適切性

### 最適化
- `next/image`の活用（画像最適化）
- `next/font`の活用（フォント最適化）
- Metadata APIの使用（SEO）
- Dynamic imports（`next/dynamic`）の活用

---

## React

### Hooks
- Hooksのルール順守（条件分岐内でのHook呼び出し禁止）
- `useEffect`の依存配列の正確性
- クリーンアップ関数の適切な実装
- カスタムHookの適切な分離

### メモ化
- `useMemo`/`useCallback`の適切な使用（不要なメモ化を避ける）
- `React.memo`の効果的な使用場面
- メモ化の依存配列の正確性

### 状態管理
- 適切なstate配置（lifting state up vs context vs 外部ライブラリ）
- 不必要な再レンダリングの回避
- フォーム状態の管理パターン
- サーバー状態 vs クライアント状態の分離

### コンポーネント設計
- 適切なコンポーネント分割（大きすぎるコンポーネントを避ける）
- Props設計の明確さ（propsの数が多すぎないか）
- Key propの適切な使用（リスト内で安定的なキー）
- Error Boundaryの配置
- childrenパターンの活用

### アクセシビリティ
- セマンティックHTML（`<button>`, `<nav>`, `<main>`等）
- ARIA属性の適切な使用
- キーボードナビゲーション対応
- alt属性の設定

---

## テスト

### React Testing Library
- `getByRole`, `getByText`等のアクセシブルなクエリを優先
- `getByTestId`の過度な使用を避ける
- ユーザーインタラクションには`userEvent`を使用
- `waitFor`/`findBy`による非同期処理のテスト

### APIモック
- MSW（Mock Service Worker）の活用
- テスト間のモック状態リセット
- 型安全なモック

### E2Eテスト
- Playwright/Cypressの適切な使用
- ページオブジェクトパターンの活用
- テストの独立性（他のテストに依存しない）

# Workout Tracker 実装計画（TASK）

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 広告のない筋トレ記録 PWA「Workout Tracker」を、段階 0〜3 の順に本番公開しながら作る。

**Architecture:** 端末側で動く React の SPA が Supabase（Auth・Postgres + RLS・Storage）と直接やり取りする。記録の操作は TanStack Query の楽観的更新と、IndexedDB に保存した未送信キューで扱い、圏外でも入力できる。計算は `src/domain/` の純粋関数にまとめ、権限は RLS とデータベース関数だけで守る。

**Tech Stack:** Vite + React + TypeScript、React Router、TanStack Query v5、Supabase JS v2、vite-plugin-pwa、Tailwind CSS、Recharts、Vitest、Vercel（Functions と定期実行）

**Spec:** [SPEC.md](SPEC.md)（「5.4」「R2」などの番号は SPEC.md の章と権限ルールを指す）

## この文書の読み方（発案者向け）

- 👤 の付いた作業は、発案者の操作が必要なもの。それ以外は Claude が実装する。
- 各段階の最後に「公開と確認」のタスクがあり、そこで実機の動作を確認してもらう。
- チェックボックスは進み具合の記録に使う。

## Global Constraints

- 言語は日本語のみ。重量の単位は kg のみ。
- 対応環境: iOS 17 以降の Safari とホーム画面に追加した状態、最新の Android Chrome。画面はスマホ縦向き基準、PC は最大幅 480px で中央に表示。
- 日付は端末の現地日付の文字列 `YYYY-MM-DD`（型 `DateStr`）。`Date#toISOString()` で日付を作らない（日本時間の 0〜9 時に前日になる）。
- すべてのテーブルに RLS を設定する。Supabase の秘密キーをリポジトリと `src/` 以下に置かない。公開キーは `VITE_SUPABASE_URL` と `VITE_SUPABASE_ANON_KEY`。
- 記録系の行（`workout_entries`、`workout_sets`）の id は端末で `crypto.randomUUID()` により作り、書き込みは upsert にする（再送しても結果が変わらない）。
- 推定1RM = 重量 × 回数 ÷ 40 ＋ 重量。回数 1 回は重量そのまま。表示は小数第1位。
- 入力範囲: 重量・加重 0〜999.99 kg（刻み 2.5）、回数 1〜999（刻み 1）、時間 1〜999 分の整数（刻み 1）、距離 0.01〜999.99 km（刻み 0.1）、体重 20.0〜300.0 kg（刻み 0.1）、メモ 500 文字、表示名 1〜20 文字、種目名とグループ名 1〜30 文字、1 グループ 20 人。
- 部位は `chest` `back` `shoulders` `arms` `legs` `abs` `cardio` `other` の固定 8 種。記録方法は `weight` `bodyweight` `cardio` の 3 種。
- データベースの変更は `supabase/migrations/` の SQL ファイルだけで行い、開発用 → 本番用の順に `npx supabase db push` で適用する。
- ブランチ: 段階ごとに `phase-0`〜`phase-3` を切り、タスクごとにコミットする。`main` への取り込みが本番公開になるので、各段階の「公開と確認」タスクでだけ `main` に入れる。
- 画面の見た目は Task 0.7 で選ばれた案のトークン（`src/styles/tokens.css`）に従う。

## Review Focus

仕様が直接は書いていないが、使う人が踏みやすい条件。各行のテストは、かっこ内のタスクに入れてある。

1. 日本時間の 0〜9 時に記録したとき、日付が前日にならない（Task 1.1 の `toDateStr`）。
2. 日本語キーボードの全角数字や「62,5」を入力したとき、数値として受け付けるか、はっきりエラーにする（Task 1.1 の `parseNumberInput`）。
3. 圏外で入力した操作が、アプリの再起動後や再送時に二重登録されない（Task 1.3 の再送テスト）。
4. アプリを開いたまま 0 時をまたいだとき、「今日」と未来日の判定が更新される（Task 1.6 の `useToday`）。
5. 招待リンクを未ログインで開いたとき、ログインと表示名の入力のあとに参加確認画面へ戻る（Task 3.2 の復帰テスト）。

## ファイル構成

```
api/keepalive.ts                 Supabase の一時停止を防ぐ定期処理（9.4）
vercel.json                      SPA の書き換えと定期実行の設定
supabase/migrations/             0001_base / 0002_records / 0003_groups / 0004_account の各 SQL
src/main.tsx, src/App.tsx        起動、ルーティング、下部タブ
src/lib/supabase.ts              Supabase クライアント
src/lib/date.ts                  DateStr の変換と計算
src/lib/queryClient.ts           キャッシュの永続化、未送信キュー
src/domain/                      純粋関数（types, numberInput, validation, setDefaults,
                                 e1rm, metrics, personalBest, calendar, exercisePicker, avatar）
src/features/auth/               ログイン、表示名、認証ガード
src/features/exercises/          種目の取得、部位と種目の選択、カスタム種目、種目の管理
src/features/workout/            種目記録とセットの取得・更新、入力画面
src/features/calendar/           カレンダー画面、月の表、その日の一覧
src/features/weight/             体重の入力、体重のグラフ
src/features/charts/             グラフの種目選択、種目のグラフ
src/features/groups/             グループ一覧、参加、グループ画面、個人ページ
src/features/settings/           設定とその下の画面
src/styles/tokens.css            配色などのトークン（ライト・ダーク、部位の色）
tests/rls/                       権限ルールの結合テスト（開発用プロジェクトに対して実行）
docs/operations.md               バックアップなど運用の手順
```

テストは、純粋関数と部品を実装ファイルの隣（`*.test.ts(x)`）に、権限ルールを `tests/rls/` に置く。

---

# 段階 0: 準備

### Task 0.1: 👤 外部サービスの準備

実装は含まない。終わったら値を Claude に伝える（秘密の値はチャットに貼らず、手元のファイルに書く）。

- [ ] Supabase で、無料枠に空きが 2 つあるか確認する。足りなければ Claude に相談する。
- [ ] Supabase でプロジェクトを 2 つ作る: `workout-tracker-dev` と `workout-tracker-prod`。リージョンは Tokyo。データベースのパスワードは控えておく。
- [ ] Google Cloud で OAuth クライアント（ウェブアプリケーション）を 2 つ作る（開発用、本番用）。リダイレクト URI は各 Supabase プロジェクトの `https://<ref>.supabase.co/auth/v1/callback`。同意画面は「外部」「本番環境」にする。
- [ ] 各 Supabase プロジェクトの Authentication で Google を有効にし、クライアント ID とシークレットを入れる。
- [ ] 開発用プロジェクトだけ、メールとパスワードのログインを有効にする（権限テスト用の利用者を作るため）。本番用では無効のままにする。
- [ ] ターミナルで `npx supabase login` を実行する。
- [ ] プロジェクト直下に `.env.local`（開発用の URL と公開キー）と `.env.test.local`（開発用の URL、公開キー、秘密キー）を作る。項目名は Task 0.2 の `.env.example` に合わせる。

完了条件: 2 つのプロジェクトの URL と公開キーがそろい、開発用プロジェクトのログイン画面で Google を選べる。

### Task 0.2: アプリの土台

**Files:**
- Create: `package.json`、`vite.config.ts`、`tsconfig.json`、`index.html`、`src/main.tsx`、`src/App.tsx`、`src/styles/tokens.css`、`.env.example`、`.gitignore`
- Test: `src/App.test.tsx`

**Interfaces:**
- Produces: npm スクリプト `dev` `build` `test`（Vitest、jsdom）`test:rls`（`tests/rls` だけを実行）`lint`。`.env.example` の項目は `VITE_SUPABASE_URL` `VITE_SUPABASE_ANON_KEY`、テスト用の `SUPABASE_TEST_URL` `SUPABASE_TEST_ANON_KEY` `SUPABASE_TEST_SERVICE_KEY`。

- [ ] Step 1: Vite（React + TypeScript）で土台を作り、React Router、TanStack Query、Tailwind CSS、Vitest、Testing Library を入れる。
- [ ] Step 2: `src/App.test.tsx` に「`<App />` を描画するとアプリ名 `Workout Tracker` が表示される」テストを書き、失敗を確認する。
- [ ] Step 3: `App` を実装し、`npm test` と `npm run build` が通ることを確認する。
- [ ] Step 4: `.gitignore` に `.env*.local` が入っていることを `git check-ignore .env.test.local` で確認し、コミットする。

### Task 0.3: データベースの基礎と権限テストの土台

**Files:**
- Create: `supabase/config.toml`、`supabase/migrations/0001_base.sql`、`src/lib/supabase.ts`、`tests/rls/helpers.ts`
- Test: `tests/rls/base.test.ts`

**Interfaces:**
- Produces（SQL）: テーブル `profiles(id uuid pk → auth.users on delete cascade, display_name text null check(1〜20文字), avatar_path text null, created_at)`、`keepalive(id int pk check(id = 1), pinged_at timestamptz)`。関数 `keepalive_ping() returns void`（security definer、`anon` に実行を許可、`keepalive` の 1 行を `now()` で upsert）。新規ユーザー作成時に `profiles` の行を作るトリガー。
- Produces（TS）: `supabase`（`src/lib/supabase.ts` の既定のクライアント）。`tests/rls/helpers.ts` の `createTestUser(label: string): Promise<{ id: string; client: SupabaseClient }>`、`anonClient(): SupabaseClient`、`cleanupTestUsers(): Promise<void>`（秘密キーでテスト利用者を作成・削除する）。

- [ ] Step 1: `tests/rls/base.test.ts` を書く。
  - 未ログインのクライアントで `profiles` を読むと 0 件（R11）。
  - 利用者 A は自分の `profiles` を読めて `display_name` を更新できる。利用者 B の行は読めず、更新しても 0 件（R1、R5）。
  - 未ログインのクライアントで `rpc('keepalive_ping')` が成功し、返り値が `null`。`keepalive` テーブルは未ログインでも利用者でも読めない（R11）。
- [ ] Step 2: `npm run test:rls` で失敗を確認する。
- [ ] Step 3: `0001_base.sql` を書き、開発用に `npx supabase db push` で適用する。`profiles` の方針は「本人だけが select と update」。
- [ ] Step 4: `npm run test:rls` が通ることを確認し、コミットする。

### Task 0.4: ログインと表示名

**Files:**
- Create: `src/features/auth/AuthProvider.tsx`、`LoginPage.tsx`、`DisplayNamePage.tsx`、`RequireAuth.tsx`、`src/features/auth/profile.ts`
- Modify: `src/App.tsx`
- Test: `src/features/auth/RequireAuth.test.tsx`、`DisplayNamePage.test.tsx`

**Interfaces:**
- Produces: `useAuth(): { status: 'loading' | 'signedOut' | 'signedIn'; userId: string | null; signInWithGoogle(returnTo?: string): Promise<void>; signOut(): Promise<void> }`。`useProfile(): UseQueryResult<{ id: string; displayName: string | null; avatarPath: string | null }>`。`useUpdateDisplayName()`。ルート `/login`、`/welcome`（表示名の入力）。
- `signInWithGoogle(returnTo)` は、ログイン後に戻る先を `sessionStorage` の `returnTo` に保存する（Task 3.2 が使う）。

- [ ] Step 1: テストを書く。
  - 未ログインで `/` を開くと `/login` に移る。
  - ログイン済みで `displayName` が `null` なら `/welcome` に移る。
  - `/welcome` で空文字と 21 文字はエラーになり保存されない。20 文字は保存され `/` に移る。
  - 表示名の入力欄の初期値は空（Google の名前を入れない）。
- [ ] Step 2: 失敗を確認し、実装する。ログイン画面はアプリ名と「Google でログイン」だけを置く。
- [ ] Step 3: テストが通ることと、`npm run dev` で開発用プロジェクトに Google ログインできることを確認し、コミットする。

### Task 0.5: Vercel への公開と一時停止対策

**Files:**
- Create: `api/keepalive.ts`、`vercel.json`
- Test: `api/keepalive.test.ts`

**Interfaces:**
- Consumes: SQL の `keepalive_ping()`（Task 0.3）。
- Produces: `GET /api/keepalive`。環境変数 `CRON_SECRET`、`KEEPALIVE_TARGETS`（`URL|公開キー` をカンマ区切りで並べたもの。本番用と開発用の 2 件）。`vercel.json` に SPA の書き換え（`/api/` 以外を `index.html` へ）と、1 日 1 回の定期実行（`0 18 * * *`、日本時間の午前 3 時台）を入れる。

- [ ] Step 1: `api/keepalive.test.ts` を書く（`fetch` は差し替える）。
  - `Authorization: Bearer <CRON_SECRET>` がない、または違うと 401 を返し、Supabase を呼ばない。
  - 正しいヘッダーなら、`KEEPALIVE_TARGETS` の各プロジェクトの `/rest/v1/rpc/keepalive_ping` を 1 回ずつ POST し、200 と `{ ok: true, results: [...] }` を返す。
  - 片方が失敗しても、もう片方は呼ぶ。結果は 500 と `{ ok: false, results: [...] }`。
- [ ] Step 2: 失敗を確認し、実装してテストを通す。
- [ ] Step 3: 本番用プロジェクトにマイグレーションを適用する（`npx supabase link` で本番用を選び `npx supabase db push`）。
- [ ] Step 4: 👤 Vercel で GitHub の `workout-tracker` を取り込み、環境変数（本番用の `VITE_SUPABASE_URL` `VITE_SUPABASE_ANON_KEY`、`CRON_SECRET`、`KEEPALIVE_TARGETS`）を設定する。発行された URL を、本番用 Supabase の Site URL とリダイレクト URL に登録する。
- [ ] Step 5: `phase-0` を `main` に取り込み、本番の URL でログインできること、Vercel の画面で定期実行が登録されていること、手動実行後に本番用と開発用の `keepalive.pinged_at` が更新されることを確認する。

### Task 0.6: PWA 化

**Files:**
- Modify: `vite.config.ts`、`index.html`
- Create: `public/icons/`（仮のアイコン 192px、512px、maskable、apple-touch-icon）

**Interfaces:**
- Produces: マニフェスト（`name: "Workout Tracker"`、`short_name: "Workout"`（仮。Task 0.8 で確定）、`display: "standalone"`、`lang: "ja"`）。サービスワーカーはアプリ本体を事前に保存し、新しい版は次回の起動で反映する。`/api/` と Supabase への通信は保存しない。

- [ ] Step 1: vite-plugin-pwa を設定する。
- [ ] Step 2: `npm run build && npm run preview` で、ブラウザの開発者ツールにマニフェストとサービスワーカーが表示され、通信を切っても再読み込みでアプリが開くことを確認し、コミットする。

### Task 0.7: 見た目の案と選択

**Files:**
- Create: `design/option-a.html`、`option-b.html`、`option-c.html`（静的な試作。本体には含めない）
- Modify: `src/styles/tokens.css`

- [ ] Step 1: frontend-design スキルを使い、カレンダー画面・入力画面・グラフ画面の 3 画面を、方向の違う 3 案で作る。各案にライトとダーク、部位 8 色、凡例を含める。内容は SPEC.md の 5.4〜5.6 の画面例に合わせる。
- [ ] Step 2: 👤 3 案をスマホで見比べて 1 つ選ぶ。部位の色もここで確定する。
- [ ] Step 3: 選ばれた案の色・文字・余白を `tokens.css` に定義する（`--bp-chest` など部位 8 色を含む。ライトとダークの両方）。SPEC.md の付録 B と 13 章を、決まった内容に更新する。コミットする。

### Task 0.8: 👤 実機の確認

- [ ] iPhone の Safari で本番の URL を開き、Google でログインできる。
- [ ] iPhone でホーム画面に追加し、そこから開いてログインできる。閉じて開き直してもログインが保たれる。
- [ ] Android の Chrome で同じ 2 点を確認する。
- [ ] ホーム画面のアイコンの下の名前を見て、短い名前を決める。Claude がマニフェストに反映する。

完了条件: 4 点すべてが通る。通らない項目があれば、段階 1 に進む前に直す。

---

# 段階 1: 記録とカレンダー

### Task 1.1: 型と入力まわりの純粋関数

**Files:**
- Create: `src/domain/types.ts`、`src/lib/date.ts`、`src/domain/numberInput.ts`、`src/domain/validation.ts`、`src/domain/setDefaults.ts`
- Test: それぞれの `*.test.ts`

**Interfaces（Produces）:**

```ts
// types.ts
type BodyPart = 'chest' | 'back' | 'shoulders' | 'arms' | 'legs' | 'abs' | 'cardio' | 'other'
type ExerciseKind = 'weight' | 'bodyweight' | 'cardio'
type DateStr = string // 'YYYY-MM-DD'
interface Exercise { id: string; ownerId: string | null; name: string; bodyPart: BodyPart; kind: ExerciseKind; sortOrder: number; deletedAt: string | null }
interface SetValues { weightKg: number | null; reps: number | null; durationMin: number | null; distanceKm: number | null }
interface WorkoutSet extends SetValues { id: string; entryId: string; position: number }
interface WorkoutEntry { id: string; userId: string; date: DateStr; exerciseId: string; createdAt: string; sets: WorkoutSet[]; memo: string | null }
const BODY_PARTS: readonly BodyPart[]            // 上の順
const BODY_PART_LABELS: Record<BodyPart, string> // 胸 背中 肩 腕 脚 腹 有酸素 その他

// date.ts
toDateStr(d: Date): DateStr                       // 端末の現地日付
today(): DateStr
addDays(date: DateStr, n: number): DateStr
daysBetween(from: DateStr, to: DateStr): number
formatRelativeDay(date: DateStr, base: DateStr): string // '今日' '昨日' 'N日前'
formatMonthDay(date: DateStr): string                    // '11/17(火)'

// numberInput.ts
parseNumberInput(text: string): number | null     // 全角数字と全角ピリオドは半角に直す。それ以外は null

// validation.ts
const LIMITS                                       // Global Constraints の範囲と刻み
isCompleteSet(kind: ExerciseKind, v: SetValues): boolean
stepValue(field: 'weightKg' | 'reps' | 'durationMin' | 'distanceKm' | 'bodyWeightKg', current: number | null, direction: 1 | -1): number

// setDefaults.ts
nextSetDefaults(previous: SetValues[] | null, current: SetValues[]): SetValues | null
```

- [ ] Step 1: テストを書く。主な検証値:

```ts
expect(toDateStr(new Date(2026, 10, 17, 0, 30))).toBe('2026-11-17')   // 日本時間 0:30 でも当日
expect(toDateStr(new Date(2026, 10, 17, 23, 59))).toBe('2026-11-17')
expect(addDays('2028-02-28', 1)).toBe('2028-02-29')
expect(formatRelativeDay('2026-11-14', '2026-11-17')).toBe('3日前')
expect(formatMonthDay('2026-11-17')).toBe('11/17(火)')

expect(parseNumberInput('62.5')).toBe(62.5)
expect(parseNumberInput('６２．５')).toBe(62.5)
expect(parseNumberInput('62,5')).toBeNull()
expect(parseNumberInput('')).toBeNull()
expect(parseNumberInput('abc')).toBeNull()

expect(isCompleteSet('weight', { weightKg: 0, reps: 1, durationMin: null, distanceKm: null })).toBe(true)
expect(isCompleteSet('weight', { weightKg: 60, reps: null, durationMin: null, distanceKm: null })).toBe(false)
expect(isCompleteSet('weight', { weightKg: 1000, reps: 5, durationMin: null, distanceKm: null })).toBe(false)
expect(isCompleteSet('weight', { weightKg: 60.125, reps: 5, durationMin: null, distanceKm: null })).toBe(false) // 小数第3位
expect(isCompleteSet('bodyweight', { weightKg: null, reps: 10, durationMin: null, distanceKm: null })).toBe(true)
expect(isCompleteSet('cardio', { weightKg: null, reps: null, durationMin: 30, distanceKm: null })).toBe(true)
expect(isCompleteSet('cardio', { weightKg: null, reps: null, durationMin: 30.5, distanceKm: 5 })).toBe(false)

expect(stepValue('weightKg', 60, 1)).toBe(62.5)
expect(stepValue('weightKg', 0, -1)).toBe(0)
expect(stepValue('reps', 1, -1)).toBe(1)
expect(stepValue('bodyWeightKg', 70.4, 1)).toBe(70.5) // 浮動小数の誤差を出さない

const s = (w: number, r: number) => ({ weightKg: w, reps: r, durationMin: null, distanceKm: null })
expect(nextSetDefaults([s(60, 10), s(60, 8)], [])).toEqual(s(60, 10))          // 前回の同じ番号
expect(nextSetDefaults([s(60, 10)], [s(62.5, 8)])).toEqual(s(62.5, 8))         // 前回に2番目がなければ今回の直前
expect(nextSetDefaults(null, [s(40, 12)])).toEqual(s(40, 12))
expect(nextSetDefaults(null, [])).toBeNull()                                   // 空欄
```

- [ ] Step 2: 失敗を確認し、実装してテストを通す。コミットする。

### Task 1.2: 記録のテーブルと権限（本人のみ）

**Files:**
- Create: `supabase/migrations/0002_records.sql`
- Test: `tests/rls/records.test.ts`

**Interfaces（Produces、SQL）:**
- 型 `body_part`、`exercise_kind`（Global Constraints の値）。
- `exercises(id uuid pk, owner_id uuid null → profiles, name text, body_part, kind, sort_order int, deleted_at timestamptz null, created_at)`。制約: `body_part = 'cardio'` と `kind = 'cardio'` は同時にだけ成り立つ。本人のカスタム種目の名前は重複不可（削除済みを含む）。デフォルト種目と同じ名前のカスタム種目はトリガーで拒否。記録がある種目の `kind` 変更はトリガーで拒否。
- `hidden_exercises(user_id, exercise_id, pk(user_id, exercise_id))`
- `workout_entries(id uuid pk, user_id, date date, exercise_id → exercises, created_at, updated_at, unique(user_id, date, exercise_id))`
- `workout_sets(id uuid pk, entry_id → workout_entries on delete cascade, user_id, position int, weight_kg numeric(5,2) null, reps int null, duration_min int null, distance_km numeric(5,2) null, created_at, updated_at)`。範囲は check 制約で Global Constraints に合わせる。
- `entry_memos(entry_id pk → workout_entries on delete cascade, user_id, body text check(1〜500文字))`
- `body_weights(user_id, date date, weight_kg numeric(4,1) check(20.0〜300.0), pk(user_id, date))`
- 関数 `exercise_last_used() returns table(exercise_id uuid, last_date date)`（呼び出した本人の、セットが 1 つ以上ある種目記録だけを数える）。
- デフォルト種目 42 件の投入（SPEC.md 付録 A の順に `sort_order` を振る。`owner_id` は null）。
- 方針: デフォルト種目はログイン済みなら誰でも select。それ以外のテーブルは、この時点では本人だけが select・insert・update・delete。`user_id` を他人の id にした書き込みは拒否。

- [ ] Step 1: `tests/rls/records.test.ts` を書く。
  - ログイン済みの利用者はデフォルト種目を 42 件読める。未ログインは 0 件。デフォルト種目の更新・削除は拒否される。
  - A がカスタム種目・種目記録・セット・メモ・体重・非表示設定を作成できる。B からは 1 件も読めず、更新・削除しても 0 件（R1、R3）。
  - A が `user_id` を B にして種目記録や体重を作ろうとすると拒否される。
  - 同じ id のセットを 2 回 upsert しても 1 行のまま。
  - 同じ日・同じ種目の種目記録を別の id で作ると拒否される。
  - `cardio` の部位に `weight` の種目を作ると拒否される。デフォルト種目と同じ名前のカスタム種目は拒否される。記録のある種目の `kind` 変更は拒否される。
  - 回数 0、重量 1000、体重 19.9、体重 300.1、メモ 501 文字は拒否される。
  - `exercise_last_used()` は、セットのある種目記録の最新日だけを返し、セットのない種目記録は数えない。
- [ ] Step 2: 失敗を確認し、SQL を書いて開発用に適用する。テストを通してコミットする。

### Task 1.3: 端末への保存と未送信キュー

**Files:**
- Create: `src/lib/queryClient.ts`、`src/lib/offline.ts`、`src/features/settings/PendingBadge.tsx`
- Test: `src/lib/offline.test.ts`

**Interfaces（Produces）:**

```ts
// queryClient.ts
queryClient: QueryClient                       // gcTime は 30 日。キャッシュと未送信の操作を IndexedDB に保存
persistOptions                                 // PersistQueryClientProvider に渡す。復元後に resumePausedMutations を呼ぶ
registerMutation<TVars>(key: string, fn: (vars: TVars) => Promise<void>): void
                                               // 再起動後も再送できるよう setMutationDefaults に登録する。
                                               // すべて scope: { id: 'records' } にして、登録順に 1 件ずつ送る
// offline.ts
usePendingCount(): number                      // 未送信の操作の件数
useIsOnline(): boolean
isRejectedByServer(error: unknown): boolean    // 権限エラー、制約違反、対象なし（HTTP 4xx。401 と 408 と 429 は除く）
```

- 送信に失敗したとき: `isRejectedByServer` が true なら、その操作を捨てて「送信できなかった記録があります」と知らせ、関係するキャッシュを取り直す。それ以外（通信エラー、401、5xx）は未送信のまま残して再送する。
- `PendingBadge` は件数が 1 以上のときだけ「未送信 n 件」を表示する。Task 1.6 が画面上部に置く。

- [ ] Step 1: テストを書く（Supabase への送信は差し替える）。
  - 圏外の状態で登録済みの操作を 3 件実行すると、`usePendingCount()` が 3 になり、送信関数は呼ばれない。
  - 通信が戻ると、実行した順に 1 件ずつ送信され、件数が 0 になる。
  - キャッシュを保存して作り直した `QueryClient` に復元すると、未送信の 3 件が残っていて、同じ順に送信される（再起動に当たる）。
  - 2 件目がサーバーに拒否されても、3 件目は送信される。拒否された操作は残らない。
  - 2 件目が通信エラーなら、2 件目と 3 件目は未送信のまま残る。
- [ ] Step 2: 失敗を確認し、実装してテストを通す。コミットする。

### Task 1.4: 種目の選択と管理

**Files:**
- Create: `src/domain/exercisePicker.ts`、`src/features/exercises/queries.ts`、`BodyPartPicker.tsx`、`ExercisePicker.tsx`、`ExerciseForm.tsx`、`ManageExercisesPage.tsx`
- Test: `src/domain/exercisePicker.test.ts`、`src/features/exercises/ExerciseForm.test.tsx`

**Interfaces:**
- Consumes: `Exercise`、`BodyPart`、`daysBetween`、`formatRelativeDay`（Task 1.1）、`exercise_last_used()`（Task 1.2）。
- Produces:

```ts
// domain/exercisePicker.ts
orderExercises(all: Exercise[], lastUsed: Map<string, DateStr>, hiddenIds: Set<string>, bodyPart: BodyPart):
  { recent: Exercise[]; list: Exercise[] }
  // recent: その部位で使った日が新しい順に最大 5 件。list: デフォルト（sortOrder 順）→ カスタム（作成順）。
  // どちらにも、非表示と削除済みは含めない
lastTrainedByBodyPart(all: Exercise[], lastUsed: Map<string, DateStr>): Map<BodyPart, DateStr>
  // 削除済み・非表示の種目の記録も数える

// features/exercises/queries.ts
useExercises(): Exercise[]                        // デフォルト + 自分のカスタム（削除済みを含む）
useHiddenExerciseIds(): Set<string>
useExerciseLastUsed(): Map<string, DateStr>
useSaveCustomExercise()                           // { id?, name, bodyPart, kind }。同じ名前の削除済みがあれば deleted_at を外して復活
useDeleteCustomExercise()                         // deleted_at を入れる
useSetExerciseHidden()                            // { exerciseId, hidden }
```

- ルート: `/add/:date`（部位の選択）、`/add/:date/:bodyPart`（種目の選択）、`/settings/exercises`（種目の管理）、`/exercises/new`、`/exercises/:id/edit`。
- 種目の管理は圏外では操作できない。圏外のときは保存・削除・非表示の操作を無効にし、その旨を表示する。

- [ ] Step 1: `exercisePicker.test.ts` を書く。
  - 使った日が 6 種目あるとき、`recent` は新しい順の 5 件。
  - 非表示の種目と削除済みの種目は `recent` にも `list` にも出ない。
  - `list` はデフォルトが `sortOrder` 順、そのあとにカスタム。
  - 別の部位の種目は出ない。
  - `lastTrainedByBodyPart` は部位ごとの最新日を返し、記録のない部位はキーがない。削除済みの種目の記録も数える。
- [ ] Step 2: `ExerciseForm.test.tsx` を書く。
  - 部位に「有酸素」を選ぶと記録方法が「有酸素」に固定され、ほかの部位では「重量×回数」「自重」だけを選べる。
  - 名前が空、31 文字、デフォルト種目と同じ名前のときはエラーになり保存されない。
  - 記録のある種目の編集では、記録方法を変えられない。
- [ ] Step 3: 失敗を確認し、実装する。部位の選択では、各部位の横に `formatRelativeDay(最新日, today())` を出し、記録がなければ「—」を出す。
- [ ] Step 4: テストを通し、開発用プロジェクトで追加・編集・削除・復活・非表示を手で確認して、コミットする。

### Task 1.5: 入力画面

**Files:**
- Create: `src/features/workout/queries.ts`、`mutations.ts`、`EntryPage.tsx`、`SetRow.tsx`、`Stepper.tsx`、`PreviousRecord.tsx`
- Test: `src/features/workout/EntryPage.test.tsx`、`Stepper.test.tsx`

**Interfaces:**
- Consumes: Task 1.1 の型と関数、`registerMutation`（Task 1.3）、`useExercises`（Task 1.4）。
- Produces:

```ts
// queries.ts
useDayEntries(date: DateStr): WorkoutEntry[]                       // その日の種目記録。createdAt の昇順。sets は position 順
useEntry(date: DateStr, exerciseId: string): WorkoutEntry | null
usePreviousEntry(date: DateStr, exerciseId: string): WorkoutEntry | null // date より前でセットのある直近
useMonthEntries(month: string /* 'YYYY-MM' */): { date: DateStr; exerciseId: string }[] // セットのある種目記録だけ

// mutations.ts（すべて registerMutation で登録し、楽観的に画面へ反映する）
useUpsertSet()      // { entry: { id, date, exerciseId }, set: WorkoutSet }  種目記録がなければ一緒に作る
useDeleteSet()      // { entryId, setId }  残りの position を詰める
useSaveMemo()       // { entry: { id, date, exerciseId }, body: string }  空文字ならメモを消す
useDeleteEntry()    // { entryId }
// セットもメモもなくなった種目記録は、useDeleteSet と useSaveMemo の中で消す
```

- ルート: `/entry/:date/:exerciseId`。
- `Stepper` の props: `{ value: number | null; field: 'weightKg' | 'reps' | 'durationMin' | 'distanceKm' | 'bodyWeightKg'; unit: string; onChange(value: number | null): void }`。＋／− は `stepValue`、数字を押すと数値キーボード（`inputMode="decimal"`）で直接入力し、`parseNumberInput` で読む。押せる範囲は 44px 以上。

- [ ] Step 1: `Stepper.test.tsx` を書く: ＋で 60 → 62.5、直接入力「６５」で 65、「abc」ではエラー表示になり `onChange` が呼ばれない。
- [ ] Step 2: `EntryPage.test.tsx` を書く（データの取得と送信は差し替える）。
  - 前回の記録が 60×10、60×8 のとき、「セットを追加」を 1 回押すと 60 kg・10 回の行ができ、`useUpsertSet` が 1 回呼ばれる。2 回目は 60 kg・8 回。3 回目は直前の行と同じ値。
  - 前回の記録がなく今回も 0 行のとき、「セットを追加」で空欄の行ができ、送信は呼ばれない。重量と回数を入れた時点で 1 回呼ばれる。
  - 行を消すと残りの番号が 1 から連番になる。
  - 自重の種目では、最初は回数だけが出て、「加重」を押すと重量の欄が出る。
  - 有酸素の種目では、時間と距離の欄が出て、時間だけで保存される。
  - メモに 500 文字を超えて入力できない。
  - 上部に前回の日付・全セット・メモが出る。前回がなければその欄がない。
  - 未来の日付の URL を開くとカレンダーに戻る。
- [ ] Step 3: 失敗を確認し、実装する。値の変更は 500 ミリ秒まとめてから送る。
- [ ] Step 4: テストを通し、コミットする。

### Task 1.6: カレンダー画面と下部タブ

**Files:**
- Create: `src/domain/calendar.ts`、`src/features/calendar/CalendarPage.tsx`、`MonthGrid.tsx`、`DayPanel.tsx`、`BodyPartLegend.tsx`、`src/lib/useToday.ts`、`src/components/TabBar.tsx`
- Modify: `src/App.tsx`
- Test: `src/domain/calendar.test.ts`、`src/lib/useToday.test.ts`、`src/features/calendar/CalendarPage.test.tsx`

**Interfaces:**
- Consumes: `useMonthEntries`、`useDayEntries`（Task 1.5）、`useExercises`（Task 1.4）、`PendingBadge`（Task 1.3）。
- Produces:

```ts
// domain/calendar.ts
monthGrid(month: string): (DateStr | null)[][]   // 日曜始まりの週の配列。月の外は null
bodyPartsByDate(entries: { date: DateStr; exerciseId: string }[], exercises: Exercise[]): Map<DateStr, BodyPart[]>
                                                 // BODY_PARTS の順。重複なし
summarizeSets(kind: ExerciseKind, sets: WorkoutSet[]): string
                                                 // '62.5×8 / 60×8'、自重は '10 / 8'（加重ありは '+10×5'）、有酸素は '30分 5km'
// lib/useToday.ts
useToday(): DateStr                              // 日付が変わったら、また画面に戻ってきたら更新する
```

- ルート `/` がカレンダー。選択中の日付は URL の `?date=` に持つ。タブは「カレンダー」「グラフ」「グループ」「設定」（グラフとグループは、この段階では「準備中」と表示する）。

- [ ] Step 1: テストを書く。
  - `monthGrid('2026-11')` の 1 週目は `[ '2026-11-01', …, '2026-11-07' ]`（2026 年 11 月 1 日は日曜）、最後の週は `'2026-11-29'` `'2026-11-30'` のあとが `null`。
  - `bodyPartsByDate` は、同じ日に肩と胸の種目があれば `['chest', 'shoulders']` を返す。
  - `summarizeSets('weight', …)` が `'62.5×8 / 60×8'` を返す。
  - `useToday` は、時計を 23:59 から 0:01 に進めると翌日の値に変わる（偽の時計を使う）。
  - カレンダー画面を開くと今日が選ばれている。記録のある日には部位の色の点が出る。日付を押すと下の一覧がその日に変わる。未来の日付には「種目を追加」も体重の行も出ない。凡例に 8 部位の名前が出る。
  - 圏外で、端末に保存されていない月を開くと「オフラインのため表示できません」と出る。保存済みの月は圏外でも表示される。
- [ ] Step 2: 失敗を確認し、実装してテストを通す。コミットする。

### Task 1.7: 体重の入力

**Files:**
- Create: `src/features/weight/queries.ts`、`WeightRow.tsx`、`WeightEditor.tsx`
- Modify: `src/features/calendar/DayPanel.tsx`
- Test: `src/features/weight/WeightEditor.test.tsx`

**Interfaces:**
- Consumes: `Stepper`（Task 1.5）、`registerMutation`（Task 1.3）。
- Produces: `useBodyWeight(date: DateStr): number | null`、`usePreviousBodyWeight(date: DateStr): number | null`（date より前の直近）、`useSaveBodyWeight()`（`{ date, weightKg }`、upsert）、`useDeleteBodyWeight()`（`{ date }`）、`useBodyWeights(from: DateStr | null): { date: DateStr; weightKg: number }[]`（Task 2.3 が使う）。

- [ ] Step 1: テストを書く。
  - 未記録の日は「体重を記録」と出る。押すと入力欄が開き、初期値は直近の体重。直近がなければ空欄。
  - 値を変えずに「記録」を押すと、その値で `useSaveBodyWeight` が 1 回呼ばれる。「記録」を押すまでは呼ばれない。
  - 19.9 と 300.1 はエラーになり保存されない。70.45 は保存されない（0.1 単位）。
  - 記録済みの日は値が出て、開くと修正と削除ができる。
- [ ] Step 2: 失敗を確認し、実装してテストを通す。コミットする。

### Task 1.8: 設定の基本

**Files:**
- Create: `src/features/settings/SettingsPage.tsx`、`ThemeSetting.tsx`、`InstallGuidePage.tsx`、`src/lib/theme.ts`
- Test: `src/lib/theme.test.ts`、`src/features/settings/SettingsPage.test.tsx`

**Interfaces:**
- Produces: `useTheme(): { mode: 'system' | 'light' | 'dark'; setMode(mode): void }`（`localStorage` に保存し、`<html data-theme>` に反映する）。設定画面の項目は、表示名の変更、種目の管理、配色、ホーム画面に追加する手順（iPhone 用と Android 用）、未送信の件数、ログアウト。

- [ ] Step 1: テストを書く。
  - `system` のときは端末の設定に従い、`dark` に固定すると端末がライトでも `data-theme="dark"` になる。
  - 未送信が 2 件あるときにログアウトを押すと、「未送信の記録が 2 件あります。ログアウトすると失われます」と確認が出る。取り消すとログアウトしない。
  - ログアウトすると、端末に保存したキャッシュが消える。
- [ ] Step 2: 失敗を確認し、実装してテストを通す。コミットする。

### Task 1.9: 段階 1 の公開と確認

- [ ] Step 1: 本番用プロジェクトに `0002_records.sql` を適用する。`npm test`、`npm run test:rls`、`npm run build` を通し、`phase-1` を `main` に取り込む。
- [ ] Step 2: 👤 スマホで確認する。
  - 種目を追加し、セットを 3 つ記録できる。翌日以降に同じ種目を開くと前回の記録が出て、「セットを追加」で前回の値が入る。
  - 機内モードにして 2 セット記録し、アプリを完全に閉じる。開き直すと「未送信 2 件」と出る。機内モードを解除すると 0 件になり、別の端末やブラウザでも記録が見える。
  - カレンダーに部位の色の点が出て、過去の日付に記録を追加・修正・削除できる。
  - 体重を記録・修正できる。
- [ ] Step 3: 確認で出た直しを反映する。

---

# 段階 2: グラフ

### Task 2.1: 指標と自己ベストの計算

**Files:**
- Create: `src/domain/e1rm.ts`、`metrics.ts`、`personalBest.ts`
- Test: それぞれの `*.test.ts`

**Interfaces（Produces）:**

```ts
// e1rm.ts
estimate1RM(weightKg: number, reps: number): number      // 丸めない
formatKg(value: number): string                          // 小数第1位に四捨五入。'75.0'

// metrics.ts
type MetricKey = 'e1rm' | 'maxWeight' | 'volume' | 'totalReps' | 'maxReps' | 'maxAddedWeight' | 'duration' | 'distance'
metricsFor(kind: ExerciseKind, hasAddedWeight: boolean): MetricKey[]
  // weight: ['e1rm','maxWeight','volume','totalReps']
  // bodyweight: ['maxReps','totalReps'] + (hasAddedWeight ? ['maxAddedWeight'] : [])
  // cardio: ['duration','distance']       先頭が最初に表示する指標
METRIC_LABELS: Record<MetricKey, string>   // 推定1RM 最大重量 総ボリューム 総回数 最大回数 最大加重 時間 距離
dailyValue(metric: MetricKey, sets: SetValues[]): number | null
type Period = '1m' | '3m' | '6m' | '1y' | 'all'
periodStart(period: Period, today: DateStr): DateStr | null
chartSeries(metric: MetricKey, days: { date: DateStr; sets: SetValues[] }[], period: Period, today: DateStr):
  { points: { date: DateStr; value: number }[]; bestDate: DateStr | null }
  // bestDate: e1rm・maxWeight・maxReps のとき、全期間の最高値を最初に達成した日。期間の外なら null。ほかの指標は常に null

// personalBest.ts
type BestKind = 'maxWeight' | 'e1rm' | 'maxReps'
currentBests(kind: ExerciseKind, days: { date: DateStr; sets: WorkoutSet[] }[]): Partial<Record<BestKind, { setId: string; value: number }>>
  // 記録が 1 日分しかないときは空。同じ値なら、日付と position が早いほう
detectNewBests(kind: ExerciseKind, saved: WorkoutSet, savedDate: DateStr, days: { date: DateStr; sets: WorkoutSet[] }[]): { kind: BestKind; value: number }[]
  // saved 以外の全セットを上回ったものだけ。saved の日以外に記録がなければ空
```

- [ ] Step 1: テストを書く。主な検証値:

```ts
expect(estimate1RM(60, 10)).toBe(75)
expect(estimate1RM(100, 1)).toBe(100)
expect(formatKg(estimate1RM(62.5, 8))).toBe('75.0')
expect(formatKg(78.125)).toBe('78.1')

const sets = [s(60, 10), s(62.5, 8), s(55, 8)]
expect(dailyValue('e1rm', sets)).toBe(75)          // 60×10 と 62.5×8 がどちらも 75
expect(dailyValue('maxWeight', sets)).toBe(62.5)
expect(dailyValue('volume', sets)).toBe(1540)      // 600 + 500 + 440
expect(dailyValue('totalReps', sets)).toBe(26)
expect(dailyValue('distance', [{ weightKg: null, reps: null, durationMin: 30, distanceKm: null }])).toBeNull()
expect(periodStart('3m', '2026-11-17')).toBe('2026-08-17')
expect(periodStart('all', '2026-11-17')).toBeNull()
```

  - `chartSeries`: 全期間の最高が 6 か月前にあり期間が `3m` のとき `bestDate` は `null`。同じ最高値が 2 日あれば早い日。`volume` では常に `null`。
  - `currentBests`: 1 日分だけなら `{}`。2 日分あれば最大重量と推定1RM のセットの id を返す。自重は `maxReps` だけ。有酸素は `{}`。
  - `detectNewBests`: 過去最高が 60 kg のとき 62.5 kg を保存すると `maxWeight` を含む。同じ 60 kg では含まない。初めての日は `[]`。
- [ ] Step 2: 失敗を確認し、実装してテストを通す。コミットする。

### Task 2.2: 種目のグラフ

**Files:**
- Create: `src/features/charts/queries.ts`、`ChartListPage.tsx`、`ExerciseChartPage.tsx`、`MetricTabs.tsx`、`PeriodTabs.tsx`
- Modify: `src/App.tsx`、`src/features/workout/EntryPage.tsx`（グラフへのリンク）
- Test: `src/features/charts/ChartListPage.test.tsx`、`ExerciseChartPage.test.tsx`

**Interfaces:**
- Consumes: Task 2.1 の関数、`useExercises`、`useExerciseLastUsed`（Task 1.4）。
- Produces: `useExerciseHistory(exerciseId: string): { date: DateStr; sets: WorkoutSet[] }[]`（全期間、日付の昇順、セットのある日だけ）。ルート `/charts`、`/charts/exercise/:exerciseId`。`PeriodTabs` の props は `{ value: Period; onChange(p: Period): void }`（Task 2.3 も使う）。

- [ ] Step 1: テストを書く。
  - 一覧の先頭に「体重」があり、その下に、記録がある種目だけが部位ごとに並ぶ。非表示の種目と削除済みの種目も、記録があれば出る。記録のない種目は出ない。
  - 重量×回数の種目を開くと、最初の指標は「推定1RM」、期間は「3か月」。指標の選択肢は 4 つ。
  - 自重の種目で加重の記録がないと、選択肢は「最大回数」「総回数」の 2 つ。
  - 棒の数は、期間内でその種目を記録した日の数と同じ。
  - `bestDate` の棒だけ、強調用のクラスが付く。
  - 記録が 1 件もない期間では「この期間の記録はありません」と出る。
- [ ] Step 2: 失敗を確認し、Recharts の棒グラフで実装する。棒を押すと日付と値を出す。棒が多いときは横にスクロールできるようにする。
- [ ] Step 3: テストを通し、コミットする。

### Task 2.3: 体重のグラフ

**Files:**
- Create: `src/features/weight/WeightChartPage.tsx`
- Test: `src/features/weight/WeightChartPage.test.tsx`

**Interfaces:**
- Consumes: `useBodyWeights`（Task 1.7）、`periodStart`（Task 2.1）、`PeriodTabs`（Task 2.2）。
- Produces: ルート `/charts/weight`。

- [ ] Step 1: テストを書く。
  - 折れ線の点の数は、期間内の体重の記録の数と同じ。
  - 横軸は日付の間隔どおり（11/1、11/2、11/10 の 3 点なら、2 点目と 3 点目の間隔が 1 点目と 2 点目の 8 倍）。
  - 縦軸の範囲は、記録の最小値と最大値に余白を足した範囲で、0 から始まらない。
  - 記録が 1 件だけでも点が出る。0 件なら「体重の記録はありません」と出る。
- [ ] Step 2: 失敗を確認し、Recharts の折れ線グラフで実装する。点を押すと日付と値を出す。テストを通してコミットする。

### Task 2.4: 記録時の自己ベスト表示

**Files:**
- Modify: `src/features/workout/EntryPage.tsx`、`SetRow.tsx`
- Create: `src/components/Toast.tsx`
- Test: `src/features/workout/EntryPage.best.test.tsx`

**Interfaces:**
- Consumes: `currentBests`、`detectNewBests`、`formatKg`（Task 2.1）、`useExerciseHistory`（Task 2.2）。

- [ ] Step 1: テストを書く。
  - 過去最高が 60 kg×8 の種目で 62.5 kg×8 を保存すると、「最大重量を更新 62.5kg」と「推定1RMを更新 75.0kg」が出て、その行に自己ベストの印が付く。
  - その後に 65 kg×8 の行を足すと、印は新しい行に移る。
  - 初めて記録する種目では、何も出ない。
  - 履歴が端末にない状態（取得できていない）では、何も出ず、エラーにもならない。
  - 有酸素の種目では、何も出ない。
- [ ] Step 2: 失敗を確認し、実装してテストを通す。コミットする。

### Task 2.5: 段階 2 の公開と確認

- [ ] Step 1: `npm test`、`npm run build` を通し、`phase-2` を `main` に取り込む。
- [ ] Step 2: 👤 スマホで確認する: 種目のグラフで指標と期間を切り替えられる。体重が折れ線で出る。自己ベストを更新すると、記録時に表示され、グラフの棒が強調される。
- [ ] Step 3: 確認で出た直しを反映する。

---

# 段階 3: グループ

### Task 3.1: グループのテーブル・関数・権限

**Files:**
- Create: `supabase/migrations/0003_groups.sql`
- Test: `tests/rls/groups.test.ts`、`tests/rls/sharing.test.ts`

**Interfaces（Produces、SQL。関数はすべて security definer で、呼び出した本人を `auth.uid()` で判定する）:**
- `groups(id uuid pk, name text check(1〜30文字), created_at)`
- `group_members(group_id → groups on delete cascade, user_id → profiles on delete cascade, role text check in ('admin','member'), shares_records boolean, shares_weight boolean, joined_at, pk(group_id, user_id))`
- `group_invites(group_id pk → groups on delete cascade, token text unique, created_at)`。token は 32 文字以上の乱数。
- `can_view_records(target uuid) returns boolean`: 本人、または「両者が所属し、target の `shares_records` が true」のグループが 1 つでもある。
- `can_view_weight(target uuid) returns boolean`: 上と同じ考え方で `shares_weight` を見る。
- `shares_group_with(target uuid) returns boolean`: 共通のグループがある。
- `create_group(p_name text, p_shares_records boolean, p_shares_weight boolean) returns uuid`
- `get_invite_preview(p_token text) returns table(group_id uuid, name text, member_count int, admin_name text, already_member boolean)`。無効な token なら 0 行。
- `join_group(p_token text, p_shares_records boolean, p_shares_weight boolean) returns uuid`。無効な token は `invalid_invite`、20 人いれば `group_full` のエラー。すでにメンバーなら何も変えずに group_id を返す。
- `rotate_invite(p_group_id uuid) returns text`（管理者のみ）
- `set_sharing(p_group_id uuid, p_shares_records boolean, p_shares_weight boolean) returns void`（本人の行だけ）
- `leave_group(p_group_id uuid) returns void`: 管理者が抜けるときは `joined_at` が最も古い残りのメンバーを管理者にする。残りがいなければグループを消す。
- `remove_member(p_group_id uuid, p_user_id uuid) returns void`、`rename_group(p_group_id uuid, p_name text) returns void`、`delete_group(p_group_id uuid) returns void`（いずれも管理者のみ）
- `group_feed(p_group_id uuid, p_before date, p_limit int) returns table(user_id uuid, date date, body_parts body_part[], exercise_count int)`: そのグループで `shares_records` が true のメンバーの、セットのある種目記録を「メンバー×日付」でまとめる。`date < p_before`（null なら制限なし）、日付の降順、同じ日は最後の更新が新しい順。
- 方針の変更: `workout_entries` と `workout_sets` の select を `can_view_records(user_id)` に、`body_weights` の select を `can_view_weight(user_id)` に、`exercises` の select を「デフォルト種目、または `can_view_records(owner_id)`」に、`profiles` の select を「本人、または `shares_group_with(id)`」にする。`entry_memos` は本人だけのまま。`groups` と `group_members` の select は所属メンバーだけ。`group_invites` の select は管理者だけ。`groups`、`group_members`、`group_invites` への直接の insert・update・delete は誰にも許可しない（関数経由だけ）。

- [ ] Step 1: `tests/rls/groups.test.ts` を書く（利用者 A、B、C、D）。
  - A が `create_group` すると、A が管理者の行ができ、招待の行ができる。
  - B は A の招待 token を `group_invites` から読めない（R7）。`get_invite_preview(token)` ではグループ名・人数 1・A の表示名を読める（R5 の例外）。でたらめな token は 0 行。
  - B が `join_group(token, true, false)` で参加できる。同じ呼び出しを 2 回しても行は 1 つ。
  - C は `group_members` に B の行を直接 insert できない（R8）。C は所属していないグループの `groups` と `group_members` を読めない（R6）。
  - 20 人いるグループに 21 人目が参加しようとすると `group_full`。
  - A が `rotate_invite` すると、古い token の `get_invite_preview` は 0 行、`join_group` は `invalid_invite`。B が `rotate_invite` を呼ぶと拒否される。
  - B は `remove_member`、`rename_group`、`delete_group` を呼べない。A は呼べる（R9）。
  - B は `set_sharing` で自分の設定だけを変えられる。A が B の行を直接 update しても 0 件（R9）。
  - A が `leave_group` すると B が管理者になる。最後の 1 人が抜けるとグループが消える。
- [ ] Step 2: `tests/rls/sharing.test.ts` を書く（グループ G1 に A・B・C、G2 に B・D）。
  - B が G1 で記録を公開しているとき、A は B の種目記録・セット・カスタム種目の名前を読める（R2、R4）。B のメモは読めない（R3）。
  - B が G1 で体重を非公開にしているとき、A は B の体重を読めない。公開に変えると読める（R12）。記録を非公開・体重を公開にした場合、体重だけ読める。
  - B が G2 では記録を非公開にしているとき、D は B の記録を読めない。G1 にも入っている A は読める。
  - C（非公開）は A と B の記録を読める。A は C の記録を読めない。
  - A は B の記録を update・delete できない（R1）。
  - `group_feed(G1)` には、G1 で記録を公開している人の、セットのある日だけが出る。体重だけ記録した日は出ない。C（非公開）は出ない。G1 に入っていない D が呼ぶと 0 行。
  - `profiles` は、同じグループのメンバーの分だけ読める。D は A の表示名を読めない（R5）。どの経路でもメールアドレスは返らない（R10）。
  - B が G1 を抜けると、A は B の記録を読めなくなる。
- [ ] Step 3: 失敗を確認し、SQL を書いて開発用に適用する。Task 1.2 の `records.test.ts` も含めて `npm run test:rls` を通し、コミットする。

### Task 3.2: グループ一覧・作成・参加

**Files:**
- Create: `src/features/groups/queries.ts`、`GroupListPage.tsx`、`CreateGroupPage.tsx`、`JoinPage.tsx`、`SharingChoice.tsx`
- Modify: `src/App.tsx`、`src/features/auth/RequireAuth.tsx`
- Test: `src/features/groups/JoinPage.test.tsx`、`CreateGroupPage.test.tsx`、`src/features/auth/returnTo.test.tsx`

**Interfaces:**
- Consumes: Task 3.1 の関数、`signInWithGoogle(returnTo)`（Task 0.4）。
- Produces: `useMyGroups(): { id: string; name: string; memberCount: number; role: 'admin' | 'member' }[]`、`useInvitePreview(token)`、`useJoinGroup()`、`useCreateGroup()`。ルート `/groups`、`/groups/new`、`/join/:token`。`SharingChoice` の props は `{ records: boolean | null; weight: boolean; onChange(v: { records: boolean | null; weight: boolean }): void }`（記録は未選択の `null` から始まり、体重は `false` から始まる）。

- [ ] Step 1: テストを書く。
  - 未ログインで `/join/abc` を開くとログイン画面に移り、ログインして表示名を入力したあと `/join/abc` に戻る（Review Focus 5）。
  - 参加確認画面にグループ名・人数・管理者の表示名が出る。記録の公開を選ぶまで「参加する」は押せない。体重のスイッチは最初オフ。
  - 「公開すると全期間の記録の詳細がメンバーに見えます。メモは見えません。後からいつでも変更できます」と表示される。
  - 無効なリンクでは「この招待リンクは無効です」、満員では「このグループは満員です」と出る。すでにメンバーならグループ画面に移る。
  - グループの作成でも同じ選択が出て、名前が空または 31 文字だと作成できない。
  - 圏外では、作成と参加のボタンが無効になる。
- [ ] Step 2: 失敗を確認し、実装してテストを通す。コミットする。

### Task 3.3: グループ画面

**Files:**
- Create: `src/features/groups/GroupPage.tsx`、`Feed.tsx`、`MemberList.tsx`、`GroupSettingsPage.tsx`、`src/domain/avatar.ts`、`src/components/Avatar.tsx`
- Test: `src/features/groups/GroupPage.test.tsx`、`src/domain/avatar.test.ts`

**Interfaces:**
- Consumes: `group_feed` などの関数（Task 3.1）、`formatRelativeDay`（Task 1.1）。
- Produces: `useGroup(groupId)`（名前と、メンバーの `{ userId, displayName, avatarPath, role, sharesRecords, sharesWeight, joinedAt }[]`）、`useGroupFeed(groupId)`（30 件ずつの追加読み込み）、`useSetSharing()`、`useLeaveGroup()`、管理者用の `useRotateInvite()` `useRemoveMember()` `useRenameGroup()` `useDeleteGroup()`。`avatarInitial(displayName: string): string`（先頭の 1 文字。絵文字などの結合文字も 1 文字として扱う）、`avatarColor(userId: string): string`（同じ id なら常に同じ色）。`Avatar` の props は `{ userId: string; displayName: string; avatarUrl: string | null; size: number }`。ルート `/groups/:groupId`、`/groups/:groupId/settings`。

- [ ] Step 1: テストを書く。
  - 新着一覧は日付ごとの見出し（今日、昨日、M/D）でまとまり、各項目に表示名・部位の名前・種目数が出る。自分の記録も出る。
  - 「さらに読み込む」で次の 30 件を取り、30 件未満が返ったらボタンが消える。
  - メンバー一覧で、記録と体重の両方が非公開の人にだけ「非公開」と出る。体重だけ公開している人には出ない。
  - 記録と体重のスイッチを切り替えると、`set_sharing` がその値で呼ばれる。
  - 管理者には、グループ名の変更・招待リンクの表示と作り直し・メンバーを外す・グループの削除が出る。メンバーには出ない。
  - 招待リンクは `<現在のオリジン>/join/<token>` の形で、共有（`navigator.share`。なければコピー）できる。
  - 退会とグループの削除は確認が出る。
  - `avatarInitial('たろう')` は `'た'`、`avatarColor` は同じ id で同じ値。
- [ ] Step 2: 失敗を確認し、実装する。画面を開いたときと、引っ張って更新したときに取り直す。テストを通してコミットする。

### Task 3.4: メンバーの個人ページと、その日の詳細

**Files:**
- Create: `src/features/groups/MemberPage.tsx`、`MemberDayDetail.tsx`、`src/features/groups/memberQueries.ts`
- Modify: `src/features/calendar/MonthGrid.tsx`、`DayPanel.tsx`（読み取り専用で使えるようにする）
- Test: `src/features/groups/MemberPage.test.tsx`

**Interfaces:**
- Consumes: `MonthGrid`、`bodyPartsByDate`、`summarizeSets`（Task 1.6）、`useGroup`（Task 3.3）。
- Produces: `useMemberMonthEntries(userId, month)`、`useMemberDayEntries(userId, date)`（メモは取得しない）、`useMemberBodyWeight(userId, date)`。ルート `/groups/:groupId/members/:userId`（`?date=` で日付を選ぶ。新着一覧の項目はこの URL に日付を付けて開く）。`MonthGrid` と `DayPanel` に `readOnly?: boolean` を足す。

- [ ] Step 1: テストを書く。
  - 記録を公開しているメンバーでは、カレンダーに部位の色の点が出て、日付を押すと種目とセットが出る。「種目を追加」、セットの編集、メモは出ない。
  - 体重も公開していれば、その日の体重が数値で出る。体重が非公開なら、体重の行そのものが出ない。
  - 体重だけ公開しているメンバーでは、色の点は出ず、日付を押すと体重だけが出る。
  - 両方とも非公開のメンバーでは、「このメンバーは記録を公開していません」だけが出る。
  - グラフへのリンクはどこにも出ない。
- [ ] Step 2: 失敗を確認し、実装してテストを通す。コミットする。

### Task 3.5: アイコン画像

**Files:**
- Create: `supabase/migrations/0004_account.sql`（この Task では Storage の部分）、`src/domain/imageCrop.ts`、`src/features/settings/AvatarSetting.tsx`
- Modify: `src/components/Avatar.tsx`
- Test: `tests/rls/avatar.test.ts`、`src/domain/imageCrop.test.ts`

**Interfaces:**
- Produces（SQL）: 非公開のバケット `avatars`。パスは `<user_id>/avatar.webp`。書き込みと削除は本人だけ。読み取りは本人と、`shares_group_with(パスの先頭の user_id)` が true の人だけ。
- Produces（TS）: `cropRect(width: number, height: number): { sx: number; sy: number; size: number }`（中央の正方形）。`useUploadAvatar()`（256×256 の WebP にしてアップロードし、`profiles.avatar_path` を更新）、`useRemoveAvatar()`、`useAvatarUrl(avatarPath: string | null): string | null`（署名付き URL を 1 時間キャッシュ）。

- [ ] Step 1: テストを書く。
  - `cropRect(400, 300)` は `{ sx: 50, sy: 0, size: 300 }`、`cropRect(300, 400)` は `{ sx: 0, sy: 50, size: 300 }`。
  - A は自分のパスにアップロードでき、B のパスにはできない。同じグループの B は A の画像を読め、グループの違う D は読めない（R5）。
  - 画像を外すと、頭文字と色のアイコンに戻る。
- [ ] Step 2: 失敗を確認し、実装してテストを通す。コミットする。

### Task 3.6: アカウント削除とプライバシーポリシー

**Files:**
- Modify: `supabase/migrations/0004_account.sql`、`src/features/settings/SettingsPage.tsx`、`src/features/auth/LoginPage.tsx`
- Create: `src/features/settings/DeleteAccountPage.tsx`、`PrivacyPage.tsx`
- Test: `tests/rls/account.test.ts`、`src/features/settings/DeleteAccountPage.test.tsx`

**Interfaces:**
- Produces（SQL）: `delete_account() returns void`。管理者をしている各グループを `leave_group` と同じ規則で引き継ぐか消し、本人の `auth.users` の行を消す（ほかのテーブルは cascade で消える）。
- Produces（TS）: ルート `/settings/delete-account`、`/privacy`（未ログインでも開ける）。

- [ ] Step 1: 👤 プライバシーポリシーに載せる連絡先を決める。
- [ ] Step 2: テストを書く。
  - `delete_account()` のあと、その利用者の `profiles`・種目記録・セット・メモ・体重・カスタム種目・`group_members` の行が 0 件になる。
  - 管理者だったグループでは、参加が最も古い残りのメンバーが管理者になっている。1 人だけだったグループは消えている。
  - 削除画面では、表示名を入力して一致するまで「削除する」を押せない。圏外では押せない。削除の前にアイコン画像を消す。
  - ログイン画面と設定画面に、プライバシーポリシーへのリンクがある。
- [ ] Step 3: 失敗を確認し、実装する。プライバシーポリシーには、保存する情報（Google アカウントのメールアドレスと ID、表示名、アイコン画像、トレーニング記録、メモ、体重）、使う目的、見える範囲（5.7 の公開のルール）、削除の方法、連絡先を書く。テストを通してコミットする。

### Task 3.7: 運用の手順

**Files:**
- Create: `docs/operations.md`
- Modify: `README.md`

- [ ] Step 1: `docs/operations.md` に次の手順を書く: 手動バックアップ（`npx supabase db dump` でスキーマとデータを取り、リポジトリの外に保存する。月 1 回を目安）、バックアップからの戻し方、Supabase が一時停止したときの再開、定期実行が動いているかの確認（`keepalive.pinged_at` を見る）、マイグレーションの適用順（開発用 → 本番用）。
- [ ] Step 2: 手順どおりにバックアップを 1 回取り、ファイルができることを確認する。README に、アプリの説明、開発の始め方、この文書へのリンクを書く。コミットする。

### Task 3.8: 段階 3 の公開と確認

- [ ] Step 1: 本番用プロジェクトに `0003_groups.sql` と `0004_account.sql` を適用する。`npm test`、`npm run test:rls`、`npm run build` を通し、`phase-3` を `main` に取り込む。
- [ ] Step 2: 👤 2 つのアカウント（自分と、友達または自分の別の Google アカウント）で確認する。
  - グループを作り、招待リンクを送って、相手が公開・非公開を選んで参加できる。
  - 公開にした側の記録が、相手の新着一覧と個人ページに出る。メモは相手に見えない。
  - 記録や体重を非公開に切り替えると、相手から見えなくなる。
  - アイコン画像を設定でき、相手の画面にも出る。
  - テスト用のアカウントを削除でき、相手のグループ画面から消える。
- [ ] Step 3: 確認で出た直しを反映し、友達に URL を渡す。

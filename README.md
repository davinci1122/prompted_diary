# 指示日記

毎日0時に開く「指示」に従って一枚撮り、カレンダーに残す写真日記。全366種のお題。

写真と記録はクラウド（Supabase）に保存されるため、キャッシュを消してもログインし直せば復元されます。端末にはキャッシュが残るのでオフラインでも閲覧可能です。

---

## セットアップ手順

### 1. Supabase プロジェクトを作る（無料）

1. [supabase.com](https://supabase.com) でアカウント作成 → **New Project**
2. プロジェクト名・パスワード（DB用、このアプリでは直接使いません）・リージョンを設定して作成

### 2. テーブルとストレージを作る

ダッシュボード左メニュー **SQL Editor** を開き、以下のSQLを**そのまま**貼り付けて **Run** をクリック：

```sql
-- entries テーブル
create table if not exists public.entries (
  user_id uuid not null references auth.users(id) on delete cascade,
  date text not null,
  prompt_id integer,
  note text default '',
  photo_path text,
  thumb_path text,
  created_at timestamptz default now(),
  updated_at timestamptz default now(),
  primary key (user_id, date)
);

-- RLS（行レベルセキュリティ）を有効化
alter table public.entries enable row level security;

-- 自分のデータだけ読み書きできるポリシー
create policy "Users can manage own entries"
  on public.entries for all
  using (auth.uid() = user_id)
  with check (auth.uid() = user_id);

-- photos バケット（写真保存用）
insert into storage.buckets (id, name, public)
values ('photos', 'photos', false)
on conflict do nothing;

-- 自分のフォルダだけ読み書きできるポリシー
create policy "Users can manage own photos"
  on storage.objects for all
  using (bucket_id = 'photos' and (storage.foldername(name))[1] = auth.uid()::text)
  with check (bucket_id = 'photos' and (storage.foldername(name))[1] = auth.uid()::text);
```

### 3. メールリンク認証を有効にする

1. ダッシュボード左メニュー **Authentication → Providers**
2. **Email** を開き、以下を確認：
   - **Enable Email provider** : ON
   - **Confirm email** : ON（推奨）
   - **Enable email OTP (magic link)** : ON
3. 保存

### 4. サイトURLを設定する

1. **Authentication → URL Configuration**
2. **Site URL** に、GitHub Pagesで公開するURL（例: `https://yourname.github.io/shiji-diary/`）を設定
3. **Redirect URLs** にも同じURLを追加して保存

### 5. キーを貼り付ける

1. **Project Settings → Data API** を開く
2. **Project URL**（`https://xxxxx.supabase.co`）と **anon public** キーをコピー
3. `index.html` の上部にある以下の2行に貼り付けて保存：

```js
const SUPABASE_URL      = "https://xxxxx.supabase.co";
const SUPABASE_ANON_KEY = "eyJhbGci...（長い文字列）";
```

anon キーは公開前提の鍵です。GitHub に置いて問題ありません（データ保護はRLSが担います）。

### 6. デプロイ（GitHub Pages）

1. `index.html` と `README.md` をリポジトリにプッシュ
2. **Settings → Pages → Deploy from a branch** で `main` / `/ (root)` を選択して保存
3. 数分後に `https://yourname.github.io/リポジトリ名/` で公開されます

---

## 以前のバックアップからの移行

v1（ローカル専用版）で「書き出す」から保存した `.json` ファイルは、クラウド版の **設定 → データを読み込む** でそのまま取り込めます。写真はクラウドにアップロードされ、以後はログインすればどの端末でも見られます。

## 仕様の要点

- 指示366個を内蔵。西暦年をシードにした決定論的シャッフル（同じ年なら並びは不変）
- 登録は当日（0:00〜23:59）のみ。過ぎた日は空白確定
- 写真は登録時にクライアント側で圧縮（本体 長辺1600px / サムネ 240px、WebP優先）
- テーマ：昼（紙）/ 夜（真夜中）/ 自動。週の始まり：月曜（既定）/ 日曜
- ヘッダーの雲アイコンで同期状態を表示（✓ 同期済み / ↻ 同期中 / ☁ オフライン / ⚠ エラー）
- オンライン復帰時・アプリ再フォーカス時に自動で同期

詳細は同梱の仕様書を参照。

---
allowed-tools: Read, Edit, Write, Bash
description: "現在のプロジェクトの CLAUDE.md に carta 憲法を適用 / 同期する"
argument-hint: "[<path>] [--profile <names>] [--force]"
---

# /carta:apply

カレントリポジトリ（または指定パス）の `CLAUDE.md` に最新の carta 憲法を適用する。マーカー方式で **既存内容を保護** しつつ、`<!-- carta:begin -->` ～ `<!-- carta:end -->` の中身だけを最新版に同期する。

## 引数

`$ARGUMENTS`:
- 指定なし: カレントディレクトリ（git root を優先、無ければ `pwd`）の `CLAUDE.md` を対象にする
- `<path>`: 指定パス。ディレクトリなら `<path>/CLAUDE.md`、ファイルなら直接そのパス
- `--profile <names>` / `--profile=<names>`: 連結適用する追加プロファイル名をカンマ区切りで指定（例: `--profile typescript,flutter`）。アルファベット順に正規化されてマーカーに記録される
- `--profile=`（空指定の推奨形式）: プロファイルなし（デフォルト憲法のみ）に明示的に戻す
- 引数に `--profile` を指定しなかった場合は、**既存マーカーに記録されている profiles を引き継いで** 再適用する
- `--force`: マーカー内本文の sha が記録値と不一致（手で編集された痕跡）の場合でも上書きを許可する

`--profile ""` も word splitting 経由で空指定として扱うが、環境依存があるためユーザー向けには `--profile=` を推奨する。

## 安全要件（必ず守る）

1. **書き込み前に必ず diff をユーザーに提示し、承認を得てから書き込む**。一発適用しない。
2. **マーカー外のテキストは 1 byte も変更しない**。書き込み後に「マーカー外文字列が apply 前と完全一致するか」を再確認する。
3. **マーカーが破損している（begin/end の片方しか無い、順序が逆、複数ある）場合は自動修復せず、エラーで停止する**。ユーザーに手動修復を促す。
4. **マーカー内 sha 不一致は `--force` 無しでは上書きしない**。

## 手順

### Step 1. 憲法本体 + プロファイルの連結とハッシュ算出

```bash
ROOT="${CLAUDE_PLUGIN_ROOT:-$(pwd)}"
SRC="$ROOT/assets/constitution.md"
[ -f "$SRC" ] || { echo "constitution.md not found at $SRC" >&2; exit 1; }

# ファイルを末尾 LF も含めて完全に読む（command substitution の trailing LF strip 回避）
read_file_preserving_trailing() {
  local content
  content=$(cat "$1"; printf x)
  printf '%s' "${content%x}"
}

# 末尾 LF を「ちょうど 1 個」に正規化
normalize_trailing_lf_to_one() {
  local body
  body=$(cat; printf x)
  body=${body%x}
  while [ "${body}" != "${body%$'\n'}" ]; do
    body=${body%$'\n'}
  done
  printf '%s\n' "$body"
}

# 引数解析（PROFILES_INPUT_GIVEN / PROFILES_INPUT は Step 0 で確定済みとする）
# 既存マーカーから抽出した profiles 値は PROFILES_FROM_MARKER に入れる（未抽出時は空文字）
if [ "$PROFILES_INPUT_GIVEN" = "true" ]; then
  RAW="$PROFILES_INPUT"
else
  RAW="$PROFILES_FROM_MARKER"
fi

# 正規化: カンマ split → trim → 空除去 → sort -u → カンマ join
NORMALIZED=$(printf '%s' "$RAW" | tr ',' '\n' \
  | awk 'NF{gsub(/^[ \t]+|[ \t]+$/, ""); if (length($0)) print}' \
  | LC_ALL=C sort -u \
  | paste -sd, -)

# 各プロファイル名のフォーマット検証 + 存在検証
for p in $(printf '%s' "$NORMALIZED" | tr ',' ' '); do
  [ -z "$p" ] && continue
  case "$p" in
    *[!A-Za-z0-9_-]*) echo "ERROR: invalid profile name: $p" >&2; exit 1 ;;
  esac
  [ -f "$ROOT/assets/profiles/$p.md" ] || {
    echo "ERROR: assets/profiles/$p.md not found" >&2
    echo "       (--profile=正しい値 で上書きしてください)" >&2
    exit 1
  }
done

# 連結: 各 part は末尾 LF 1 個に正規化したまま、part 間に "\n---\n\n" (6 byte) を挿入
COMBINED=$(read_file_preserving_trailing "$SRC" | normalize_trailing_lf_to_one; printf x)
COMBINED=${COMBINED%x}
for p in $(printf '%s' "$NORMALIZED" | tr ',' ' '); do
  [ -z "$p" ] && continue
  PROFILE_BODY=$(read_file_preserving_trailing "$ROOT/assets/profiles/$p.md" | normalize_trailing_lf_to_one; printf x)
  PROFILE_BODY=${PROFILE_BODY%x}
  COMBINED="${COMBINED}"$'\n---\n\n'"${PROFILE_BODY}"
done

# sha は連結後バイト列に対して計算する（v0.1.0 互換: profiles 空時は cat constitution.md と byte-for-byte 一致）
SHORT_HASH=$(printf '%s' "$COMBINED" | (sha256sum 2>/dev/null || shasum -a 256) | awk '{print substr($1,1,7)}')

VERSION=$(node -p "require('$ROOT/.claude-plugin/plugin.json').version")
echo "carta v$VERSION sha=$SHORT_HASH profiles=${NORMALIZED:-(none)}"
```

> **不変条件**: `NORMALIZED` が空の場合、`COMBINED` のバイト列は `cat assets/constitution.md` と完全一致する。これにより `sha` も v0.1.0 と完全一致する（実機検証値: `f0c10c8`）。

### Step 2. 対象 CLAUDE.md の特定

```bash
ARG_PATH="<引数で渡された path、無ければ空>"
if [ -z "$ARG_PATH" ]; then
  TARGET_DIR=$(git rev-parse --show-toplevel 2>/dev/null || pwd)
  TARGET="$TARGET_DIR/CLAUDE.md"
elif [ -d "$ARG_PATH" ]; then
  TARGET="$ARG_PATH/CLAUDE.md"
else
  TARGET="$ARG_PATH"
fi
echo "target: $TARGET"
```

### Step 3. 既存 CLAUDE.md の状態判定

`TARGET` を読み、以下の 5 ケースに分岐:

| 状態 | 判定方法 | 次のステップ |
|------|---------|------------|
| 存在しない | `[ ! -f "$TARGET" ]` | Step 4a 新規作成 |
| 存在するがマーカー無し | begin / end の grep ヒット数がいずれも 0 | Step 4b 冒頭挿入 |
| マーカー両方あり | begin / end ともにヒット 1 | Step 4c 同期判定 |
| マーカー破損 | begin / end のヒット数が不揃い、順序逆転、または複数 | エラーで停止（手動修復を促す） |

検出例（`profiles=` 属性はオプショナル、行アンカーで保護）:

```bash
BEGIN_COUNT=$(grep -cE '^<!-- carta:begin v[^ ]+ sha=[0-9a-f]+( profiles=[A-Za-z0-9_,-]+)? -->$' "$TARGET" || true)
END_COUNT=$(grep -cE '^<!-- carta:end -->$' "$TARGET" || true)

if [ "$BEGIN_COUNT" != "$END_COUNT" ] || [ "$BEGIN_COUNT" -gt 1 ]; then
  echo "ERROR: marker broken (begin=$BEGIN_COUNT end=$END_COUNT). 手動で修復してください" >&2
  exit 1
fi
```

既存マーカーから profiles を引き継ぐ場合の抽出例（macOS BSD sed / Linux GNU sed 共通の `sed -nE` 形式を使う）:

```bash
PROFILES_FROM_MARKER=$(sed -nE 's/^<!-- carta:begin v[^ ]+ sha=[0-9a-f]+ profiles=([A-Za-z0-9_,-]+) -->.*/\1/p' "$TARGET" | head -1)
```

`profiles=` 属性が無いマーカー（v0.1.0 形式）は `PROFILES_FROM_MARKER=""` のままでよい（= プロファイルなしとして互換動作）。同様に version / sha も sed で抽出する:

```bash
EXIST_VERSION=$(sed -nE 's/^<!-- carta:begin v([^ ]+) sha=[0-9a-f]+( profiles=[A-Za-z0-9_,-]+)? -->.*/\1/p' "$TARGET" | head -1)
EXIST_SHA=$(sed -nE 's/^<!-- carta:begin v[^ ]+ sha=([0-9a-f]+)( profiles=[A-Za-z0-9_,-]+)? -->.*/\1/p' "$TARGET" | head -1)
```

### Step 4a. 新規作成

`profiles` が空ならマーカーに `profiles=` 属性を含めない（v0.1.0 と完全に同形）。

```
<!-- carta:begin v{VERSION} sha={SHORT_HASH} -->
{COMBINED 本文（末尾 LF 1 個保持）}
<!-- carta:end -->

## プロジェクト固有

（リポジトリ固有の規約・運用・テストコマンド等をここに記述）
```

`profiles` が空でない場合は begin マーカーに `profiles={NORMALIZED}` を付与:

```
<!-- carta:begin v{VERSION} sha={SHORT_HASH} profiles={NORMALIZED} -->
```

新規作成の場合は diff 提示の代わりに「新規ファイルの内容」を見せて承認を取る。

### Step 4b. 既存 CLAUDE.md の冒頭にマーカーブロックを挿入

既存内容の前に以下を挿入（最後に空行 1 行を入れる）。`profiles` 空 / 非空の出し分けは Step 4a と同じ:

```
<!-- carta:begin v{VERSION} sha={SHORT_HASH}[ profiles={NORMALIZED}] -->
{COMBINED 本文}
<!-- carta:end -->

```

既存内容は完全保持（マーカー外保護）。

### Step 4c. 同期判定

既存マーカーから version / sha / profiles / 本文を抽出する。解析用正規表現:

```
<!-- carta:begin v(\S+) sha=([0-9a-f]+)( profiles=([A-Za-z0-9_,-]+))? -->
```

`group(3)` が無ければ `profiles=` 属性なし（= 空リスト）として扱う。本文は begin の次行から end の前行まで。

判定:

```
needs_update = (
  EXIST_VERSION != VERSION
  or EXIST_SHA != SHORT_HASH
  or sorted(EXIST_PROFILES) != sorted(NORMALIZED_LIST)
  or EXIST_BODY != COMBINED
)
```

- **すべて一致** → 「最新です」と報告して終了。書き込まない（idempotent）。
- **EXIST_BODY が新版 COMBINED と異なるのに、現実の `EXIST_BODY` から再計算した sha が記録 `EXIST_SHA` と一致しない**（マーカー内が手で改変された痕跡） → 警告を出し、`--force` が指定されていれば上書き、無ければ停止。**「body 一致なら sha 不一致を許す」というフォールバックは採用しない**。
- **それ以外（バージョン / sha / profiles / 本文のいずれかが異なる、新版へ更新が必要）** → diff を提示して承認を得てから上書き。マーカーの version / sha / profiles も更新。

profiles の変化（追加 / 削除 / 順序のみの違い）も「不一致」として扱う。順序は正規化後に比較するので、順序のみの違いは事実上発生しない（保存時に常にソートするため）。

### Step 5. diff 提示と承認

書き込み前に、変更の要約を以下の形式で提示:

```
[carta:apply] {target}
  before: v{EXIST_VERSION} sha={EXIST_SHA} profiles={EXIST_PROFILES_OR_NONE}  ({X} 行)
  after:  v{VERSION} sha={SHORT_HASH} profiles={NORMALIZED_OR_NONE}            ({Y} 行)
  --- diff (マーカー内のみ) ---
  ...
```

ユーザーが承認したら次のステップへ進む。

### Step 6. 書き込みとマーカー外保護のアサーション

書き込み後、以下を検証:

1. begin / end マーカーが各 1 行ずつ存在する
2. マーカー外（begin より前のテキスト + end より後のテキスト）が apply 前と完全一致する
3. 新しいマーカーの version / sha / profiles が想定値と一致する

不一致なら直ちにロールバック（書き込み前のテキストを書き戻す）してエラー報告。

### Step 7. 完了報告

```
[carta:apply] OK
  target:   {TARGET}
  version:  v{EXIST_VERSION} → v{VERSION}   （新規時は v- → v{VERSION}）
  sha:      {EXIST_SHA} → {SHORT_HASH}
  profiles: {EXIST_PROFILES_OR_NONE} → {NORMALIZED_OR_NONE}
  changes:  {summary}
```

## エッジケース

| 状況 | 挙動 |
|------|------|
| `assets/constitution.md` が見つからない | エラーで停止。`${CLAUDE_PLUGIN_ROOT}` の確認を促す |
| 対象 CLAUDE.md が読めない（パーミッション） | エラーで停止 |
| 既存マーカーが複数ある | エラーで停止（v0.1.0 / v0.2.0 では root の 1 つだけが対象） |
| 既存マーカーの順序が逆（end が先・begin が後） | マーカー破損として停止 |
| version 文字列が不正 | エラーで停止（`<!-- carta:begin vX.Y.Z sha=abcdefg -->` を期待） |
| 改行コードが CRLF の CLAUDE.md | LF に正規化せず、書き込み時も既存の改行コードを保持する（マーカー外保護の一環） |
| `--profile foo` で `assets/profiles/foo.md` が無い | `assets/profiles/foo.md not found` を stderr に出力して停止（exit 1） |
| `--profile typescript,foo` のように一部不存在 | 1 件目の不存在で停止 |
| `--profile` の値に許容外文字（空白・`/`・`.` 等）含む | `invalid profile name: <name>`（許容: `[A-Za-z0-9_-]+`） |
| `--profile=`（明示の空） | プロファイル全外し。マーカーから `profiles=` 属性が消え、本文はデフォルト憲法のみになる |
| 既存マーカー `profiles=foo,bar` が指す `assets/profiles/foo.md` が存在しない | 通常のプロファイル不存在エラーで停止し、誘導メッセージで「`--profile=正しい値` で上書きしてください」と案内 |
| `--profile` 引数の値が次トークン未指定 | `--profile requires a value`（exit 1） |
| 既存マーカーの `profiles=` 値が許容外 | `marker profiles attribute malformed`（手動修復を促す、exit 1） |

## 実装メモ

- shell + Read / Edit / Write の組み合わせで実現。複雑なテンプレートエンジンは使わない
- マーカー判定の正規表現は `^<!-- carta:begin v[^ ]+ sha=[0-9a-f]+( profiles=[A-Za-z0-9_,-]+)? -->$`（行アンカー保持・`profiles=` オプショナル）。揺らぎは許容しない
- 連結区切りは 6 byte の `"\n---\n\n"`（前 part 末尾 LF と組み合わせて視覚的には「最終行 / 空行 / `---` / 空行 / 次 part 最初の行」になる）
- profiles アルファベット順正規化と sort -u（重複除去）は **引数解析時と書き出し時の両方** で行う（既存マーカーの値を信頼しない）
- `git rev-parse --show-toplevel` が空（git 未初期化）の場合は `pwd` にフォールバック
- 大文字小文字は厳密に一致させる（`Carta:Begin` 等は別物として扱う）
- profiles 本文の先頭 `<!-- carta-profile: NAME -->` コメントは **凍結された定数**（リリース後に書式を変更しない）

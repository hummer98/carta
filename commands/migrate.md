---
allowed-tools: Read, Edit, Write, Bash
description: "CLAUDE.md のマーカー外から、憲法本体と意味的に重複する記述を検出し削除候補を提示する"
argument-hint: "[<path>] [--profile <names>] [--include-medium] [--apply]"
---

# /carta:migrate

`carta` を導入する前から運用していたプロジェクトの `CLAUDE.md` には、`/carta:apply` でマーカーを差し込んだ後も、マーカー外に「日本語でドキュメントを書く」「テストを必ず実行する」など、**憲法と重複する記述** が残り続ける。`/carta:migrate` はそれを **検出して削除候補として提示する** 移行ツール。

- **削除のみ提案**（言い換え保持はしない）
- **マーカー内本文は 1 byte も変更しない**
- デフォルト dry-run。`--apply` 付きでも最終承認プロンプトを取る
- carta 導入直後に **一度走らせて掃除する** 用途を想定（定常運用ではない）

## 引数

`$ARGUMENTS`:
- 指定なし: カレントディレクトリ（git root を優先、無ければ `pwd`）の `CLAUDE.md` を対象にする
- `<path>`: 指定パス。ディレクトリなら `<path>/CLAUDE.md`、ファイルなら直接そのパス
- `--profile <names>` / `--profile=<names>`: 連結対象プロファイル名（apply と同じ引数規約）
- `--profile=`（空指定）: プロファイル全外し
- `--profile` 未指定時: **既存マーカーに記録された profiles を引き継ぐ**
- `--include-medium`: MEDIUM 確信度の候補も削除対象に含める（デフォルトは HIGH のみ）
- `--apply`: dry-run を解除して書き込みモードに入る（最終承認プロンプトは別途必須）

`--force` フラグは **受け取らない**。マーカー内本文が手で改変されている場合は `/carta:apply --force` で先に同期してから migrate せよ（責務分離）。

## 安全要件（必ず守る）

1. **デフォルト dry-run**。`--apply` 無しでは絶対に書き込まない（§ 安全要件 #1）
2. **マーカー内本文は 1 byte も変更しない**。書き込み後 assertion で `inside_after == inside_before` をバイト比較する（§ 安全要件 #2）
3. **マーカーが破損していたら自動修復しない**。エラーで停止し、ユーザーに手動修復を促す（§ 安全要件 #4。apply.md Step 3 と同じ判定）
4. **マーカー内 sha 不一致なら警告停止**。migrate は `--force` を取らない（§ 安全要件 #5）
5. **`--apply` 付きでも、削除候補提示後に y/n の最終承認**（§ 安全要件 #1）
6. **候補 0 件なら何もせず終了**（§ 安全要件 #9）
7. **コードブロック内は検出対象外**（§ 安全要件 #7）
8. **CLAUDE.md が git 管理されていない / `.gitignore` 配下なら警告**し、`--apply` の続行可否を確認する（§ 安全要件 #8）

## エッジケース

| 状況 | 挙動 |
|------|------|
| マーカー無し | 「先に `/carta:apply` を実行してください」と誘導して exit 1 |
| マーカー破損（begin/end 数不一致・順序逆・複数） | 「マーカー破損。手動で修復してください」と停止（exit 1） |
| マーカー内 sha 不一致 | 「`/carta:apply --force` で先に同期してください」と誘導して exit 1。migrate は `--force` を取らない |
| 候補 0 件 | 「マーカー外に重複は検出されませんでした」と報告して exit 0。書き込み・diff 出力なし |
| `--profile foo` の `assets/profiles/foo.md` 不在 | apply と同じエラー文言で exit 1 |
| CLAUDE.md が untracked / ignored | 「`git restore` で戻せない可能性があります」と警告。`--apply` 続行可否を y/n で確認 |
| マーカー外のコードブロック内に重複文 | コードブロックは検出対象外（item 抽出で skip）。候補に含めない |
| CRLF 改行の CLAUDE.md | 既存改行コードを保持。LF に正規化しない |
| `--apply` 付きでマーカー内 byte が変わる（バグ） | assertion で検出して書き込み前の状態にロールバック・exit 1 |

## しきい値定数（環境変数で上書き可能）

```bash
# 既定値。fixture 03 / 04 で実測した上で確定する（plan §4.4）。
: "${MIGRATE_OVERLAP_MEDIUM:=0.85}"   # token overlap >= ここで MEDIUM
: "${MIGRATE_OVERLAP_LLM:=0.50}"      # token overlap >= ここで Claude 本人による意味判定に回す
```

`[要調整]`: 想定外の値が観測された場合は `MIGRATE_OVERLAP_MEDIUM` / `MIGRATE_OVERLAP_LLM` を環境変数で上書きする（再リリース不要）。

## 手順

### Step 1. 連結本文と参照テキストの構築（apply.md Step 1 を流用）

`commands/*.md` 間で source 共有しない方針のため、apply.md Step 1 と同じ bash ロジックを **コピーで持つ** 。`COMBINED` / `SHORT_HASH` / `VERSION` / `NORMALIZED` を得る。

```bash
ROOT="${CLAUDE_PLUGIN_ROOT:-$(pwd)}"
SRC="$ROOT/assets/constitution.md"
[ -f "$SRC" ] || { echo "constitution.md not found at $SRC" >&2; exit 1; }

read_file_preserving_trailing() {
  local content
  content=$(cat "$1"; printf x)
  printf '%s' "${content%x}"
}

normalize_trailing_lf_to_one() {
  local body
  body=$(cat; printf x)
  body=${body%x}
  while [ "${body}" != "${body%$'\n'}" ]; do
    body=${body%$'\n'}
  done
  printf '%s\n' "$body"
}

# 引数解析（PROFILES_INPUT_GIVEN / PROFILES_INPUT は引数解析で確定済みとする）
# 既存マーカーから抽出した profiles 値は PROFILES_FROM_MARKER に入れる（未抽出時は空）
if [ "$PROFILES_INPUT_GIVEN" = "true" ]; then
  RAW="$PROFILES_INPUT"
  PROFILES_SOURCE="--profile"
else
  RAW="$PROFILES_FROM_MARKER"
  PROFILES_SOURCE="marker"
fi

NORMALIZED=$(printf '%s' "$RAW" | tr ',' '\n' \
  | awk 'NF{gsub(/^[ \t]+|[ \t]+$/, ""); if (length($0)) print}' \
  | LC_ALL=C sort -u \
  | paste -sd, -)

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

COMBINED=$(read_file_preserving_trailing "$SRC" | normalize_trailing_lf_to_one; printf x)
COMBINED=${COMBINED%x}
for p in $(printf '%s' "$NORMALIZED" | tr ',' ' '); do
  [ -z "$p" ] && continue
  PROFILE_BODY=$(read_file_preserving_trailing "$ROOT/assets/profiles/$p.md" | normalize_trailing_lf_to_one; printf x)
  PROFILE_BODY=${PROFILE_BODY%x}
  COMBINED="${COMBINED}"$'\n---\n\n'"${PROFILE_BODY}"
done

SHORT_HASH=$(printf '%s' "$COMBINED" | (sha256sum 2>/dev/null || shasum -a 256) | awk '{print substr($1,1,7)}')
VERSION=$(node -p "require('$ROOT/.claude-plugin/plugin.json').version")
echo "carta v$VERSION sha=$SHORT_HASH profiles=${NORMALIZED:-(none)} (from ${PROFILES_SOURCE})"
```

### Step 2. 対象 CLAUDE.md の特定（apply.md Step 2 を流用）

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
[ -f "$TARGET" ] || { echo "ERROR: $TARGET not found" >&2; exit 1; }
echo "target: $TARGET"
```

### Step 3. マーカー検出と sha 検証

apply.md Step 3 と同じ regex でマーカーを検出する。

```bash
BEGIN_RE='^<!-- carta:begin v[^ ]+ sha=[0-9a-f]+( profiles=[A-Za-z0-9_,-]+)? -->$'
END_RE='^<!-- carta:end -->$'

BEGIN_COUNT=$(grep -cE "$BEGIN_RE" "$TARGET" || true)
END_COUNT=$(grep -cE "$END_RE" "$TARGET" || true)

if [ "$BEGIN_COUNT" = "0" ] && [ "$END_COUNT" = "0" ]; then
  echo "ERROR: carta マーカーが見つかりません。" >&2
  echo "       先に '/carta:apply' を実行してください。" >&2
  exit 1
fi

if [ "$BEGIN_COUNT" != "$END_COUNT" ] || [ "$BEGIN_COUNT" -gt 1 ]; then
  echo "ERROR: marker broken (begin=$BEGIN_COUNT end=$END_COUNT). 手動で修復してください" >&2
  exit 1
fi

# begin / end の行番号と existing profiles を抽出
BEGIN_LINE=$(grep -nE "$BEGIN_RE" "$TARGET" | head -1 | cut -d: -f1)
END_LINE=$(grep -nE "$END_RE"   "$TARGET" | head -1 | cut -d: -f1)

# 順序逆転チェック
if [ "$BEGIN_LINE" -ge "$END_LINE" ]; then
  echo "ERROR: marker order broken (begin line=$BEGIN_LINE end line=$END_LINE)." >&2
  exit 1
fi

EXIST_VERSION=$(sed -nE 's/^<!-- carta:begin v([^ ]+) sha=[0-9a-f]+( profiles=[A-Za-z0-9_,-]+)? -->.*/\1/p' "$TARGET" | head -1)
EXIST_SHA=$(sed -nE 's/^<!-- carta:begin v[^ ]+ sha=([0-9a-f]+)( profiles=[A-Za-z0-9_,-]+)? -->.*/\1/p' "$TARGET" | head -1)
PROFILES_FROM_MARKER=$(sed -nE 's/^<!-- carta:begin v[^ ]+ sha=[0-9a-f]+ profiles=([A-Za-z0-9_,-]+) -->.*/\1/p' "$TARGET" | head -1)

# マーカー内本文（begin の次行 〜 end の前行）を抽出して sha を再計算。
# 注意: command substitution は trailing LF を 1 個剥がすため、apply.md が
# COMBINED の末尾 LF も含めて sha 計算しているのと整合させるべく
# printf '%s\n' で 1 個分の LF を補ってから hash する。
INSIDE_BODY=$(sed -n "$((BEGIN_LINE+1)),$((END_LINE-1))p" "$TARGET")
INSIDE_SHA=$(printf '%s\n' "$INSIDE_BODY" | (sha256sum 2>/dev/null || shasum -a 256) | awk '{print substr($1,1,7)}')

if [ "$INSIDE_SHA" != "$EXIST_SHA" ]; then
  echo "ERROR: マーカー内本文 sha が記録値と一致しません (recorded=$EXIST_SHA actual=$INSIDE_SHA)" >&2
  echo "       先に '/carta:apply --force' で同期してから migrate してください。" >&2
  exit 1
fi
```

### Step 4. マーカー外を item 単位に分解

§4 の `extract_items_tsv` を `outside_before` / `outside_after` に対して **独立に** 実行する。

```bash
# OUTSIDE_BEFORE: 1 行目 〜 BEGIN_LINE-1 行目
# OUTSIDE_AFTER:  END_LINE+1 行目 〜 ファイル末尾
OUTSIDE_BEFORE=$(sed -n "1,$((BEGIN_LINE-1))p" "$TARGET")
OUTSIDE_AFTER=$(sed  -n "$((END_LINE+1)),\$p" "$TARGET")
```

コードブロック内部は kind=code として記録し、後段で skip する。マーカー前後を跨ぐ集約は **絶対にしない**。

### Step 5. 重複判定（5a → 5b → 5c）

§4 を参照。

- **Step 5a. 正規化マッチング**: bash + Python で各 item を正規化し、参照側 norm セットと突合（HIGH 抽出）
- **Step 5b. LLM 判断（このコマンドを読んでいる Claude 自身が内部評価）**: 5a で決まらなかった item を §4.2 のプロンプト雛形に沿って判定。**外部プロセス（`claude` CLI 等）は呼ばない**
- **Step 5c. 見出し集約**: §2.2 の規則で `outside_before` / `outside_after` を独立に処理

### Step 6. 候補テーブル + unified diff の提示

§ 出力フォーマット参照。

### Step 7. `--apply` かつ最終承認 → 書き込み

§ 出力フォーマット 5.3 参照。最終承認 (`y`) を得たら、書き込み前の `TARGET` 全体を `BEFORE_FULL` 変数に保持し、削除候補を反映した `AFTER_FULL` を計算して `Write` する。

```bash
# 書き込み前の全体を変数に保持（ロールバック用）。git tracked / untracked いずれでも復元可能。
BEFORE_FULL=$(cat "$TARGET"; printf x)
BEFORE_FULL=${BEFORE_FULL%x}

# 削除候補に該当する item の行範囲を AFTER_FULL から除く
# （マーカー内 INSIDE_BODY は触らない）
# ... (詳細は実装側で組み立てる。ここでは概念のみ)
```

### Step 8. 書き込み後 assertion

書き込み完了後、必ず以下を検証する。失敗時はロールバック (`Write` で `BEFORE_FULL` を書き戻す) して `exit 1`。

```bash
# 1. begin / end が各 1 行ずつ存在する
NEW_BEGIN=$(grep -cE "$BEGIN_RE" "$TARGET" || true)
NEW_END=$(  grep -cE "$END_RE"   "$TARGET" || true)
[ "$NEW_BEGIN" = "1" ] && [ "$NEW_END" = "1" ] || {
  echo "FATAL: マーカー数が壊れた (begin=$NEW_BEGIN end=$NEW_END). ロールバックします" >&2
  printf '%s' "$BEFORE_FULL" > "$TARGET"
  exit 1
}

# 2. マーカー内本文の sha が apply 前と完全一致（byte-for-byte）
NEW_BEGIN_LINE=$(grep -nE "$BEGIN_RE" "$TARGET" | head -1 | cut -d: -f1)
NEW_END_LINE=$(  grep -nE "$END_RE"   "$TARGET" | head -1 | cut -d: -f1)
NEW_INSIDE=$(sed -n "$((NEW_BEGIN_LINE+1)),$((NEW_END_LINE-1))p" "$TARGET")
NEW_INSIDE_SHA=$(printf '%s\n' "$NEW_INSIDE" | (sha256sum 2>/dev/null || shasum -a 256) | awk '{print $1}')
OLD_INSIDE_SHA=$(printf '%s\n' "$INSIDE_BODY" | (sha256sum 2>/dev/null || shasum -a 256) | awk '{print $1}')
[ "$NEW_INSIDE_SHA" = "$OLD_INSIDE_SHA" ] || {
  echo "FATAL: マーカー内本文が変化しました (recorded=$OLD_INSIDE_SHA actual=$NEW_INSIDE_SHA). ロールバックします" >&2
  printf '%s' "$BEFORE_FULL" > "$TARGET"
  exit 1
}
```

---

## 検出アルゴリズム（Markdown + bash 風 擬似実装）

apply.md / show.md / diff.md と同じ粒度で記述する。**前提**: この `commands/migrate.md` を読んでいるのは Claude Code セッション内の Claude 本人である。「LLM 判断」と書かれている箇所は **Claude 本人がそのプロンプト雛形を内部評価し、結果を内部の候補配列として保持する**。bash から `claude` CLI 等の外部プロセスを呼び出すことは一切ない（`allowed-tools: Read, Edit, Write, Bash` の範囲を超えるため）。

### Step 5a. 正規化マッチング（bash でやる範囲）

```bash
# OUTSIDE_BEFORE / OUTSIDE_AFTER は Step 4 で抽出済みのマーカー外文字列
# COMBINED は Step 1 で得た連結本文（マーカー内本文に展開される予定のテキスト）

# 1. bash で「素朴な item」を抽出する。
#    完璧な Markdown 解析は bash では困難なので、以下の方針で粗く抽出する:
#    - 行頭が "[-*+] " / "\d+\. " で始まる連続行を 1 item
#    - 空行で区切られた段落を 1 item
#    - "^#+ " 行は heading として独立した item
#    - "```" で開く fenced code block は skip フラグを立てる
#    - 表（"|" を含む連続行）は 1 item として paragraph 扱い
#    抽出結果は 1 件 1 行の TSV に落とし、Claude 本人が解釈する:
#    id<TAB>side<TAB>kind<TAB>start_line<TAB>end_line<TAB>parent_heading_id<TAB>text_b64
#
#    "side" は "before" / "after" のいずれか。outside_before / outside_after を完全に独立に処理する。
#    "text_b64" は base64 で渡し、改行を含む item を 1 行に収める。

extract_items_tsv() {
  local side="$1" content="$2"
  printf '%s' "$content" | awk -v side="$side" '
    BEGIN {
      in_code = 0
      cur_heading_id = "-"
      next_id = 1
      buf = ""
      buf_start = 0
      buf_end = 0
      buf_kind = ""
    }
    function flush() {
      if (buf == "") return
      cmd = "printf %s " "\047" buf "\047" " | base64 | tr -d \"\\n\""
      cmd | getline b64
      close(cmd)
      printf("%d\t%s\t%s\t%d\t%d\t%s\t%s\n", next_id, side, buf_kind, buf_start, buf_end, cur_heading_id, b64)
      next_id++
      buf = ""; buf_kind = ""; buf_start = 0; buf_end = 0
    }
    /^```/ {
      flush()
      in_code = 1 - in_code
      buf_kind = "code"; buf_start = NR; buf_end = NR; buf = $0
      flush()
      next
    }
    in_code == 1 { next }
    /^[[:space:]]*$/ { flush(); next }
    /^#+ / {
      flush()
      buf_kind = "heading"; buf_start = NR; buf_end = NR; buf = $0
      cur_heading_id = next_id
      flush()
      next
    }
    /^[[:space:]]*([-*+]|[0-9]+\.) / {
      flush()
      buf_kind = "bullet"; buf_start = NR; buf_end = NR; buf = $0
      next
    }
    /^[|]/ {
      if (buf_kind != "table") { flush(); buf_kind = "table"; buf_start = NR; buf = $0 } else { buf = buf "\n" $0 }
      buf_end = NR
      next
    }
    {
      if (buf_kind == "" || buf_kind == "paragraph") {
        if (buf_kind == "") { buf_kind = "paragraph"; buf_start = NR; buf = $0 }
        else { buf = buf "\n" $0 }
        buf_end = NR
      } else {
        flush()
        buf_kind = "paragraph"; buf_start = NR; buf_end = NR; buf = $0
      }
    }
    END { flush() }
  '
}

ITEMS_BEFORE_TSV=$(extract_items_tsv "before" "$OUTSIDE_BEFORE")
ITEMS_AFTER_TSV=$(extract_items_tsv "after"  "$OUTSIDE_AFTER")
```

`extract_items_tsv` が返すフィールド（Claude 本人が解釈する内部スキーマ）:

| フィールド | 説明 |
|---|---|
| `id` | item 一意 ID（内部参照のみ。表示には出さない） |
| `side` | `before` または `after`（マーカー前後どちらか。集約はこの side 内でのみ） |
| `kind` | `heading` / `bullet` / `paragraph` / `code` / `table` |
| `start_line`, `end_line` | CLAUDE.md 全体での 1-based 行番号 |
| `parent_heading_id` | 直近上位の heading id（同じ side 内）。なければ `-` |
| `text_b64` | 表示用原文 (base64) |

`section_number` は持たない。表示用には `ref_quote` 冒頭 30 文字 + 該当行番号を出す。

### 正規化関数（bash で実装）

```bash
# 正規化: NFKC、小文字化、URL 除去、Markdown 記号除去、絵文字除去、半角/全角コロン統一、空白圧縮
normalize_md() {
  python3 -c '
import sys, unicodedata, re
s = sys.stdin.read()
s = unicodedata.normalize("NFKC", s)
s = s.lower()
s = re.sub(r"https?://\S+", " ", s)            # URL 除去
s = re.sub(r"[`*_~\[\]\(\)<>!#]+", " ", s)     # Markdown 記号除去
s = re.sub(r"[\U0001F300-\U0001FAFF\U00002600-\U000027BF]", " ", s)  # 絵文字粗くカット
s = s.replace("：", ":")                       # 全角コロン → 半角
s = re.sub(r"\s+", " ", s).strip()
sys.stdout.write(s)
'
}
```

`python3` は macOS / Linux 共通に同梱されており、bash 単体での Unicode 処理（NFKC / 絵文字判定）が困難なため経由する。

### Step 5a 続き — HIGH 抽出 + token overlap

```bash
# COMBINED の各「ref item」を正規化して、ハッシュセットを作る
declare -A REF_NORM_SET
declare -a REF_NORM_LIST          # token overlap 計算用に norm 文字列を保存
declare -a REF_LINE_LIST          # 同 ref の行番号（COMBINED 内 1-based）を保存
ref_idx=0
while IFS=$'\t' read -r rid rside rkind rstart rend rparent rb64; do
  [ -z "$rid" ] && continue
  [ "$rkind" = "code" ] && continue
  ref_text=$(printf '%s' "$rb64" | base64 -d)
  norm=$(printf '%s' "$ref_text" | normalize_md)
  [ -z "$norm" ] && continue
  REF_NORM_SET["$norm"]=1
  REF_NORM_LIST[$ref_idx]="$norm"
  REF_LINE_LIST[$ref_idx]="$rstart"
  ref_idx=$((ref_idx + 1))
done < <(extract_items_tsv "ref" "$COMBINED")

# 各 outside item を判定
candidates_high=()
candidates_medium=()
candidates_low=()
pending_llm=()

while IFS=$'\t' read -r oid oside okind ostart oend oparent ob64; do
  [ -z "$oid" ] && continue
  [ "$okind" = "code" ] && continue
  text=$(printf '%s' "$ob64" | base64 -d)
  norm=$(printf '%s' "$text" | normalize_md)
  [ -z "$norm" ] && continue

  # 1. HIGH: 完全一致
  if [ -n "${REF_NORM_SET[$norm]:-}" ]; then
    record_candidate "$oid" "$oside" "HIGH" "正規化完全一致"
    continue
  fi

  # 2. token overlap (Jaccard) を Python で計算。最大値とその ref_line を返す
  read -r overlap ref_line < <(python3 -c '
import sys, json
norm = sys.argv[1]
refs = json.loads(sys.argv[2])
ref_lines = json.loads(sys.argv[3])
target = set(norm.split())
best = 0.0
best_line = 0
for i, r in enumerate(refs):
    rs = set(r.split())
    if not rs or not target: continue
    j = len(target & rs) / len(target | rs)
    if j > best:
        best = j; best_line = ref_lines[i]
print(best, best_line)
' "$norm" "$(printf '%s\n' "${REF_NORM_LIST[@]:-}" | python3 -c 'import sys, json; print(json.dumps([l.rstrip() for l in sys.stdin]))')" "$(printf '%s\n' "${REF_LINE_LIST[@]:-0}" | python3 -c 'import sys, json; print(json.dumps([int(l.strip()) for l in sys.stdin if l.strip()]))')")

  if awk -v o="$overlap" -v t="$MIGRATE_OVERLAP_MEDIUM" 'BEGIN{exit !(o>=t)}'; then
    record_candidate "$oid" "$oside" "MEDIUM" "正規化部分一致 overlap=$overlap" "$ref_line"
  elif awk -v o="$overlap" -v t="$MIGRATE_OVERLAP_LLM" 'BEGIN{exit !(o>=t)}'; then
    pending_llm+=("$oid|$oside|$ostart|$ob64|$overlap|$ref_line")
  fi
done < <(printf '%s\n%s\n' "$ITEMS_BEFORE_TSV" "$ITEMS_AFTER_TSV")
```

しきい値 `MIGRATE_OVERLAP_MEDIUM=0.85` / `MIGRATE_OVERLAP_LLM=0.50` は `[要調整]`。fixture で実測してから本値を確定する。環境変数で上書き可能。

### Step 5b. LLM 判断（Claude 本人が内部評価）

bash で大半の候補は Step 5a で決まるが、`overlap ∈ [MIGRATE_OVERLAP_LLM, MIGRATE_OVERLAP_MEDIUM)` のグレーゾーンは **Claude 本人** が以下のプロンプト雛形に沿って各 item を判定する。

> あなた（このコマンド本文を実行している Claude）は、`pending_llm` 中の各 item について、`COMBINED`（マーカー内に書かれる予定の参照テキスト）と読み比べ、下記の判定プロンプトに従って `{"is_duplicate": ..., "confidence": ..., "ref_quote": ..., "ref_line": ..., "reason": ...}` を **頭の中で評価し、結果を内部の候補配列に追加する**。**外部プロセス（`claude` CLI 等）を Bash 経由で呼び出してはいけない**。

#### 判定プロンプト雛形

```
あなたは carta 移行アシスタントです。
下記の「参照テキスト」（憲法本体 + 適用済みプロファイル）と、
「対象 item」（CLAUDE.md マーカー外から抽出された 1 件）を比較し、
対象 item が参照テキスト中のいずれかの記述と意味的に重複しているか
判定してください。

判定基準:
- 同じ指示・原則・運用ルールを述べているなら「重複」
- 参照側はより一般、対象 item はより具体（プロジェクト固有の条件付き）なら「非重複」
- 表現は違っても主旨が同じなら「重複」
- 関連はあるが補完情報を含むなら「非重複」（保留）

出力フォーマット (JSON):
{
  "is_duplicate": true|false,
  "confidence": "HIGH"|"MEDIUM"|"LOW",
  "ref_quote": "<対応する参照側の連続 1〜2 行をそのまま引用 (改行を含めない)>",
  "ref_line": <参照テキスト COMBINED 内での該当開始行番号>,
  "reason": "<判定根拠を 1 文>"
}
```

判定結果の取り扱い:

| `is_duplicate` | `confidence` | デフォルト動作 | `--include-medium` 指定時 |
|---|---|---|---|
| true | HIGH | 削除候補 | 削除候補 |
| true | MEDIUM | 参考表示（削除しない） | 削除候補 |
| true | LOW | 参考表示（削除しない） | 参考表示（削除しない） |
| false | * | **何もしない**（候補配列にも参考枠にも入れない） | 同左 |

> N2 への対応: `is_duplicate=false` の item は候補・参考のどちらにも追加しない。重複ではないと判定された記述は触らないのが安全。

### Step 5c. 見出し集約（保守的、side 独立）

```bash
# outside_before / outside_after を独立に処理する。両者を跨ぐ集約はしない。
removable_confidences="HIGH"
[ "$INCLUDE_MEDIUM" = "true" ] && removable_confidences="HIGH MEDIUM"

for side in before after; do
  for heading_id in $(items_in_side "$side" | filter_kind heading | cut -f1); do
    children=$(items_in_side "$side" | filter_parent "$heading_id")
    [ -z "$children" ] && continue

    # bullet 以外（paragraph / table / code）が 1 件でも含まれるなら見出し集約は発火しない
    has_non_bullet=false
    while IFS=$'\t' read -r cid cside ckind cstart cend cparent cb64; do
      if [ "$ckind" != "bullet" ]; then
        has_non_bullet=true
        break
      fi
    done <<< "$children"
    [ "$has_non_bullet" = "true" ] && continue

    # 配下の全 bullet が削除候補（HIGH / --include-medium 時は HIGH+MEDIUM）か?
    all_removable=true
    while IFS=$'\t' read -r cid cside ckind cstart cend cparent cb64; do
      if ! is_candidate_with_confidence "$cid" "$removable_confidences"; then
        all_removable=false
        break
      fi
    done <<< "$children"

    if [ "$all_removable" = "true" ]; then
      record_candidate "$heading_id" "$side" "HIGH" "配下の全 bullet が削除候補（side=$side）"
    fi
  done
done
```

配下に paragraph / code / table が 1 つでも残るなら見出し集約は発火しない。これは「段落 1 行で重要な但し書きが残っているのに見出しを誤削除する」事態を防ぐ保守的な選択。

---

## 出力フォーマット

### 候補テーブル（人間可読サマリ）

```
[carta:migrate] /Users/foo/myrepo/CLAUDE.md
  version: v0.3.0 sha=abc1234 profiles=typescript (from marker)
  mode:    dry-run            include-medium=false
  status:  2 削除候補（HIGH） / 1 件の参考 (MEDIUM) / 1 件の参考 (LOW)

  # 削除候補（HIGH） — マーカー外 → マーカー内 重複
  -----------------------------------------------------------------
  no. line side    confidence  text                          ref_quote (先頭 30 文字)        ref_line
  -----------------------------------------------------------------
   1   42  after   HIGH        - ドキュメントは日本語で書く  ドキュメント・コメント・ユー...   13
   2   43  after   HIGH        - 識別子は英語                コード・変数名・関数名・コマ...   14
  -----------------------------------------------------------------

  # 参考: MEDIUM（--include-medium で削除候補に昇格）
   - L58 after  ## テスト方針 (見出し + 配下 4 件)
     → §「2. テストと完了基準」と関連。配下の 1 件「## 命名規約」が独立内容のため見出し集約は発火していない

  # 参考: LOW（関連はあるが削除非推奨）
   - L72 after  「PR 作成時に必ず gh pr review を回す」
     → §4 と関連するが運用ツール固有のため保留
```

ヘッダ `profiles=typescript (from marker)` は **profile の出所** を区別表示する。`--profile` 引数で明示した場合は `(from --profile)` を出す。

### unified diff（書き込み前提のもの）

```
--- a/CLAUDE.md (現在)
+++ b/CLAUDE.md (migrate 後)
@@ -40,3 +40,0 @@
-## 言語ルール
-- ドキュメントは日本語で書く
-- 識別子は英語
```

削除のみのケースでは hunk header の変更後ブロック行数は 0 になる（`-N,M +K,0`）。差分は `diff -u` 互換形式。マーカー領域は変更がないため diff に出ない（出たら assertion で落とす）。

### 最終承認プロンプト

```
削除候補 2 件（HIGH のみ）を CLAUDE.md に適用しますか?
※ MEDIUM / LOW は今回の削除対象外です。MEDIUM も含めたい場合は --include-medium を付けて再実行してください。
[y]es / [n]o:
```

`[s]elect`（個別選択）は本リリースで実装しない。必要であれば `--apply` 無しで diff を出力し、ユーザーが手で `CLAUDE.md` を編集する運用とする。

### 候補 0 件のとき（no-op）

```
[carta:migrate] /Users/foo/myrepo/CLAUDE.md
  version: v0.3.0 sha=abc1234 profiles=typescript (from marker)
  status:  no duplicates — マーカー外に重複は検出されませんでした
```

書き込みも diff 出力も行わず exit 0 で終了する。

---

## 安全要件

| # | 要件 | 実装手段 |
|---|---|---|
| 1 | 一発適用しない | デフォルト dry-run。`--apply` でも最終承認プロンプト |
| 2 | マーカー内不変 | 書き込み後 assertion で `inside_after == inside_before` をバイト比較（Step 8） |
| 3 | マーカー位置不変 | 書き込み後に同じ regex でマーカーを再検出し、begin/end のテキスト一致を保つ |
| 4 | マーカー破損時の停止 | apply.md Step 3 と同じ判定。自動修復しない |
| 5 | マーカー内 sha 不一致時の停止 | 「先に `/carta:apply --force` で同期」と誘導。migrate 自身は `--force` を取らない |
| 6 | マーカー無しでの実行禁止 | 「先に `/carta:apply` を実行」と誘導して停止 |
| 7 | コードブロック内を触らない | item 抽出時にコードブロックを skip |
| 8 | バックアップ | 書き込み直前に `BEFORE_FULL` 変数に全体を保持し、assertion 失敗時は変数から書き戻す。`git ls-files --error-unmatch "$TARGET"` で **tracked / untracked / ignored を区別** し、untracked または ignored なら警告を出して `--apply` 続行可否を確認する |
| 9 | 候補 0 件時の no-op | 何も書き込まずに「no duplicates」と報告して exit 0 終了 |
| 10 | 部分採用 | 本リリースは all-or-nothing（y/n）。`--include-medium` で MEDIUM の取り込みは制御可能 |

`git ls-files --error-unmatch` 判定例:

```bash
if ! git ls-files --error-unmatch "$TARGET" >/dev/null 2>&1; then
  echo "WARNING: $TARGET は git の管理下にありません（untracked または ignored）。" >&2
  echo "         '--apply' すると 'git restore' で戻せない可能性があります。" >&2
  if [ "$APPLY" = "true" ]; then
    printf "続行しますか? [y]es / [n]o: "
    read -r ans
    case "$ans" in y|Y|yes|Yes) ;; *) exit 1 ;; esac
  fi
fi
```

---

## 実装メモ

- shell + Read / Edit / Write の組み合わせで実現。複雑なテンプレートエンジンは使わない
- マーカー判定の正規表現は apply.md と完全に同じ: `^<!-- carta:begin v[^ ]+ sha=[0-9a-f]+( profiles=[A-Za-z0-9_,-]+)? -->$`
- `commands/*.md` 間で source 共有しない既存方針のため、apply.md Step 1 / Step 3 の bash ロジックは **コピーで持つ**
- `python3` は macOS / Linux 共通に同梱されており、Unicode 正規化と token overlap 計算に使う
- ロールバック手段は **書き込み前 content の変数保持** に統一する（git tracked / untracked いずれでも復元可能）
- 大文字小文字は厳密に一致させる（`Carta:Begin` 等は別物として扱う）
- マーカー前後を跨ぐ集約は **絶対にしない**（`outside_before` と `outside_after` は完全独立）

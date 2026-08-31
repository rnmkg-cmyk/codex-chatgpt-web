# codex-chatgpt-web Full Harness Portable Runbook

対象: `miuuyy/codex-chatgpt-web` を別の環境へ導入する AI / Codex エージェント  
目的: **このファイルだけを渡せば、環境差分を自力で検出し、Full harness の実動作確認まで止まらず進める**  
基準日: 2026-08-31  
基準 Release: v4.0.6  

---

## 0. このファイルを受け取った AI への実行指示

あなたはこの手順を「固定値をそのまま実行する手順書」として扱ってはいけない。

最初に対象環境と upstream の最新版を調べ、**現在の実環境に合わせて値・パス・UI・Release を更新してから**作業すること。

途中確認は原則不要。安全に自力解決できる問題は、README、docs、Release、Issues、Launcher Activity、doctor、ローカルログを確認して解決する。

ユーザーへ操作を依頼してよいのは、原則として以下のみ。

- ChatGPT / OpenAI へのログイン
- CAPTCHA
- 2FA
- OpenAI Platform 上での API key 新規作成
- OpenAI Platform 上での Secure MCP Tunnel 新規作成
- ChatGPT UI 上での Developer Mode / Connector 作成・権限設定
- macOS が明示的に要求するセキュリティ許可

それ以外で「どこですか」「どれですか」とユーザーへ返さない。まず自分で調べる。

### 絶対禁止

- 既存 Git リポジトリや本番プロジェクトで動作テストしない
- `curl ... | sh` を内容未確認のまま実行しない
- API key / Tunnel runtime key / Cookie / bearer token をチャットやログへ表示しない
- Keychain や既存秘密ファイルを勝手に探索しない
- 古い Connector を勝手に rename / refresh / delete しない
- OpenAI のアカウント制限を迂回しない
- sandbox / approval を無効化して成功扱いしない
- README と実環境が違うのに、古い記憶で押し切らない

---

# 1. 完了条件

以下がすべて確認できるまで「完了」と言わない。

1. Launcher が正常起動
2. embedded browser で ChatGPT ログイン済み
3. Browser smoke test 成功
4. ChatGPT Web モデルが Codex model picker に表示
5. Browser-only の応答成功
6. Full harness tunnel が接続済み
7. 現行 README が要求する Connector が作成済み
8. 現行 README が要求する Authentication / Permissions が設定済み
9. tool scan 成功
10. ChatGPT Web → `pwd` 成功
11. ChatGPT Web → `ls` 成功
12. ChatGPT Web → file read 成功
13. ChatGPT Web → file create 成功
14. ChatGPT Web → file modify 成功
15. ChatGPT Web → patch 成功
16. ChatGPT Web → test file delete 成功
17. Codex sandbox / approval が外側の harness で有効
18. 通常 Codex モデルも利用可能
19. doctor で重大 error 0
20. 残った warning がある場合、機能影響の有無を実動作で説明できる
21. バックアップ場所と rollback 方法を記録
22. ChatGPT Web → native Codex model 切替で `ocx1` / `invalid_encrypted_content` が再発しない
23. 作業時点 upstream に `ocx1` 互換修正が未収録なら、8.3 の互換パッチを適用・テスト済み
24. このMacと同じ構成を再現する場合、Bigger Context と subagent protocol の状態を確認

---

# 2. 最初に環境を検出する

## 2.1 OS / CPU / shell / Home

macOS では以下を実行する。

```bash
uname -a
uname -m
sw_vers
printf 'shell=%s\n' "$SHELL"
printf 'home=%s\n' "$HOME"
```

期待する主な CPU:

- Apple Silicon: `arm64`
- Intel Mac: `x86_64`

macOS 以外だった場合、このファイルの macOS 固有コマンドをそのまま使わない。
upstream README の Windows / Linux installer を使う。

## 2.2 Codex Desktop の存在確認

macOS:

```bash
test -d "/Applications/Codex.app" && echo "Codex.app found" || true
mdfind 'kMDItemCFBundleIdentifier == "com.openai.codex"' 2>/dev/null || true
```

アプリ名や bundle id が変わっている可能性があるため、見つからない場合は `/Applications` を確認する。

## 2.3 既存 codex-chatgpt-web の存在確認

```bash
test -d "$HOME/.codex-chatgpt-web" && echo "existing install found" || echo "fresh install"
test -f "$HOME/.codex-chatgpt-web/config.json" && echo "config exists" || true
```

CLI が PATH にある場合:

```bash
command -v codex-chatgpt-web || true
codex-chatgpt-web --version 2>/dev/null || true
```

PATH にない場合でも「未インストール」と断定しない。Launcher 内 runtime の可能性がある。

---

# 3. upstream を必ず最新確認する

作業開始時に以下を確認する。

- `https://github.com/miuuyy/codex-chatgpt-web`
- `https://github.com/miuuyy/codex-chatgpt-web/releases`
- `README.md`
- `docs/security-model.md`
- `docs/architecture.md`
- `docs/release-validation.md`
- 最新 Release notes
- open / recently closed Issues の重大障害

Web が使える場合は GitHub を直接確認する。

CLI しか使えない場合は一時ディレクトリへ取得する。

```bash
AUDIT_DIR="$(mktemp -d /tmp/codex-chatgpt-web-audit.XXXXXX)"
git clone --depth 1 https://github.com/miuuyy/codex-chatgpt-web.git "$AUDIT_DIR/repo"
```

Git clone が使えない場合は GitHub raw / release assets を取得する。

### ここで抽出して固定する値

次の値を README から毎回抽出する。

```text
LATEST_RELEASE=
CONNECTOR_NAME=
CONNECTIVITY=
AUTHENTICATION=
PERMISSIONS=
INSTALLER_ASSET=
SUPPORTED_MACOS=
SUPPORTED_ARCH=
```

2026-08-31 / v4.0.6 では参考値として:

```text
CONNECTOR_NAME=Codex Native2
CONNECTIVITY=Tunnel
AUTHENTICATION=None
PERMISSIONS=Allow all actions
INSTALLER_ASSET=install-launcher.sh
```

ただし、**作業時点の README が違えば必ず作業時点の値を使う。**

---

# 4. OpenAI 公式仕様も最新確認する

最低限:

- Developer mode / MCP apps
- Secure MCP tunnels
- 現在の plan / workspace の write / modify 制限

確認先:

- `https://help.openai.com/en/articles/12584461-developer-mode-and-mcp-apps-in-chatgpt`
- `https://developers.openai.com/api/docs/guides/secure-mcp-tunnels`

GitHub README と OpenAI 公式説明が食い違う場合:

1. どちらかを勝手に無視しない
2. 実アカウント UI に Developer Mode / Tunnel / Connector / permission が存在するか確認
3. 実 tool call が許可される範囲だけ使う
4. アカウント側が拒否した機能は迂回しない

---

# 5. 既存環境を壊さないための preflight

## 5.1 作業場所

既存 Git repo の外に作る。

```bash
RUN_ROOT="$HOME/Documents/Codex/codex-chatgpt-web-setup-$(date +%Y%m%d-%H%M%S)"
mkdir -p "$RUN_ROOT"/{downloads,backups,logs,notes}
printf '%s\n' "$RUN_ROOT"
```

`$HOME/Documents/Codex` が存在しない・利用したくない環境では:

```bash
RUN_ROOT="$(mktemp -d "$HOME/codex-chatgpt-web-setup.XXXXXX")"
mkdir -p "$RUN_ROOT"/{downloads,backups,logs,notes}
```

## 5.2 Git repo 外であることを確認

```bash
cd "$RUN_ROOT"
git rev-parse --is-inside-work-tree 2>/dev/null || echo "not a git worktree"
```

`true` が返った場合は別の場所へ移動する。

---

# 6. 既存 Codex 設定を確認・バックアップ

## 6.1 存在確認

```bash
ls -ld "$HOME/.codex" 2>/dev/null || true
ls -ld "$HOME/.codex-chatgpt-web" 2>/dev/null || true
test -f "$HOME/.codex/config.toml" && echo "Codex config exists" || true
```

秘密情報を `cat` しない。

## 6.2 変更され得る項目を把握

`~/.codex/config.toml` では少なくとも次を確認する。

- `model`
- `openai_base_url`
- `[mcp_servers.*]`
- codex-chatgpt-web 管理コメント
- subagent / multi-agent 関連 managed values

秘密値を含まない範囲で:

```bash
rg -n 'model|openai_base_url|mcp_servers|Managed by codex-chatgpt-web|multi_agent|max_depth' \
  "$HOME/.codex/config.toml" 2>/dev/null || true
```

## 6.3 バックアップ

```bash
STAMP="$(date +%Y%m%d-%H%M%S)"
BACKUP_DIR="$RUN_ROOT/backups/codex-$STAMP"
mkdir -p "$BACKUP_DIR"

cp -p "$HOME/.codex/config.toml" "$BACKUP_DIR/config.toml" 2>/dev/null || true
cp -p "$HOME/.codex/models_cache.json" "$BACKUP_DIR/models_cache.json" 2>/dev/null || true
```

既存 `~/.codex-chatgpt-web` がある場合、設定・状態だけ保全する。

秘密情報を含む可能性があるので、バックアップはユーザーの Mac 内だけに置く。

### バックアップ禁止事項

- Git commit しない
- Cloud sync / issue / chat に貼らない
- browser profile / cookies を外部共有しない
- key 本文をログへ転記しない

---

# 7. fresh install / upgrade を判定

以下のどちらかに分類する。

## A. Fresh install

`~/.codex-chatgpt-web` がない、または正式インストールが確認できない。

→ Release installer から導入。

## B. Upgrade / existing install

既存設定または Launcher がある。

→ まず現在バージョンを確認し、Release notes の migration を読む。

特に Connector identity の migration がある Release では、**旧 Connector を rename / refresh しない。**

### 7.1 Upgrade では「設定上のバージョン」と「実稼働 Runtime」を分けて確認する

v4.0.5 → v4.0.6 の実機更新で、次の不整合が実際に発生した。

```text
Launcher app: 4.0.6
config.releaseVersion: 4.0.6
config.runtimeCommand: 4.0.6
実際に 127.0.0.1:17841 で稼働中の Runtime: 4.0.5
```

この状態では「更新済み」に見えても、旧 Runtime が turn を処理するため hotfix が効かない。

Upgrade 後は必ず次の3点を照合する。

```bash
printf 'health='; curl -fsS http://127.0.0.1:17841/healthz 2>/dev/null || true
jq '{releaseVersion,runtimeCommand}' "$HOME/.codex-chatgpt-web/config.json" 2>/dev/null || true
defaults read '/Applications/Codex Web GPT.app/Contents/Info' CFBundleShortVersionString 2>/dev/null || true
ps ax -o pid=,etime=,command= | rg 'codex-chatgpt-web/versions/.*/app/cli.js serve|Codex Web GPT.app/Contents/MacOS' | rg -v 'rg ' || true
```

合格条件:

- Launcher app version = latest Release
- `config.releaseVersion` = latest Release
- `runtimeCommand` の version directory = latest Release
- `/healthz` の `version` = latest Release
- 稼働中 `serve` process の path = latest Release

1つでも古い場合、Runtime の完全再生成・Launcher 再起動を完了するまで次工程へ進まない。

---

# 8. Release installer を安全に取得

README の正式 installer asset 名を確認してから実行する。

v4.0.6 の例:

```bash
cd "$RUN_ROOT/downloads"

curl -fL \
  -o install-launcher.sh \
  "https://github.com/miuuyy/codex-chatgpt-web/releases/latest/download/install-launcher.sh"

curl -fL \
  -o checksums.txt \
  "https://github.com/miuuyy/codex-chatgpt-web/releases/latest/download/checksums.txt"
```

### 8.1 ダウンロード元確認

```bash
ls -l install-launcher.sh checksums.txt
head -40 install-launcher.sh
```

確認:

- GitHub `miuuyy/codex-chatgpt-web` Release 由来
- 不審な third-party download がない
- 不要な `sudo` がない
- shell profile への予想外の永続変更がない
- checksum 検証ロジックが upstream 説明と一致

### 8.2 SHA-256

```bash
shasum -a 256 install-launcher.sh
rg -n 'install-launcher\.sh' checksums.txt || true
```

manifest に installer 自体が含まれない Release の場合、installer が取得する app package の checksum 検証方法を script と Release notes で確認する。

**checksum 不一致なら実行禁止。**

### 8.3 このMacと同じ挙動にするための `ocx1` 互換パッチ確認

重要: 2026-08-31 時点の upstream v4.0.6 / main には、今回このMacで追加した
「ChatGPT Web 独自の `ocx1:` compaction envelope を native Codex へ送る前に
プレーンテキストへ正規化する処理」は入っていない。

したがって、**公式 Release を入れただけでは、このMacと完全に同じ状態にはならない。**

まず作業時点の upstream に同等修正が既に入ったかを確認する。

```bash
rg -n 'isBridgeCompactionItem|compactionItemToText|BRIDGE_COMPACTION_PREFIX' \
  src/native-passthrough.ts tests/native-passthrough.test.ts
```

次の両方が upstream に存在し、テストも通るなら、この章のローカル互換パッチは不要。

- `type: "compaction"` かつ `encrypted_content` が `ocx1:` で始まる item を検出
- native Codex passthrough 前にその item を通常の plaintext `message` へ変換

存在しない場合は、**最新版へ追従したうえで同等修正を再適用する。**
バージョン番号だけを見て「v4.0.6だから修正済み」と判断しない。

#### このMacで追加した最小修正の意味

`src/native-passthrough.ts` の native 送信直前で:

```ts
import { BRIDGE_COMPACTION_PREFIX, compactionItemToText } from "./responses/compaction";

type BridgeCompactionItem = JsonObject & { encrypted_content: string };

function isBridgeCompactionItem(value: unknown): value is BridgeCompactionItem {
  return isObject(value)
    && value.type === "compaction"
    && typeof value.encrypted_content === "string"
    && value.encrypted_content.startsWith(BRIDGE_COMPACTION_PREFIX);
}

function isBridgeArtifactItem(value: unknown): boolean {
  return isBridgeReasoningItem(value) || isBridgeCompactionItem(value);
}
```

既存の `scrubBridgeArtifactsForNative()` は、bridge reasoning だけでなく bridge compaction も
変換対象にする。

```ts
if (!isObject(value) || !Array.isArray(value.input) || !value.input.some(isBridgeArtifactItem)) {
  return { value, changed: false };
}

const input = value.input.flatMap(item => {
  if (!isObject(item)) return [item];

  if (isBridgeCompactionItem(item)) {
    return [{
      type: "message",
      role: "user",
      content: [{ type: "input_text", text: compactionItemToText(item.encrypted_content) }],
    }];
  }

  // 以降は upstream の既存 reasoning scrub 処理を維持する
});
```

同時に `tests/native-passthrough.test.ts` へ、少なくとも次を保証する回帰テストを追加する。

```text
入力:
  type=compaction
  encrypted_content=ocx1:<base64 summary>
  previous_response_id=bridge local id

期待:
  previous_response_id が native request から消える
  ocx1: が native request に残らない
  summary が plaintext user message として残る
  通常の native encrypted reasoning は改変しない
```

#### パッチを適用するソースは「作業時点の最新 Release tag」

既存プロジェクトではなく専用一時ディレクトリへ取得する。

```bash
PATCH_ROOT="$(mktemp -d /tmp/codex-chatgpt-web-patched.XXXXXX)"
LATEST_TAG="$(curl -fsSL https://api.github.com/repos/miuuyy/codex-chatgpt-web/releases/latest | jq -r '.tag_name')"
git clone --depth 1 --branch "$LATEST_TAG" \
  https://github.com/miuuyy/codex-chatgpt-web.git "$PATCH_ROOT/repo"
cd "$PATCH_ROOT/repo"
```

最新版でコード構造が変わっている場合、上の断片を機械的に貼らない。
`responses/compaction.ts` と native passthrough の現行実装を読み、同じ安全条件を満たす最小差分へ移植する。

#### 必須テスト

公式 Launcher を先に導入済みなら、config の runtime command から同梱 Bun を取得できる。

```bash
BUN_BIN="$(jq -r '.runtimeCommand[0]' "$HOME/.codex-chatgpt-web/config.json")"
"$BUN_BIN" --version
"$BUN_BIN" install --frozen-lockfile
"$BUN_BIN" test tests/native-passthrough.test.ts
"$BUN_BIN" run typecheck
"$BUN_BIN" test tests/*.test.ts
```

対象テスト → typecheck → 全テストの順に通す。

#### macOS で patched Launcher を作る

```bash
"$BUN_BIN" run app:package
find launcher/artifacts -maxdepth 1 -type f \
  \( -name '*.dmg' -o -name '*.zip' \) -print
```

生成物を確認してから既存 Launcher を置き換える。置き換え前に `/Applications/Codex Web GPT.app`
の存在と現在バージョンを記録し、実行中 turn が 0 であることを確認する。

patched app の起動後は、必ず 7.1 の version coherence 確認を行う。

今後 upstream に同等修正が入った Release が出たら、ローカルパッチを重ねず、公式実装へ戻す。

---

# 9. Launcher インストール

安全確認済み installer を実行する。

```bash
sh "$RUN_ROOT/downloads/install-launcher.sh"
```

macOS Gatekeeper が表示された場合:

1. Release asset / checksum の確認済みであることを再確認
2. upstream が未署名 build を明示しているか確認
3. macOS の明示許可が必要なら、その UI 操作だけユーザーへ依頼

許可を回避する非公式 command は使わない。

---

# 10. Launcher の場所と runtime を自力で探す

アプリ名や PATH を固定しない。

macOS:

```bash
find /Applications "$HOME/Applications" -maxdepth 2 \
  -name '*Codex*Web*GPT*.app' -print 2>/dev/null || true
```

設定:

```bash
test -f "$HOME/.codex-chatgpt-web/config.json" && echo "launcher config found"
```

CLI の候補:

```bash
command -v codex-chatgpt-web || true
find "$HOME/.codex-chatgpt-web" -maxdepth 4 -type f \
  \( -name 'cli.js' -o -name 'codex-chatgpt-web' \) -print 2>/dev/null | head -20
```

PATH にないことを故障と誤認しない。

---

# 11. Launcher 基本セットアップ

Launcher を起動。

順番:

1. embedded browser 起動確認
2. ChatGPT login
3. Browser smoke test
4. account capabilities 検出
5. Install models
6. Codex Desktop 完全再起動
7. Upgrade の場合は `/healthz` と実 process path が最新版へ切り替わったことを確認

## 11.1 ChatGPT login

ログイン / 2FA / CAPTCHA はユーザーへ依頼してよい。

「ログインしてください」だけでなく、Launcher の embedded browser 内で行うことを明示する。

外部 Chrome のログイン済み Cookie が自動流用される前提にしない。

## 11.2 Browser smoke test

成功条件:

- authenticated session が取れる
- Temporary Chat を作れる
- composer を検出できる
- Launcher が Passed / Ready 相当を示す

失敗時は Activity を読む。

## 11.3 Install models

Install models 後、**Codex Desktop を完全終了して再起動する。**

macOS ではウィンドウを閉じるだけでなく Cmd+Q / Quit を使う。

再起動要求が Launcher state に残る場合、Codex 再起動前後で state を比較する。

---

# 12. model picker 検証

Codex model picker で:

- `ChatGPT Web — ...` 系が表示される
- 通常 Codex モデルも残る

ことを確認。

表示される Web model は account capabilities に依存する。

モデル名を固定して「Pro がないから失敗」と判断しない。

例:

- Free / Go: Luna のみの場合がある
- eligible account: Instant / Medium / High / Extra High / Pro 等

---

# 13. Browser-only smoke test

Full harness の前に、ChatGPT Web モデル自体が動くことを確認。

テスト prompt:

```text
BROWSER_ONLY_OK とだけ返してください。
```

成功条件:

- Codex task UI 内で Web backend の回答が返る
- tool 不要の単純 turn が完走する

ここが失敗している状態で MCP に進まない。

## 13.1 このMacと同じ追加設定を再現する場合

2026-08-31 の実機では:

```text
experimentalBiggerContext=true
subagents protocol=compatibility-v1
autoApproveToolCalls=false
```

を確認済み。

`Bigger Context` は v4.0.6 Release で experimental とされ、最大で通常より大きい context / compaction
limit を使える一方、追加リクエストにより rate limit / cooldown の可能性が上がる。

このMacと同じ環境を再現する目的なら Launcher の現行 UI で Bigger Context を有効化し、設定反映を
秘密値を出さず確認する。

```bash
jq '{experimentalBiggerContext,autoApproveToolCalls}' \
  "$HOME/.codex-chatgpt-web/config.json"
```

subagent protocol は v4.0.6 の新規 install では README 上 `Compatibility V1` が既定。

```bash
codex-chatgpt-web subagents status 2>/dev/null || true
```

CLI が PATH にない場合は config の `runtimeCommand` から同梱 CLI を使う。

Bigger Context を有効化した直後に compaction / rate-limit 問題が出た場合、原因を切り分けるため
一時的に無効化して比較してよい。ただし、それで `ocx1` 問題の根本修正済みとは扱わない。

---

# 14. Full harness 用 MCP セットアップ

Launcher の **MCP** ページを開く。

v4.0.6 の画面構成:

1. Create a tunnel and API key
2. Connect the local harness
3. Attach the ChatGPT connector

UI 文言が変わっていたら、最新版 README の Full harness 節と照合する。

---

# 15. Tunnel ID と API key を混同しない

## Tunnel ID

OpenAI Secure MCP Tunnel の ID。

通常:

```text
tunnel_...
```

ChatGPT の Channel ID、Connector ID、Plugin ID ではない。

## API key

OpenAI Platform の通常 API key。

ChatGPT Settings ではなく OpenAI Platform 側で作成する。

README が要求する scope / runtime key 形式が将来変わった場合は最新版に従う。

### 秘密情報ルール

- key 本文をチャットへ貼らない
- shell command argument へ直書きしない
- `ps` に見える形で起動しない
- Launcher の secret input / hidden prompt / runtime key file を使う
- 既存 key を Keychain から勝手に探索しない

---

# 16. Secure MCP Tunnel 作成

OpenAI Platform の Secure MCP Tunnel UI を開く。

ユーザーが作成する。

AI は以下だけを案内する。

1. Tunnel を新規作成
2. Tunnel ID を確認
3. 同じ OpenAI account で API key を新規作成
4. 値は chat に送らず Launcher へ直接入力

Tunnel と Connector に別アカウントを使わない。

---

# 17. Connect harness

Launcher MCP 画面へ:

- Tunnel ID
- API key / runtime key

を入力し、**Connect harness**。

成功条件:

- tunnel client running
- harness connected / ready
- broker / MCP process running

CLI が利用可能なら status も確認する。

```bash
codex-chatgpt-web tunnel status 2>/dev/null || true
codex-chatgpt-web service status 2>/dev/null || true
codex-chatgpt-web route status 2>/dev/null || true
```

PATH に CLI がない場合は Launcher Activity / doctor を使う。

---

# 18. Tunnel 接続トラブルの標準復旧

同じボタンを連打しない。

## stale retained turn / 502 / false green が疑われる場合

順番を守る。

1. 実行中 Web turn があれば終了を待つ
2. Launcher Activity を確認
3. retained turn が残っていれば Launcher Settings から Cancel
4. harness Disconnect
5. tunnel / MCP が stopped になったことを確認
6. Connect harness
7. doctor / health check
8. 新規 Codex task でテスト

同じ失敗を observable state が変わらないまま繰り返さない。

---

# 19. ChatGPT Developer Mode

ChatGPT Settings 内で Developer Mode を有効化する。

UI 名は rollout により `Apps` / `Plugins` / `Connectors` 等に変わり得る。

固定メニュー名だけを探して「存在しない」と判断しない。

探す概念:

- Settings
- Apps / Plugins
- Advanced / Developer Mode
- Create connector / New plugin / Add MCP

workspace policy により Developer Mode / write actions が見えない場合、迂回しない。

---

# 20. Connector 作成

**必ず作業時点の README から Connector 名を再取得する。**

v4.0.6 では:

```text
Name: Codex Native2
Connectivity: Tunnel
Authentication: None
Permissions: Allow all actions
```

Tunnel は Launcher へ設定したものと同じものを選択。

## 20.1 旧 Connector がある場合

README が v4 系の direct turn-token contract を要求するなら:

- `Codex Native` は残す
- rename しない
- refresh しない
- `Codex Native2` を新規作成

理由: ChatGPT が Connector identity 単位で public MCP contract を cache するため。

---

# 21. Permissions

README が `Allow all actions` を要求している場合はその通りに設定。

`Allow low-risk actions` では command / patch が ChatGPT 側で外側の Codex harness に届く前にブロックされる。

ただし `Allow all actions` は Codex sandbox / approval を消す設定ではない。

外側 Codex の:

- sandbox
- approval
- workspace roots
- tool registry

が最終権限を持つ。

---

# 22. Tool Scan

Connector 作成時に Scan Tools / tool discovery を実行。

成功条件:

- current connector の tools が列挙される
- legacy connector の schema ではない

失敗時:

1. Tunnel connected?
2. correct Tunnel selected?
3. Authentication correct?
4. Connector name exact?
5. current README と一致?
6. old Connector を誤選択していない?
7. Launcher Activity に MCP error はない?

---

# 23. Verify runtime

Launcher MCP → **Verify runtime**。

成功条件:

- 現行 Connector identity を exact match
- tool contract を認識
- runtime verification successful

## legacy-only error

現行 README が `Codex Native2` なのに `Codex Native` しか見つからない場合:

→ 新規 current Connector を作る。legacy の rename で直さない。

## Temporary Chat / structural personalization 系 warning

次の順で切り分ける。

1. Launcher version = latest?
2. Browser smoke passed?
3. current Connector enabled?
4. Connector name exact?
5. correct tunnel?
6. Permissions correct?
7. Codex restarted after Install models?
8. stale retained turn を cancel
9. Launcher restart
10. new Codex task
11. Verify runtime again
12. doctor
13. 実 tool call test

Verify runtime の UI probe だけが warning でも、実際の shell / file / patch が同一 turn で成功するなら、その差を記録して機能判定する。

## 23.1 `ocx1... could not be decrypted or parsed` / model 切替エラー

v4.0.6 で重要な回帰確認対象。

代表症状:

```text
The encrypted content ocx1... could not be verified.
Reason: Encrypted content could not be decrypted or parsed.
```

または:

```text
Error running remote compact task: ... structured context handoff ...
```

これは、長い ChatGPT Web task の compaction / checkpoint 情報を通常 Codex model へ切り替える際の履歴引き継ぎで発生し得る。

2026-08-31 時点の stock upstream v4.0.6 では、bridge 独自 `ocx1:` compaction を native Codex
へ渡す前に plaintext 化する直接修正は入っていない。このMacでは v4.0.6 に 8.3 のローカル
互換パッチを追加している。

したがって、まず「stock か patched か」を確認し、そのうえで実際にどの Runtime が turn を
処理しているかを確認する。

調査順序:

1. `/healthz.version`
2. `config.releaseVersion`
3. `config.runtimeCommand`
4. 実 `serve` process path
5. Launcher app version
6. upstream に `ocx1:` 正規化があるか、または 8.3 の互換パッチ適用済みか
7. すべて一致後に同じ既存 session で再テスト

既存 session 自体が一律で修正対象外という意味ではない。8.3 の変換処理は native 送信直前に
適用されるため、patched Runtime が実際に稼働していれば既存 session でも再テストする価値がある。

それでも既存 session だけ失敗し、新規 session では成功する場合は、既存履歴固有の未対応形式として記録し、新規 task + plaintext handoff を安全な回避策とする。

### 23.2 Runtime 更新後の安全な再起動

Launcher 管理型では、旧 macOS LaunchAgent service が存在しないことが正常な場合がある。

```text
Service is not installed: ~/Library/LaunchAgents/io.github.codex-chatgpt-web.daemon.plist
```

この場合に `codex-chatgpt-web service restart` を何度も繰り返さない。

v4.0.6 architecture では Desktop Launcher が macOS / Windows / Linux の通常 install で process supervisor を担う。

安全な再起動は原則:

1. 実行中 turn がないことを `/healthz` の active counters で確認
2. Codex Desktop を完全終了
3. Codex Web GPT Launcher を完全終了
4. Launcher を起動
5. `/healthz.version` と process path を確認
6. Codex Desktop を起動
7. model picker / Browser smoke / Full harness を再確認

`active_http_turns` / `active_browser_turns` が 0 でない最中に Runtime を落とさない。

---

# 24. Full harness の実動作テスト

本番 repo ではテストしない。

ChatGPT Web model を選択して、以下をそのまま送る。

```text
Full harness の安全な実動作テストを行ってください。

既存プロジェクト、Gitリポジトリ、本番環境には一切変更を加えないでください。
専用の一時ディレクトリを /tmp 配下に mktemp で作成し、その中だけを操作してください。

以下を実際のツールで順に行ってください。

1. pwd
2. ls
3. test.txt を新規作成し `alpha` と書く
4. test.txt を読み込む
5. 通常のファイル変更で `beta` を追加
6. apply_patch で `gamma` を追加
7. test.txt を読み込み `alpha / beta / gamma` を確認
8. ls で test.txt の存在を確認
9. test.txt を削除
10. 一時ディレクトリを削除

文章上の自己申告ではなく、各 tool result と実ファイル状態で成功を検証してください。
filesystem / shell / patch が現在の外側 Codex harness 経由で実行されたことも確認してください。
```

---

# 25. Full harness テストの合格基準

以下を個別に YES / NO で記録する。

| 能力 | 合格条件 |
|---|---|
| shell | `pwd`, `ls` の実 tool result |
| filesystem read | `test.txt` を実際に読めた |
| file create | 実ファイルが作成された |
| file modify | 内容が変わった |
| patch | native patch tool が成功 |
| delete | 作成物を削除できた |
| sandbox | 外側 Codex の sandbox 情報と整合 |
| approval | 外側 Codex の approval policy に従う |
| MCP | ChatGPT Web → current Codex turn に tool call が戻る |

ChatGPT が「成功」と返しただけでは合格にしない。

---

# 26. 通常 Codex の回帰テスト

ChatGPT Web model から通常 Codex model へ切り替える。

v4.0.6 では、新規 task だけでなく、可能なら ChatGPT Web で数ターン進めた既存 task でも切替をテストする。

まず新しい task で read-only の簡単な操作を行う。

```text
通常Codexの回帰確認です。
現在の作業ディレクトリを読み取りだけで確認してください。
ファイル変更はしないでください。
```

成功条件:

- native Codex model が選べる
- bridge 経由でなく通常 provider も使える
- Codex UI / task / tools が正常
- ChatGPT Web → native Codex 切替で `ocx1` / compaction handoff error が出ない

長い task を作る必要はないが、少なくとも Web 側で複数ターン進めた session を1つ切替テストする。

---

# 27. Doctor / Activity / health

Launcher で利用可能なものをすべて実行。

- Browser smoke test
- Activity
- Run doctor
- health check
- Verify runtime

CLI が使える場合:

```bash
codex-chatgpt-web doctor --json
```

### 合格

- fatal / error = 0
- Browser ready
- service ready
- route ready
- tunnel ready
- connector / MCP functional
- Codex integration verified

warning が残った場合:

1. warning の exact text を保存
2. 実機能へ影響するか検証
3. 影響するなら原因解決まで続行
4. 影響しない UI probe warning なら、実動作証拠と共に記録

### 27.1 health の version coherence も Doctor と別に確認

Doctor が正常でも、upgrade 直後は旧 Runtime が残っていないかを明示的に確認する。

```bash
curl -fsS http://127.0.0.1:17841/healthz
```

最低限確認:

- `status=ok`
- `version=latest Release`
- `mode=full`
- `accepting_turns=true`
- 再起動前に安全停止するなら `active_http_turns=0`
- 再起動前に安全停止するなら `active_browser_turns=0`

---

# 28. Launcher state の確認

macOS の典型例:

```bash
STATE="$HOME/Library/Application Support/Codex Web GPT/launcher-state.json"
test -f "$STATE" && sed -n '1,220p' "$STATE"
```

アプリ名が変わっている場合は Application Support 内を探索する。

確認候補:

- `browserSmokePassed`
- `coreSetupComplete`
- `mcpRuntimeInstalled`
- `mcpSetupComplete`
- `codexCatalogVerified`
- `codexRestartRequired`

state field は version により変わり得る。README / source を優先。

`codexRestartRequired=true` が残る場合:

1. Codex Desktop 完全終了
2. Codex 再起動
3. model picker 確認
4. Launcher state 再確認
5. doctor 再実行

---

# 29. プロセス確認

必要な場合のみ read-only で確認。

```bash
ps aux | rg 'Codex Web GPT|codex-chatgpt-web|tunnel-client|turn-broker' | rg -v 'rg '
```

期待:

- Launcher
- service / serve
- tunnel-client
- MCP / broker

多重起動が疑われる場合、PID を見て即 kill しない。
Launcher の正式な Disconnect / restart / cancel を先に使う。

---

# 30. よくある詰まりと自動解決表

| 症状 | 最初に疑う | 自動解決 |
|---|---|---|
| API key の場所不明 | ChatGPT Settings を探している | OpenAI Platform API keys を案内 |
| Tunnel ID 不明 | Channel ID と混同 | Secure MCP Tunnel の `tunnel_...` |
| MCP ページ不明 | 古い記事 | 最新 Launcher の nav + README |
| Web model が出ない | Codex 再起動不足 | Install models → Quit → relaunch |
| Browser smoke NG | login / UI drift | Activity → login state → latest Release |
| Full だけ NG | Tunnel / connector | status → identity → permissions |
| shell NG | permission | README 要求値。v4.0.6 は Allow all actions |
| patch NG | low-risk / stale schema | Allow all + new current Connector |
| legacy connector のみ | migration | legacy 放置 + current connector 新規作成 |
| Verify runtime NG | UI drift / stale turn | cancel → reconnect → new task → doctor |
| 502 | retained turn / tunnel state | stop state 確認後 reconnect |
| 429 | account rate limit | setup 障害と分離。時間を置く |
| Pro がない | account capability | 失敗扱いしない |
| write UI がない | plan/workspace policy | 迂回せず Browser-only/read-only |
| normal Codex NG | route integration | remove integration / rollback を検討 |
| PATH に CLI なし | launcher runtime install | config/runtime path を探す |
| Gatekeeper | unsigned build | Release/checksum 確認後ユーザー許可 |
| app/config は最新版なのに修正が効かない | stale Runtime | `/healthz` と `ps` で実稼働版確認 → Launcher 完全再起動 |
| `ocx1... could not be decrypted or parsed` | cross-backend history / old Runtime | まず実Runtime版を一致 → 既存session再テスト → 新規taskで切り分け |
| `service restart` が LaunchAgent not installed | Launcher 管理型 | 異常扱いしない。Desktop Launcher を supervisor として扱う |
| embedded login が passkey-only で完了しない | upstream open Issue #209 相当 | 現行 Issue / Release を確認。認証迂回はしない |
| Temporary Chat が `Temporarily Limited` | account / ChatGPT temporary chat 制限、Issue #230 相当 | setup故障と分離し、時間を置く・Issue/Release確認 |
| `Reconnecting` のまま | browser/runtime reconnect、Issue #193 相当 | Activity → health → active turn → latest Release/Issue確認 |
| `stream disconnected before completion` | browser turn / rate limit / UI drift、Issue #192 相当 | exact error と Activity を保存し、再送連打せず切り分け |
| `Stuck on thinking` | current browser execution、Issue #234 相当 | active MCP/tool progress と Activity を確認し、最新版Issueへ照合 |

---

# 31. エラー時の調査順序

質問する前にこの順番で調べる。

1. exact error text
2. current Launcher Activity
3. `doctor --json`
4. current README
5. current Release notes
6. security-model
7. architecture
8. relevant source / tests
9. GitHub Issues
10. OpenAI official docs

同一 command の deterministic failure を、入力や state を変えずに繰り返さない。

---

# 32. セキュリティモデルの理解

Full harness では ChatGPT Web が Mac を直接無制限操作するわけではない。

概念上:

```text
Codex task
  ↓
local Responses bridge
  ↓
ChatGPT Temporary Chat
  ↓
OpenAI Secure MCP Tunnel
  ↓
current turn-bound Codex Native connector
  ↓
outer Codex harness
  ↓
sandbox / approval / filesystem / shell / patch / configured tools
```

重要:

- turn-scoped capability
- random turn token
- current outer Codex turn の tool registry のみ
- turn 終了 / abort / expiry で capability 失効
- localhost Responses listener
- Tunnel は outbound
- inbound public port 不要
- automatic permanent approval を与えない

---

# 33. アカウント制限

OpenAI 公式の plan / workspace policy を毎回確認する。

実 UI で:

- Developer Mode
- Secure MCP Tunnel
- Connector creation
- Allow all actions 相当
- write / modify actions

が使えるか確認する。

使えない場合:

```text
Full harness: アカウント側制限により未完了
Browser-only: 利用可能 / 利用不可
read-only MCP: 利用可能 / 利用不可
拒否された UI / action: exact text
```

を報告して止める。

制限回避はしない。

---

# 34. rollback

まず upstream が提供する正式な integration removal を使う。

v4.0.6:

Launcher:

```text
Settings → Remove Codex integration
```

その後 Codex Desktop を完全再起動。

CLI の正式 uninstall が current README で存在する場合のみ使用。

v4.0.6 例:

```bash
codex-chatgpt-web uninstall --yes
```

順番:

1. Remove Codex integration
2. Codex restart
3. normal Codex 動作確認
4. 必要なら Launcher uninstall
5. backup と現在 config を diff
6. upstream の復元が失敗した場合だけ backup を使う

`rm -rf ~/.codex` のような復元は絶対にしない。

---

# 35. 最終報告フォーマット

余計な長文は不要。最後は必ずこの形式。

```text
Install: 成功 / 失敗
Installed version: x.y.z
Runtime implementation: stock upstream / local compatibility patched / upstream fix included
Full harness: 成功 / 失敗
Connector: <actual current connector name>
Authentication: <actual value>
Permissions: <actual value>
Browser smoke: 成功 / 失敗
ChatGPT Web → shell: 成功 / 失敗
ChatGPT Web → file read: 成功 / 失敗
ChatGPT Web → file create/modify: 成功 / 失敗
ChatGPT Web → patch: 成功 / 失敗
ChatGPT Web → delete: 成功 / 失敗
ChatGPT Web → native Codex model switch: 成功 / 失敗
Normal Codex: 成功 / 失敗
Bigger Context: enabled / disabled / not applicable
Subagents protocol: compatibility-v1 / native / not configured
Doctor: ok / errors / warnings
Backup: <path>
Rollback: <current official method>
Remaining issues: なし / exact details
```

秘密値は書かない。

---

# 36. このファイルを AI に渡すときの一行指示

この Markdown ファイルを添付して、AI には次だけ伝えればよい。

```text
添付の portable runbook を実行してください。作業時点の最新版とこの環境を最初に検出し、途中確認なしで Full harness の実動作確認と通常Codexの回帰確認まで完了してください。人間操作が不可避な認証・OpenAI UI・macOS明示許可だけ、その場で具体的に案内してください。
```

---

# 37. 基準情報（2026-08-31 / v4.0.6）

この章は fallback 用。作業時点の upstream が優先。

```text
Release: v4.0.6
Connector: Codex Native2
Connectivity: Tunnel
Authentication: None
Permissions: Allow all actions
Full harness transport: OpenAI Secure MCP Tunnel
Browser host: Launcher embedded browser
Supported release targets: macOS 13+ arm64/x64, Windows x64, Linux x64
```

v4.0.6 README / Release の重要事項:

- Full mode は current Codex task の filesystem / shell / images / approvals / configured tools/apps を MCP 経由で利用
- `Codex Native2` を新規作成
- legacy `Codex Native` を rename / refresh / reuse しない
- `Allow low-risk actions` は commands / patches をブロック
- outer Codex harness が sandbox / approvals を保持
- Tunnel は outbound で public inbound port 不要
- unsigned builds のため macOS Gatekeeper が警告する場合がある
- installer は published SHA-256 manifest を検証
- continuous task session と native compaction を使用
- retained Web agent から checkpoint を取り、context boundary 後は clean chat へ継続
- cross-backend history / compaction 周辺には v4.0.6 の改善があるが、`ocx1:` → plaintext の直接フォールバックは 2026-08-31 時点の upstream main に未収録
- Desktop Launcher が通常 install の process supervisor
- Upgrade 後は config/app だけでなく実 Runtime の version を確認する

---

# 38. 今回実機で確認済みだった成功パターン

参考として、2026-08-31 に確認できた環境では、**stock v4.0.6 ではなくローカル互換パッチを追加した v4.0.6** を使用している。

```text
releaseVersion: 4.0.6
mode: full
appName: Codex Native2
browserHost: launcher
solAvailable: true
autoApproveToolCalls: false
experimentalBiggerContext: true
subagentsProtocol: compatibility-v1
localCompatibilityPatch: ocx1 bridge compaction -> plaintext before native passthrough
```

実動作:

- ChatGPT Web → Codex shell 成功
- file create 成功
- file modify 成功
- apply_patch 成功
- delete 成功
- normal Codex model 成功
- `/healthz` で Runtime 4.0.6 を確認
- Launcher app / config / runtime process の 4.0.6 一致を確認

### v4.0.5 → v4.0.6 で実際に踏んだ落とし穴

更新直後、Launcher app と config は patched 4.0.6 だったが、11時間以上前から起動していた stock 4.0.5 Runtime が残っていた。

その状態では、既存 session で Web model → native Codex model に切り替えると、旧 Runtime が処理して `ocx1... could not be decrypted or parsed` が再発した。

Launcher / Codex を完全終了して patched 4.0.6 Runtime に切り替えた後、version coherence を確認することが重要。

### この互換パッチを手順書へ入れる理由

別環境へこの runbook だけを渡して stock Release の導入だけで終了すると、Full harness 自体は
構築できても、ChatGPT Web の長い session から通常 Codex model へ切り替える際に同じ
`invalid_encrypted_content` が再発する可能性がある。

そのため「Full harness が動く」と「このMacと同じ model-switch 耐性がある」は別の完了条件として扱う。

この章の path、Tunnel ID、token、ユーザー名は他環境へコピーしない。

---

# 39. 最後の自己監査

完了報告前に AI 自身が以下を答える。

```text
Q1. 最新 README を作業開始時に確認したか？
Q2. Release version を推測せず確認したか？
Q3. Connector identity を README から確認したか？
Q4. 旧 Connector を変更していないか？
Q5. 秘密情報を出力していないか？
Q6. 本番 repo をテストに使っていないか？
Q7. Browser-only を先に確認したか？
Q8. shell/file/patch を実 tool result で確認したか？
Q9. normal Codex の回帰確認をしたか？
Q10. doctor の重大 error が 0 か？
Q11. rollback 方法を確認したか？
Q12. 残件を隠して「完了」と言っていないか？
Q13. stock upstream と local compatibility patch の違いを確認したか？
Q14. Web → native Codex の model-switch 回帰テストを実施したか？
Q15. このMacと同じ構成が目的なら Bigger Context / subagents protocol まで一致確認したか？
```

1つでも NO なら、完了報告の前に解消する。

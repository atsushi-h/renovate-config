# renovate-config

複数プロジェクトで使い回す [Renovate](https://docs.renovatebot.com/) の共有プリセット。

言語非依存の共通設定を `default.json` に置き、技術スタック固有の設定はプリセットとして分けている。
プロジェクト側は必要なものだけ `extends` して、差分だけ上書きする。

## 使い方

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": [
    "github>atsushi-h/renovate-config",
    "github>atsushi-h/renovate-config:pnpm",
    "github>atsushi-h/renovate-config:biomejs",
    "github>atsushi-h/renovate-config:go"
  ]
}
```

## プリセット一覧

| 参照 | ファイル | 内容 |
|---|---|---|
| `github>atsushi-h/renovate-config` | `default.json` | **共通ベース（必須）**。`config:best-practices` + ラベル / コミット規約 / automerge ポリシー |
| `...:pnpm` | `pnpm.json` | 更新後に `pnpm dedupe` |
| `...:biomejs` | `biomejs.json` | Biome の customManager + グループ化 |
| `...:react` | `react.json` | `react` / `react-dom` / `@types/react*` を 1 PR にまとめる |
| `...:react-native` | `react-native.json` | Expo / RN。major は Dependency Dashboard 承認制。`react` を内包する |
| `...:go` | `go.json` | `go mod tidy` + indirect 依存の単独更新を抑止 |
| `...:terraform` | `terraform.json` | provider をまとめて 1 PR |
| `...:docker` | `docker.json` | base image の major は automerge しない |

### `default.json` の中身

- ベースは `config:best-practices`。`config:recommended` に加えて以下が有効になる
  - `security:minimumReleaseAgeNpm` — npm パッケージは公開から 3 日待つ
  - `helpers:pinGitHubActionDigests` — GitHub Actions を commit SHA で固定
  - `docker:pinDigests` — Docker イメージを sha256 で固定
  - `:pinDevDependencies` — devDependencies を exact version に固定
  - `:maintainLockFilesWeekly` — 毎週月曜に lockfile メンテ PR（下記のとおり `schedule` は上書きしている）
  - `:configMigration` — 非推奨記法の自動移行 PR
  - `abandonments:recommended` — メンテ放棄パッケージの警告
- コミットは `chore(deps): update X to Y` 形式（`:semanticCommits` で明示的に有効化）
- `patch` と GitHub Actions は automerge（squash / platform automerge）
- 脆弱性アラートは automerge しない
- PR 上限は `prConcurrentLimit: 5` / `prHourlyLimit: 5`
- `timezone: "Asia/Tokyo"`（Renovate の既定は UTC。`schedule` の曜日・時刻判定に効く）
- `lockFileMaintenance.schedule` を `["* * * * 1"]`（月曜のみ・時間帯不問）に上書き

## 実行タイミングについて

**`schedule` はこのプリセットに含めていない。** 実行タイミングは各プロジェクトの
`.github/workflows/renovate.yml` の cron で管理する。

self-hosted Renovate（`renovatebot/github-action`）ではゲートが 2 段になるため。

| 段 | 制御するもの |
|---|---|
| workflow の `cron` | Renovate プロセスが**起動**するタイミング |
| `renovate.json` の `schedule` | 起動したプロセスが**ブランチ / PR を作ってよい**時間帯 |

両方を満たしたときだけ PR が出る。cron の起動時刻が `schedule` の枠外だと **PR が一切作られない**。
GitHub Actions のスケジュール実行は数分〜数十分遅延するのが常態なので、狭い `schedule` を併用すると
遅延で枠を外して PR が出ない事故も起きる。

`lockFileMaintenance.schedule` を緩めているのも同じ理由。`:maintainLockFilesWeekly` の既定
`["* 0-3 * * 1"]` は cron の起動時刻と重ならず、**定期実行では一度も発火しない**。
時間帯を外して「月曜のみ」にすることで、月曜の起動時に確実に走るようにしている。

cron の例（GitHub Actions の cron は UTC 固定）:

```yaml
on:
  schedule:
    - cron: '0 22 * * *' # 毎日 7:00 JST
  workflow_dispatch:
```

## プロジェクト側でオーバーライドする

```jsonc
{
  "extends": [
    "github>atsushi-h/renovate-config",
    "github>atsushi-h/renovate-config:pnpm"
  ],

  // スカラーはそのまま上書きされる
  "prConcurrentLimit": 10,

  // ラベルを「足す」場合は addLabels を使う（labels は置換されるため）
  "addLabels": ["frontend"],

  // packageRules は連結され、後ろのルールが勝つ。
  // 例: プリセットの patch automerge を無効化する
  "packageRules": [
    { "matchUpdateTypes": ["patch"], "automerge": false }
  ],

  // プリセットを丸ごと外す（ネストされた内部プリセットも指定できる）
  "ignorePresets": [
    "github>atsushi-h/renovate-config:docker",
    "helpers:pinGitHubActionDigests"
  ]
}
```

### マージ規則

Renovate は設定オプションごとにマージ方法が違う（`lib/config/utils.ts` の `mergeChildConfig`）。

| 対象 | 挙動 |
|---|---|
| `packageRules` | **連結**（プリセット → プロジェクトの順）。後ろが勝つので、プロジェクト側に書けば自動的に優先される |
| `addLabels` / `postUpdateOptions` / `customManagers` / `ignoreDeps` | **連結**。足したいものだけ書く |
| `labels` / `reviewers` / `assignees` | **置換**。プロジェクト側で書くとプリセットの値が消えるので、原則 `addLabels` を使う |
| `vulnerabilityAlerts` などのオブジェクト | **深いマージ**。一部キーだけ上書きできる |
| スカラー（`prConcurrentLimit` など） | プロジェクト側が勝つ |

## 検証

```bash
npx --yes --package renovate@latest -- renovate-config-validator --strict *.json .github/renovate.json
```

> `renovate@latest` を明示しないと npx のキャッシュで古いメジャーバージョンが使われることがある。

CI（`.github/workflows/validate.yml`）でも同じチェックを実行している。

ただし **`renovate-config-validator` はリモートプリセットを解決しない**（存在しないプリセットを
書いても通ってしまう）。プリセットが実際に解決されるか、マージ後の設定がどうなるかを確認するには、
プロジェクトのディレクトリで dry-run する。

```bash
GITHUB_COM_TOKEN=$(gh auth token) npx --yes --package renovate@latest -- \
  renovate --platform=local --dry-run=extract --print-config
```

> Renovate 44 は Node 26 では動かない（`Unsupported node environment`）。Node 24 で実行すること。

## このリポジトリ自体の Renovate

自分のプリセットを dogfooding している。管理対象は `.github/workflows/` の GitHub Actions のみ。

- 設定: `.github/renovate.json`（`github>atsushi-h/renovate-config` を extends）
- 実行: `.github/workflows/renovate.yml`（毎朝 7:00 JST の cron + 手動実行）
- Actions は `helpers:pinGitHubActionDigests` により commit SHA で固定し、更新は automerge される

設定ファイルを**ルートの `renovate.json` ではなく `.github/renovate.json` に置いている**のは、
`default.json` を消したときにルートの `renovate.json` が default preset として拾われ、
自分自身を extends する状態になるのを避けるため。

> 動かすには GitHub App をこのリポジトリにインストールし、リポジトリシークレットに
> `MY_RENOVATE_APP_ID` と `MY_RENOVATE_APP_PRIVATE_KEY` を登録する必要がある。

## 注意点

- **プリセットはデフォルトブランチから即座に全プロジェクトに読まれる。** 壊れた設定を main に入れると全プロジェクトの Renovate が同時に止まるので、必ず CI を通してからマージする
- default preset のファイル名は `default.json`。`renovate.json` は default preset 名としては非推奨
- Biome 用プリセットのファイル名を `biome.json` にしない。Biome 本体の設定ファイル名と衝突し、`customManagers:biomeVersions` の対象ファイル名にもなるため `biomejs.json` としている
- best-practices の導入直後は pin 系の初回 PR がまとまって出る（Actions の SHA 固定、Docker digest 固定、devDeps の exact 固定）
- Docker の digest 更新は updateType が `digest` なので `patch` の automerge 対象外。自動マージしたい場合は `docker.json` に `{ "matchUpdateTypes": ["digest"], "automerge": true }` を足す
- `:pinDevDependencies` は npm publish するライブラリには不向き。該当する場合はプロジェクト側で `ignorePresets` する
- プリセット内から同リポジトリの別プリセットを参照する場合も `github>atsushi-h/renovate-config:pnpm` のように完全形で書く
- プロジェクト側で `schedule` を書かない（理由は「実行タイミングについて」を参照）
- このリポジトリを private にすると、Renovate を動かしている GitHub App にこのリポジトリへの読み取り権限を追加する必要がある

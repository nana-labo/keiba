# MCP 構成メモ（keiba）

## 前提：このリポジトリの位置づけ

`nana-labo/keiba` は **`nana-labo/keiba-yosou` から `deploy <sha>` で吐き出される公開用の出力**。
ロジック・データ・スクレイパはすべて `keiba-yosou` 側にある（HEAD `a8b2b3a` = `deploy a8b2b3a`）。
MCP を検討するときは `keiba-yosou` 側の実装を見てから判断すること。

## 何を繋いだか

`.mcp.json` に **Playwright MCP**（`@playwright/mcp`）を1つだけ登録している。

## 収集経路の現状（`keiba-yosou` を実際に読んで確認）

| 対象 | 現状 | MCP の要否 |
|---|---|---|
| 過去成績・血統（`db.netkeiba.com`） | `data/scraper/`（requests＋EUC-JP、`data/raw` にキャッシュ、礼儀的スリープ）。`data/keiba.sqlite` に races 51,189 / runs 144,185 / features 21,177 | **不要**。静的HTMLで完結しており解決済み |
| 最終オッズ | `data/analytics/fetch_odds.py`（`--dryrun` / `--apply`） | **不要**。自動化済み |
| 当日の馬場・含水率 | prepost タスクが netkeiba 出馬表と JRA 馬場情報を直接取得。2ソース一致を必須化 | **原則不要**。ただし 2026-08-07 に含水率の誤取得が起きている |
| 当日馬体重 | netkeiba 出馬表の「馬体重(増減)」を馬番主キーで取得。`---.-` は未発表として埋めない | **原則不要** |
| **X（Step 5・スキルが必須と規定）** | **コードなし・自動化なし。`site:x.com` 付き WebSearch のスニペットのみ** | **ここだけ Playwright が要る** |

つまり Playwright MCP を入れる理由は「netkeiba が取れないから」ではない。**netkeiba 側はほぼ解決済み**で、
残っているのは **X だけがログイン必須で構造的に取れていない**という一点。

スキルは Step 5 を「フル・中モードで必須」「取らずに印を組み立てるのは半分目を閉じて予想するのと同じ」
と規定しているのに、そこだけ実装が無い。水曜夜〜木曜朝に出る調教評価・回避情報がまるごと
スニペット頼みになっている。ここが今いちばん薄い。

## 使う前に

初回はブラウザを起動して X に手でログインする。Playwright MCP は既定で永続プロファイルを使うため、
ログイン状態は次回以降に引き継がれる（`--isolated` を付けると毎回破棄されるので付けない）。

取得先の利用規約とレートには配慮する。人が読む速度を超える連続取得はしない。

## この設定が効く範囲

`.mcp.json` は **cwd がこのリポジトリの Claude Code セッション**にのみ適用される。
実際に予想を生成しているのは `keiba-yosou` 側なので、運用に効かせるなら
**同じ内容を `keiba-yosou` の `.mcp.json` かユーザー設定に置く**必要がある。
このファイルは推奨構成の正本・判断の記録として残している。

## 見送ったもの

- **AccuWeather**（認証不要・claude.ai コネクタ）— 気温／降水確率／風は構造化して取れるが、
  実運用で効く**含水率・クッション値は JRA 公式にしか無く**、prepost タスクが既に取りに行っている。
  重複投資。
- **Firecrawl** — netkeiba は自前スクレイパで解決済み。第三者サービス経由にする利点がない。
- **Airtable / Notion / Neon** — 回顧・学習の蓄積は `data/keiba.sqlite`、週次ログ `tasks/lessons.md`、
  および R1〜R50 のルール台帳 `references/evaluation-criteria.md`（スキル同梱・chmod 400）で既に成立している。
  外部DBを足すと正典が二重になる。
- **Vercel** — 公開は GitHub Pages（`.github/workflows/pages.yml`）で完結。

## 注意

`main` は `deploy <sha>` コミットで機械更新される。このファイル群を `main` に入れる場合、
デプロイ側が全体同期をしていると消える可能性がある。

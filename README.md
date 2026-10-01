# mantrst

`mantrst` はローカルに入っている man ページを、OpenAI互換のLLMで翻訳して表示するRust製CLI。llama.cppルータと Google AI Studio の Gemma に対応する。
元の man ファイルは変更しない。翻訳結果はキャッシュへ保存する。

## 動かし方

`llama-server` のルータを起動する。既定ではルータ内の `translategemma-4b` を選ぶ。

```sh
cargo build --release
./target/release/mantrst ddcutil
```

PATH に入れるなら、リポジトリ直下でこれだけでいい。

```sh
cargo install --path .
```

## Man page

The English man-page source is [man/man1/mantrst.1](man/man1/mantrst.1).
Test it without installing system-wide files:

```sh
MANPATH="$PWD/man" man mantrst
MANPATH="$PWD/man" mantrst mantrst
```

既定プロバイダは llama.cpp。既定の接続先は `http://127.0.0.1:8080/v1/chat/completions`、既定モデル名は `translategemma-4b`。
環境変数で変更できる。

```sh
MANTRST_MODEL=my-local-model mantrst --lang ja-JP ddcutil
MANTRST_LLM_URL=http://localhost:8080/v1/chat/completions mantrst ddcutil
```

現在は OpenAI互換の Chat Completions API を使う。API が利用できない場合は、エラーにして原文を勝手に「翻訳済み」として表示しない。
キャッシュがないときは、LLM への要求を始める前に使用モデル名を含む進捗（例: `mantrst: 翻訳中… (translategemma-4b)`）を標準エラーへ表示する。各リクエストも番号付きで表示し、接続または応答が60秒を超えるとエラーにする。

未キャッシュの翻訳対象は、プロバイダに応じて送る。llama.cpp はコンテキストに収まる最大サイズ（推定入力1,600トークン）でまとめ、Google AI Studio の Gemini は段落単位の通常テキスト要求を使う。いずれも応答を段落ごとにキャッシュするため、中断後は完了済み段落を再利用できる。
Gemini が一時的な高負荷（HTTP 429 / 503）を返した場合は、1秒、2秒と待って最大3回まで自動再試行する。
段落ごとの翻訳もキャッシュするので、途中で中断した長いページは次回実行時に続きから処理する。キャッシュは接続先 URL とモデルごとに分かれる。

```sh
mantrst --lang ja-JP ddcutil      # 翻訳して表示
mantrst --source ddcutil          # 翻訳済みテキストを標準出力へ
mantrst --rebuild ddcutil         # ページ全体だけを段落キャッシュから再構築
mantrst --refresh ddcutil         # 段落を含む全キャッシュを無視して作り直す
mantrst --original ddcutil        # 通常の man をそのまま起動
mantrst -s 1 printf               # man セクション指定
```

## Google AI Studio の Gemma 4

Google AI Studio で API キーを作成し、シェルにだけ設定する。キーを設定ファイルやリポジトリへ保存する必要はない。

```sh
export GEMINI_API_KEY='AI Studio で作成したキー'
mantrst --provider gemini ddcutil
```

`--provider gemini` は Google の OpenAI 互換エンドポイント、`gemma-4-26b-a4b-it`、および `GEMINI_API_KEY` を使う。より大きいモデルを使う場合は `--model gemma-4-31b-it` を指定する。キーは Gemini を選んだときだけ Bearer 認証として送るので、llama.cpp に戻すときにローカルサーバーへ渡らない。

通常表示では、端末に直接出している場合だけ `MANPAGER`、`PAGER`、`less -R` の順に pager を起動する。パイプした場合と `--source` では pager を使わない。
端末表示は、manのヘッダーと色付き見出し・コマンド名を付けた軽い `glow` 風テーマになる。`--source` は装飾なしのテキストを出力する。
テーマは既定で `COLORFGBG` を見て自動選択する。判定できない端末ではライトテーマになる。明示する場合は `--theme light` / `--theme dark`、または `MANTRST_THEME=light` / `dark` を使う。

```sh
PAGER=cat mantrst ddcutil         # pager を使わず表示
mantrst ddcutil | less -R         # 明示的に pager へ渡す
```

## 辞書

辞書は `rust/glossaries/` に置く TOML。読み込み順は次の通り。

```text
common.toml → ja.toml → ja-JP.toml
```

ユーザー辞書は `~/.config/mantrst/glossaries/` に同じファイル名で置くと、同名エントリを上書きできる。

## 安全性と対象範囲

- `.SH` / `.SS` の見出しと通常の説明段落だけを翻訳する。
- roff マクロ、オプションらしい行、表、コード例は維持する。
- LLM へ渡す roff エスケープはトークンに退避して復元する。
- man ページの内容はデータであり、LLMへの命令ではないと明示する。

これはMVPなので、複雑な roff マクロを多用するページは原文と見比べて確認すること。

## ライセンス

GPL-2.0-or-later（man-db と同じ）。詳細は `LICENSE` を参照。

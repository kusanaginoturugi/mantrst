# mantrst

English | [日本語](README.ja.md)

`mantrst` is a Rust CLI that translates locally installed man pages with an OpenAI-compatible LLM and displays the result. It supports a llama.cpp router and Gemma on Google AI Studio.
The original man files are never modified. Translations are stored in a cache.

## Usage

Start the `llama-server` router. By default, `translategemma-4b` is selected from the router.

```sh
cargo build --release
./target/release/mantrst ddcutil
```

To put it on your PATH, run this from the repository root:

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

The default provider is llama.cpp. The default endpoint is `http://127.0.0.1:8080/v1/chat/completions` and the default model name is `translategemma-4b`.
Both can be changed with environment variables.

```sh
MANTRST_MODEL=my-local-model mantrst --lang ja-JP ddcutil
MANTRST_LLM_URL=http://localhost:8080/v1/chat/completions mantrst ddcutil
```

It currently uses the OpenAI-compatible Chat Completions API. If the API is unavailable, it fails with an error instead of showing the original text as if it were "translated".
When there is no cache, it prints a progress message including the model name (e.g. `mantrst: 翻訳中… (translategemma-4b)`) to stderr before sending requests to the LLM. Each request is also numbered, and a connection or response taking longer than 60 seconds is treated as an error.

Uncached text is sent according to the provider. For llama.cpp, it is batched into the largest chunk that fits in the context (an estimated 1,600 input tokens); for Gemini on Google AI Studio, plain-text requests are sent per paragraph. In both cases responses are cached per paragraph, so completed paragraphs are reused after an interruption.
If Gemini returns a temporary overload (HTTP 429 / 503), it retries automatically up to 3 times, waiting 1 second, then 2 seconds.
Because per-paragraph translations are cached, a long page that was interrupted resumes where it left off on the next run. The cache is separated per endpoint URL and model.

```sh
mantrst --lang ja-JP ddcutil      # translate and display
mantrst --source ddcutil          # write translated text to stdout
mantrst --rebuild ddcutil         # rebuild only the whole page from the paragraph cache
mantrst --refresh ddcutil         # ignore all caches, including paragraphs, and redo
mantrst --original ddcutil        # run plain man as is
mantrst -s 1 printf               # specify a man section
```

## Gemma 4 on Google AI Studio

Create an API key in Google AI Studio and set it only in your shell. There is no need to store the key in a config file or the repository.

```sh
export GEMINI_API_KEY='key created in AI Studio'
mantrst --provider gemini ddcutil
```

`--provider gemini` uses Google's OpenAI-compatible endpoint, `gemma-4-26b-a4b-it`, and `GEMINI_API_KEY`. For a larger model, pass `--model gemma-4-31b-it`. The key is sent as Bearer authentication only when Gemini is selected, so it never reaches your local server when you switch back to llama.cpp.

In normal display mode, a pager is started only when writing directly to a terminal, trying `MANPAGER`, `PAGER`, then `less -R`. No pager is used when piping or with `--source`.
Terminal output uses a light `glow`-like theme with a man header and colored headings and command names. `--source` outputs undecorated text.
The theme is chosen automatically from `COLORFGBG` by default, falling back to the light theme when it cannot be detected. To set it explicitly, use `--theme light` / `--theme dark` or `MANTRST_THEME=light` / `dark`.

```sh
PAGER=cat mantrst ddcutil         # display without a pager
mantrst ddcutil | less -R         # pipe to a pager explicitly
```

## Glossaries

Glossaries are TOML files in `rust/glossaries/`. They are loaded in this order:

```text
common.toml → ja.toml → ja-JP.toml
```

Placing a file with the same name in `~/.config/mantrst/glossaries/` overrides entries with the same key.

## Safety and scope

- Only `.SH` / `.SS` headings and regular description paragraphs are translated.
- roff macros, option-like lines, tables, and code examples are preserved.
- roff escapes are replaced with tokens before being sent to the LLM and restored afterwards.
- The prompt states explicitly that man page content is data, not instructions to the LLM.

This is an MVP, so compare pages that make heavy use of complex roff macros against the original.

## License

GPL-2.0-or-later (same as man-db). See [LICENSE](LICENSE) for details.

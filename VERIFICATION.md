# setup-ruby bundler-cache 検証結果

## 概要

`ruby/setup-ruby`の`bundler-cache: true`を使用した際、異なるGitHub Actionsランナー間でキャッシュが共有されることにより、C拡張を持つgemのロードに失敗する問題を検証した。

## 前提

- `ruby/setup-ruby`のキャッシュキーは `setup-ruby-bundler-cache-v6-{OS}-{ARCH}-{RUBY}-...` の形式
- `ubuntu-latest`と`ubuntu-slim`は共にUbuntu 24.04であり、**キャッシュキーが一致する**
- mysql2 gemはC拡張を持ち、ビルド時にリンクされた共有ライブラリ(.so)に依存する
- GitHub Actionsのキャッシュは同一キーでは上書きされない（最初に作成されたものが優先）

## 検証環境

| 項目 | 値 |
|------|-----|
| Ruby | 3.3 / 4.0.0 / 4.0.2 |
| Bundler | 4.0.0 |
| Gem | mysql2 0.5.7 |
| ランナー | ubuntu-latest (Ubuntu 24.04.3), ubuntu-slim (Ubuntu 24.04.4) |
| リポジトリ | [Yuhi-Sato/setup-ruby-playground](https://github.com/Yuhi-Sato/setup-ruby-playground) |

## 検証1: ubuntu-latest → ubuntu-slim でキャッシュ復元

### 手順

1. ubuntu-latestでbundle install（mysql2がプリインストール済みの`libmysqlclient-dev`にリンクされてビルド）
2. `bundler-cache: true`によりキャッシュ保存
3. ubuntu-slimで`libmariadb-dev`をインストールし、同キャッシュを復元
4. `Bundler.require`でmysql2をロード

### 結果: 失敗

```
Bundler::GemRequireError: There was an error while trying to load the gem 'mysql2'.
Gem Load Error is: libmysqlclient.so.21: cannot open shared object file: No such file or directory
```

### 原因

- ubuntu-latestでビルドされたmysql2.soは`libmysqlclient.so.21`にリンクされている
- ubuntu-slimには`libmysqlclient.so.21`が存在しない（`libmariadb-dev`は`libmariadb.so.3`を提供）
- キャッシュキーが同一のため、ubuntu-latest製のバイナリがそのまま復元される

### ldd出力の比較

**ubuntu-latest (create-cache):**
```
libmysqlclient.so.21 => /usr/lib/x86_64-linux-gnu/libmysqlclient.so.21
```

**ubuntu-slim (consume-cache):**
```
libmysqlclient.so.21 => not found
```

## 検証2: キャッシュなしでubuntu-slimで新規ビルド

### 手順

1. 全キャッシュを削除
2. ubuntu-slimで`libmariadb-dev`をインストールし、キャッシュなしでbundle install
3. `Bundler.require`でmysql2をロード

### 結果: 成功

キャッシュがない場合、mysql2はubuntu-slim上で`libmariadb.so.3`にリンクされてビルドされるため、正常にロードできる。

## 検証3: ubuntu-slim → ubuntu-latest でキャッシュ復元（逆方向）

### 手順

1. 全キャッシュを削除
2. ubuntu-slimでbundle install（mysql2が`libmariadb.so.3`にリンクされてビルド）、キャッシュ保存
3. ubuntu-latestで同キャッシュを復元し、mysql2をロード

### 結果: 失敗

```
LoadError: libmariadb.so.3: cannot open shared object file: No such file or directory
```

### 原因

- ubuntu-slimでビルドされたmysql2.soは`libmariadb.so.3`にリンクされている
- ubuntu-latestには`libmariadb.so.3`がプリインストールされていない
- キャッシュキーが同一のため、逆方向でも同じ問題が発生する

## 検証4: Ruby 4.0.0での再現確認

### 仮説

Ruby 3.3 → 4.0.0に変更すると、bundleパスが`vendor/bundle/ruby/3.3.0` → `vendor/bundle/ruby/4.0.0`に変わる。このパス変更によりキャッシュの不整合が発生して失敗するのではないか。

### 手順

1. 全キャッシュを削除
2. `verify-cache.yml`の`ruby-version`を`4.0.0`に変更
3. ワークフローを実行（create-cache → consume-cache）

### 結果: 失敗（ただし仮説とは異なる原因）

```
Bundler::GemRequireError: There was an error while trying to load the gem 'mysql2'.
Gem Load Error is: libmysqlclient.so.21: cannot open shared object file: No such file or directory
  - vendor/bundle/ruby/4.0.0/gems/mysql2-0.5.7/lib/mysql2/mysql2.so
```

### 分析

- bundleパスは確かに`vendor/bundle/ruby/4.0.0`に変わっている
- しかし、setup-rubyのキャッシュキーにはRubyバージョンが含まれている（`ruby-4.0.0`）ため、Ruby 3.3のキャッシュとは別キーになり、パス変更による不整合は発生しない
- 失敗の原因はパス変更ではなく、**検証1と同じ`libmysqlclient.so.21`の不在**
- キャッシュキーが`ubuntu-24.04-x64-ruby-4.0.0-...`で両ランナー共通のため、ubuntu-latestでビルドされたバイナリがそのままubuntu-slimに復元された

### 結論

**仮説は棄却。** Rubyバージョンの変更自体はキャッシュの不整合を引き起こさない。Rubyバージョンに関係なく、根本原因はランナー間のシステムライブラリの差異にある。

## 検証5: Rubyバージョンアップ後に正常動作していたワークフローが壊れるシナリオ

### 仮説

ubuntu-slimのワークフローはキャッシュ削除後に何度実行しても成功していた。しかしRubyのバージョンアップ後、別のワークフロー（ubuntu-latest）が先に新バージョンのキャッシュを作成してしまうことで、ubuntu-slimのワークフローが失敗するようになる。

### 手順

1. 全キャッシュを削除
2. Ruby 4.0.0でubuntu-slimワークフロー（verify-no-cache）を実行 → **成功**（キャッシュ作成）
3. 同ワークフローを再実行 → **成功**（キャッシュヒットで正常動作を確認）
4. Rubyを4.0.2にバージョンアップ
5. verify-cacheワークフローを実行（create-cache → consume-cache）

### 結果

| ステップ | ワークフロー | ランナー | Ruby | 結果 |
|---------|-------------|---------|------|------|
| 2 | verify-no-cache | ubuntu-slim | 4.0.0 | 成功（新規ビルド・キャッシュ作成） |
| 3 | verify-no-cache | ubuntu-slim | 4.0.0 | 成功（キャッシュヒット） |
| 5-a | verify-cache / create-cache | ubuntu-latest | 4.0.2 | 成功（新キーでキャッシュ作成） |
| 5-b | verify-cache / consume-cache | ubuntu-slim | 4.0.2 | **失敗** |

### 失敗時のエラー

```
Bundler::GemRequireError: There was an error while trying to load the gem 'mysql2'.
Gem Load Error is: libmysqlclient.so.21: cannot open shared object file: No such file or directory
```

### 分析

1. Ruby 4.0.0では、ubuntu-slimが先にキャッシュを作成していたため、キャッシュは`libmariadb.so.3`にリンクされたバイナリを含んでいた → ubuntu-slimで正常動作
2. Ruby 4.0.2にバージョンアップすると、キャッシュキーが`ruby-4.0.0`→`ruby-4.0.2`に変わり、既存キャッシュは使われない
3. verify-cacheのcreate-cache（ubuntu-latest）が先に実行され、`libmysqlclient.so.21`にリンクされたバイナリで新キャッシュを作成
4. consume-cache（ubuntu-slim）がそのキャッシュを復元するが、`libmysqlclient.so.21`が存在しないため失敗

**仮説は検証された。** Rubyのバージョンアップはキャッシュキーをリセットするため、どのランナーが最初にキャッシュを作成するかの「競争」が再び発生する。ubuntu-latestが先にキャッシュを作成した場合、ubuntu-slimのワークフローが壊れる。

## 総括

### 根本原因

`ruby/setup-ruby`のキャッシュキーにランナーラベル（`ubuntu-latest` / `ubuntu-slim`）が含まれていない。両ランナーは同じUbuntu 24.04をベースとしているため、キャッシュキーが一致するが、プリインストールされているシステムライブラリが異なる。

### 影響

C拡張を持つgem（mysql2, pg, nokogiriなど）をビルドする際、リンクされる共有ライブラリがランナーによって異なる。キャッシュが共有されると、ビルド環境と実行環境の不整合により`LoadError`や`Bundler::GemRequireError`が発生する。

### 問題が顕在化するタイミング

- **Rubyのバージョンアップ時**: キャッシュキーがリセットされ、最初にキャッシュを作成するランナーが変わる可能性がある
- **キャッシュの有効期限切れ時**: 同様にキャッシュの再作成が発生する
- **新しいランナータイプの導入時**: 異なるシステムライブラリを持つランナーが追加された場合

### 対策案

1. **`cache-version`を利用してランナーごとにキャッシュキーを分離する**: setup-rubyの`cache-version`入力にランナーラベルを含める
2. **C拡張を持つgemを含むプロジェクトでは`bundler-cache: true`を使わず、手動でキャッシュを管理する**: `actions/cache`を使ってランナーラベルをキャッシュキーに含める
3. **全ランナーで同じシステムライブラリをインストールする**: ビルド環境を統一する

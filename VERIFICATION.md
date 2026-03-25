# setup-ruby bundler-cache 検証結果

## 概要

`ruby/setup-ruby`の`bundler-cache: true`を使用した際、異なるGitHub Actionsランナー間でキャッシュが共有されることにより、C拡張を持つgemのロードに失敗する問題を検証した。

## 前提

- `ruby/setup-ruby`のキャッシュキーは `setup-ruby-bundler-cache-v6-{OS}-{ARCH}-{RUBY}-...` の形式
- `ubuntu-latest`と`ubuntu-slim`は共にUbuntu 24.04であり、**キャッシュキーが一致する**
- mysql2 gemはC拡張を持ち、ビルド時にリンクされた共有ライブラリ(.so)に依存する

## 検証環境

| 項目 | 値 |
|------|-----|
| Ruby | 3.3 / 4.0.0 |
| Bundler | 4.0.0 |
| Gem | mysql2 0.5.7 |
| ランナー | ubuntu-latest (Ubuntu 24.04.3), ubuntu-slim (Ubuntu 24.04.4) |

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

Ruby 3.3 → 4.0.0に変更しても、同じエラーが再現することを確認。bundleパスは`vendor/bundle/ruby/4.0.0`に変わるが、キャッシュキーの不整合問題は同様に発生する。

## 結論

- `ubuntu-latest`と`ubuntu-slim`は同じUbuntu 24.04ベースのため、setup-rubyのキャッシュキーが一致する
- しかし、プリインストールされているシステムライブラリが異なる（`libmysqlclient` vs `libmariadb`）
- C拡張を持つgemのキャッシュを異なるランナー間で共有すると、共有ライブラリの不整合によりロードエラーが発生する
- この問題はどちらの方向でも発生する（ubuntu-latest → ubuntu-slim、ubuntu-slim → ubuntu-latest）
- **setup-rubyのキャッシュキーにランナーラベル（`ubuntu-latest` / `ubuntu-slim`）が含まれていないことが根本原因**

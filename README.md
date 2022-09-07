# docker-laravel-ci-layer-caching

LaravelのアプリケーションのテストをGitHub Actionsで行っています。docker compose upしてからテストを実行しているのですが、毎回イメージのダウンロードやビルドがされていて遅いです。

そのため、イメージやビルドしたもの（？←レイヤー）をキャッシュして高速化をしようと思いました。

## 調査

[satackey/action-docker-layer-caching](https://github.com/satackey/action-docker-layer-caching)がお手軽そうなの、これを使ってみようと思います。レイヤー、Buildkit、Buildxあたりの用語が分からないので、そのあたりを調べるとよさそうです。

- [GitHub Actions上でdocker composeを使ってCIを回すためにうまいことキャッシュする方法 - Qiita](https://qiita.com/yu-ichiro/items/c1a1248c0cdeeb0e6b42)
  - Buildxを使ってビルドし、Buildxの生成ファイルをactions/cacheでキャッシュする
  - ビルドしたイメージを別のジョブで使用するために、ローカルレジストリにアップロードして、ローカルレジストリのデータをactions/cacheでキャッシュする
    - 裏技的な使い方みたいです
- docker-layer-cachingっていうカスタムアクションがあるっぽい
  - [Github Actionsでdocker-composeのレイヤーキャッシュを使う方法](https://zenn.dev/fujisawa33/articles/94fe522852ad22)
  - これお手軽で良さそうなので、使ってみようと思います
- [Dockerに関するキャッシュたち](https://zenn.dev/masibw/articles/57a47a7381b9b3)

## やってみた

```yaml
name: backend-ci
on:
  push:
jobs:
  build-and-test:
    runs-on: ubuntu-22.04
    steps:
      - uses: actions/checkout@v3
      - run: docker compose pull
      - uses: satackey/action-docker-layer-caching@v0.0.11
        continue-on-error: true
      - run: make up
      - run: docker compose exec app composer install
      - run: make test
```

キャッシュがない場合

![](https://i.gyazo.com/e7362cac92971c392d9ec1e092b4fdbc.png)

キャッシュがある場合

![](https://i.gyazo.com/f79c200ce3a7e25950c17fe775df440a.png)

ビルドとコンテナの起動が、1m19sから36sに短縮されました。キャッシュから復元するのもそれなりに時間がかかるっぽいです。

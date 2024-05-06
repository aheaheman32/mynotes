# Asciidoctor環境構築

## 要求事項

- HTML, PDFの出力ができること
- 図を出力して埋め込められること HTML出力時URLで取得してくるのではなく、画像ファイルを出力してそのパスを指定する
- 図の出力はローカルで完結すること 外部サーバに送信しない
- プレビューを出しながら執筆ができること
- インストールが面倒でないこと(=環境を汚さないこと)
    - ruby版はインストールせずに、VSCodeとコンテナでがんばる

## 共通設定

VScode拡張機能

- AsciiDoc
- HTML Preview (HTMLが閲覧できるのであったほうがいい)
- Paste Image
- vscode-pdf

コンテナ

- asciidoctor/docker-asciidoctor
    - https://github.com/asciidoctor/docker-asciidoctor
    - asciidoctor, asciidoctor-diagram, asciidoctor-krokiなど
    - asciidoctor-diagramはダイアグラムのコードから画像を生成して埋め込んでくれるツール。
      ただし、krokiと同じでmermaidとかは別途mermaid-cliをインストールする必要があるので、これは使わずにkrokiを使う。
    - asciidoctor-krokiは別途画像変換サーバを指定することで、adoc内のダイアグラムコードをサーバに送って画像に変換してもらうリンクを生成して、HTML内に埋め込む(PDFは画像を埋め込む)。
      HTML出力時に別途画像を出力してそれを参照するようにもできる。

VSCodeのプレビューにkrokiを使う場合は以下も必須。

- yuzutech/kroki
    - https://github.com/yuzutech/kroki

さらに以下は必要に応じて追加する。mermaidは入れておいたほうがいい。

- yuzutech/kroki-mermaid
- yuzutech/kroki-bpmn
- yuzutech/kroki-excalidraw
- yuzutech/kroki-diagramsnet


## 構築手順

プレビューはVSCode拡張機能とkrokiサーバ、html/pdfはコンテナで行う。
コンテナを稼働させたら、`podman container attach <asciidoctorコンテナ>`でasciidoctorコンテナにアタッチして変換を行う。

### ポッドを作る場合

ポッドの場合だと複数コンテナで1コンテナのように振る舞うことができる(リソースを共有)。コンテナ間通信はlocalhost:portで行う。したがって、この段階でマウント、公開ポートの設定をする。

- ボリュームマウントでコンテナ側の方に`:z`が[必須](https://github.com/asciidoctor/docker-asciidoctor)
- Mermaidなどのコンテナサーバを公開するため、krokiサーバの8000以外も開けておく。https://github.com/yuzutech/kroki

```
podman pod create -v <workdir>:/documents:z -p 8000-8005:8000-8005 asciidoc-pod
```

コンテナを作る。

```
podman run -itd --name asciidoctor --pod asciidoc-pod asciidoctor/docker-asciidoctor
podman run -itd --name kroki --pod asciidoc-pod yuzutech/kroki
podman run -itd --name mermaid --pod asciidoc-pod yuzutech/kroki-mermaid
```

以降はpodで管理する。

```
podman pod stop asciidoc-pod
podman pod restart asciidoc-pod
# 起動中でもpodを停止して削除できる
podman pod rm -f asciidoc-pod 
```

Kubernetes YAMLに起こして、次から簡単に起動できるようにする。

```
# yamlにする
podman generate kube -f asciidoc-pod.yaml asciidoc-pod
# ポッド削除
podman pod rm -f asciidoc-pod 
# yamlから起動
podman play kube asciidoc-pod.yaml
```

docker環境ならdocker-composeでもいい。

### ポッドを作らない場合

krokiとmermaidサーバの通信をするためにネットワークを作る。

```
podman network create asciidoc-nw
```

コンテナを作る。

```
podman run -itd --name asciidoctor --network asciidoc-nw -v <workdir>:/documents:z asciidoctor/docker-asciidoctor
podman run -itd --name kroki --network asciidoc-nw -p 8000:8000 -e KROKI_MERMAID_HOST=mermaid yuzutech/kroki
podman run -itd --name mermaid --network asciidoc-nw -p 8002:8002 yuzutech/kroki-mermaid
```

## VSCodeの設定

`setting.json`に以下を追記する。

```
    "asciidoc.extensions.enableKroki": true,
    "asciidoc.preview.asciidoctorAttributes": {
        "kroki-server-url": "http://localhost:8000",
    },
```

注意
- VSCoodeのAsciidoc拡張機能でHTML出力すると、上記の設定をしてもHTMLはkroki.ioから取得される。
  そのため、HTML出力はコンテナで行う。

## HTMLで出力する場合

単に以下でよい。
`kroki-fetch-diagram=false`は画像を出力して、そのファイルをHTMLに参照させる。
これをなくすと`kroki-server-url`へ画像を取得するみたいな感じになってしまって都合が悪い。

```
asciidoctor \
-r asciidoctor-kroki \
-a kroki-server-url=http://localhost:8000 \
-a kroki-fetch-diagram=false \
sample.adoc
```

[次に書いてあること](#変換した画像の保存先について)をしなければ、adocファイルと同じ場所にHTMLと画像ファイルが生成される。

### 変換した画像の保存先について

ダイアグラムをsvgに変換したものを格納する場所を指定するには、adocファイルの最初に`:imagesdir: img`とフォルダを指定する。
これは単に画像保管場所を指定するもの。通常の画像の埋め込みは`imagesdir`からのパスになる(指定されていれば)。HTMLのsvgパスも`imagesdir`が先頭につく。
また、コマンドではなくて、adocファイルにつけることで、画像ファイルをインサートするときもこのディレクトリを基準とした補完になる。

参考
- https://docs.asciidoctor.org/asciidoc/latest/macros/images-directory/
- https://docs.asciidoctor.org/diagram-extension/latest/output/

### tips

- ダイアグラムの属性に`options=inline`を入れると、svgが直接HTMLに埋め込まれる。他に画像を貼り付けることがなければhtmlだけで完結する。 
    ```
    [plantuml,format=svg,options=inline]
    ....
    @startuml
    Alice -> Bob: Authentication Request
    Bob --> Alice: Authentication Response

    Alice -> Bob: Another よさおぴこ Request
    Alice <-- Bob: Another よろぴこ Response
    @enduml
    ....
    ```


## 日本語を含むPDFの出力の仕方

単に`-a pdf-theme=default-with-font-fallbacks`と指定すると、デフォルトではM+1pフォントがフォールバック(フォントに該当文字がないとき)として日本語表示に使われる。
ただし、ボールドとかがない。
そこで、自前で用意したttfフォントを使う場合の設定を以下に記述する。
また、pdfにsvg画像を埋め込むときに日本語が化けるのでその対処方法も載せる。

プロジェクトフォルダの下に、`theme`フォルダを作り、その中に`fonts`フォルダを用意する。(ここらへんは自由)

pdfで使いたいttsフォントを用意する。コード用のフォントも用意するとよい。
例えば以下のフォントをDLして`fonts`フォルダに入れる。

- M+ https://mplusfonts.github.io/
- 源真ゴシック http://jikasei.me/font/genshin/

次に、[ここ](https://github.com/asciidoctor/asciidoctor-pdf/blob/main/data/themes/default-theme.yml)からasciidoctor-pdfのデフォルトのテーマ(ymlファイル)をDLして、`theme`の中にいれる。

そして、`font-catalog`の部分を使うフォントに変更する。以下は例。フォントへのパスだが、`fonts`フォルダからの相対パスと思っていい。あとで、フォントフォルダのパスを指定するが`fonts`フォルダを指定する。

```
font:
  catalog:
    GenShinGothic-P:
      normal: GenShinGothic-P/GenShinGothic-P-Normal.ttf
      italic: GenShinGothic-P/GenShinGothic-P-Regular.ttf
      bold: GenShinGothic-P/GenShinGothic-P-Medium.ttf
      bold_italic: GenShinGothic-P/GenShinGothic-P-Bold.ttf
    GenShinGothic-Monospace:
      normal: GenShinGothic-Monospace/GenShinGothic-Monospace-Normal.ttf
      italic: GenShinGothic-Monospace/GenShinGothic-Monospace-Regular.ttf
      bold: GenShinGothic-Monospace/GenShinGothic-Monospace-Medium.ttf
      bold_italic: GenShinGothic-Monospace/GenShinGothic-Monospace-Bold.ttf
    MPLUS1:
      normal: MPLUS1/MPLUS1-Regular.ttf
      bold: MPLUS1/MPLUS1-SemiBold.ttf
      italic: MPLUS1/MPLUS1-Medium.ttf 
      bold_italic: MPLUS1/MPLUS1-Bold.ttf
    MPLUS1Code:
      normal: MPLUS1Code/MPLUS1Code-Regular.ttf
      bold: MPLUS1Code/MPLUS1Code-SemiBold.ttf
      italic: MPLUS1Code/MPLUS1Code-Medium.ttf
      bold_italic: MPLUS1Code/MPLUS1Code-Bold.tt
```

`base-font_family`(通常フォント)を上の`font-catalog`のフォントを指定する。例えば`GenShinGothic-P`。
`codespan-font_family`(コードのフォント)も`font-catalog`から指定する。

次に、svgの日本語文字化けについて、`theme`フォルダ下に`config.rb`を作り、以下を記載する。=>の隣は`font-catalog`のフォントを記載する。

```
Prawn::Svg::Font::GENERIC_CSS_FONT_MAPPING.merge!(
  'sans-serif' => 'GenShinGothic-P',
  'serif' => 'GenShinGothic-P'
)
```

pdfを出力する。`-a`の部分はadocファイルに代わりに記述してもいいが以下の項目は環境によりそうなのでコマンドの方で書く。

```
asciidoctor-pdf \
-r asciidoctor-kroki \
-a kroki-server-url=http://localhost:8000 \
-a allow-uri-read \
-a scripts=cjk \
-a pdf-theme=theme/mytheme.yml \
-a pdf-fontsdir=theme/fonts \
-r ./theme/config.rb \
sample.adoc
```

## Mermaidをsvgで出力する場合

asciidoctor-diagram、krokiのどちらの場合でも、mermaidがsvgで文字を出力するのにforeignObjectタグを使っていて、pdfに変換する際に使われるpwawn-svgがそれに対応していないので、エラーを吐いて完全には図が出力されない。

```
asciidoctor: WARNING: problem encountered in image: /tmp/image-20240505-135-h6oht5.svg; Must have attributes width, height on tag rect; skipping tag
```

[回避方法](https://github.com/yuzutech/kroki/issues/1410)として、Mermaidのコードブロックの最初に以下を記述する。

```
%%{ init: { "flowchart": { "htmlLabels": false } } }%%
```

この場合でも以下のエラーは出るが、問題なくPDFに図が出力される。

```
asciidoctor: WARNING: problem encountered in image: /tmp/image-20240505-135-h6oht5.svg; Invalid attributes on tag rect; skipping tag
asciidoctor: WARNING: problem encountered in image: /tmp/image-20240505-135-h6oht5.svg; Must have attributes width, height on tag rect; skipping tag
```

## tips

- htmlのフォントをサンセリフにする方法? スタイルシートを作らないといけなくてちょっと面倒?
- スタイルシートスキン集がここにある https://github.com/darshandsoni/asciidoctor-skins/tree/gh-pages
- Asciidoctor Web PDFというのもある。HTMLをcssで体裁を整えているならこっちでも良さげ? https://github.com/ggrossetie/asciidoctor-web-pdf?tab=readme-ov-file

## Asciidoc参考リンク集

### 書き方

- [公式ドキュメント](https://docs.asciidoctor.org/)
- [AsciidoctorとGradleでつくる文書執筆環境](https://h1romas4.github.io/asciidoctor-gradle-template/index.html)
  - [上記のgithub](https://github.com/h1romas4/asciidoctor-gradle-template)
- [AsciiDoc文書作成入門:: Asciidocによる文書作成環境の構築](https://itcweb.cc.affrc.go.jp/affrit/documents/guide/asciidoc/start)


### ツール

- [krokiマニュアル](https://docs.kroki.io/kroki/)
- ほかにもいろいろ

---
layout: post
title: "How to write new post on octopress"
date: 2013-08-14 14:14
comments: true
categories: OpenSource
---


ここでOctopressで新規記事を投稿する方法を紹介する。<br />
Octopressを知っていない方もいると思うが、始まる前に少しOctopressを紹介してから進めましょう。

Octopressとは？
---
[Octopress](http://octopress.org/)は最近人気上昇のブログエンジンである。特にhackerたちに流行している。膨大なWordPressに対して不満をもって、より簡単、hackerらしいブログツールを望んでいる中でOctopressを生み出した。<br />
Octopressはrubyとgitを利用して、[Markdown](http://ja.wikipedia.org/wiki/Markdown)記法のtextをhtmlページに変換する。[Github](https://github.com/)のpage serviceを利用して、簡単にブログを公開できる。Githubを利用しなくてもrsync、[Heroku](https://www.heroku.com/)のpage serivceでもブログを構築できる。<br />
ここでGithubのpage serviceを利用する前提で本ブログに投稿の作業を解説する。

環境設定
---
Windows環境はruby gemの相性がよくないので、Mac OSXまたはLinux環境を推奨する。<br />
ruby環境は[RVM](https://rvm.io/)と[rbenv](http://rbenv.org/)があるが、ここでMac OSXのrbenvを例として解説する。

Octopressの環境を構築する。[ここ](http://octopress.org/docs/setup/)を参考してください。<br />
事前準備：

  * Git
  * Ruby 1.9.3以上(rbenvでインストール)

``` bash
$ rbenv install 1.9.3-p194
$ rbenv versions # インストールされたrubyバージョンを確認
$ rbenv global 1.9.3-p194 # 1.9.3のバージョンを使用
$ rbenv version # 現在ruby versionを調べる
```

#### sjitechブログ取得
Githubからブログのソースを取得する。

``` bash
$ git clone https://github.com/sjitech/sjitech.github.com.git
$ cd sjitech.github.com
$ git checkout source # ブランチを切り替え
$ ruby --version # 現在ruby versionを調べる
$ gem install bundler # 依存ライブラリをインストール
$ rbenv rehash
$ bundle install
$ rake setup_github_pages # デプロイ準備フォルダーの作成
```

新規記事
---

#### 新記事の追加
``` bash
$ cd sjitech.github.com
$ rake new_post["記事タイトル"] # Creates source/_posts/yyyy-MM-dd-記事タイトル.markdown
```

#### markdown編集
markdownは幾つかのオンラインエディタ(例：[markable](http://markable.in/))を利用できる。Macなら[Mou](http://mouapp.com/)を強く推奨する。<br />
markdown文法は[JOHN GRUBERのブログ](http://daringfireball.net/projects/markdown/syntax.php)に参考ください。和訳は[ここ](http://blog.2310.net/archives/6)にご参考ください。<br />

##### コードブロック
Octopressにコードブロックは以下のように挿入する。

    ``` ruby
    puts 'hello markdown!'
    ```

上記のコードは以下のように表示される。

``` ruby
puts 'hello markdown!'
```

またはtagを使用する。
{% raw %}
    {% codeblock [title] [lang:language] [url] [link text] %}
    code snippet
    {% endcodeblock %}
{% endraw %}

例：
{% raw %}
    {% codeblock Time to be Awesome - awesome.rb %}
    puts "Awesome!" unless lame
    {% endcodeblock %}
{% endraw %}

以下のように表示する。

{% codeblock Time to be Awesome - awesome.rb %}
puts "Awesome!" unless lame
{% endcodeblock %}

##### 画像
tagを使う。

{% raw %}
    {% img [class names] /path/to/image [width] [height] [title text [alt text]] %}
{% endraw %}

例：

{% raw %}
    {% img http://www.sji-inc.jp/Portals/0/images/index/top-img02.jpg %}
    {% img left /images/sji_logo.png #2 %}
    {% img right /images/sji_outline.jpg Shinagawa Seaside East Tower #3 %}
{% endraw %}

中央寄せに表示する画像：<br />
{% img http://www.sji-inc.jp/Portals/0/images/index/top-img02.jpg %}<br />
<br />
{% img left /images/sji_logo.png #2 %}
左寄せに表示する画像<br />
左寄せに表示する画像<br />
左寄せに表示する画像<br />
左寄せに表示する画像<br />
左寄せに表示する画像<br />
左寄せに表示する画像<br />
左寄せに表示する画像<br />
左寄せに表示する画像<br />
{% img right /images/sji_outline.jpg Shinagawa Seaside East Tower #3 %}<br />
右寄せに表示する画像<br />
右寄せに表示する画像<br />
右寄せに表示する画像<br />
右寄せに表示する画像<br />
右寄せに表示する画像<br />

画像ファイルをsource/imagesフォルダーに置いてください。

##### 改行
markdownにhtmlタグも使えるが、極力的に避けたほうがいいと思う。改行の時に&lt;br /&gt;を使っても良い。

#### 途中の内容をプレビュー

``` bash
$ rake generate
$ rake preview
```

とすれば、[http://localhost:4000](http://localhost:4000)で確認できる。

投稿公開
---
記事を書き終えたら

``` bash
$ rake gen_deploy
$ rake deploy # sourceブランチをmasterブランチにマージし、githubにpushする
```

deploy途中にgithubアカウントとパスワードの入力を求める。

最後にsourceブランチも一緒にgithubにpushしましょう。

``` bash
$ git add .
$ git commit -m "add new post 記事タイトル"
$ git push origin source
```

では、さっそく記事を投稿しましょう。


参考リンク
---
- [Octopressのインストールから運用管理まで](http://tokkonopapa.github.io/blog/2011/12/30/octopress-on-github-and-bitbucket/)
- [Markdown記法](http://kojika17.com/2013/01/starting-markdown.html)


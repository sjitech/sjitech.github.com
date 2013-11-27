---
layout: post
author: Haidong Wang
title: "Chef soloとKnife soloで環境構築を自動化"
date: 2013-11-27 20:00
comments: true
categories: [DevOps]
tags: [Chef Ruby インフラ DevOps]
description: 最新のインフラツールChefで環境構築を自動化する方法を紹介
keywords: Chef, Ruby, DevOps, Infrastructrue, インフラ自動化
---

<p class="info">
最近の人気なインフラ自動化ツールChef soloを利用して、開発環境の構築を自動化してみます。<br />
</p>

<!-- more -->

## 参考サイト
最近Chef soloに関する記事はだいぶ増えています。本文は以下の記事を参考しました。


+ [Chef Soloと Knife Soloでの ニコニコサーバー構築 (2) 〜導入編〜](http://ch.nicovideo.jp/dwango-engineer/blomaga/ar322283)

+ [サーバー設定ツール「Chef」の概要と基礎的な使い方](http://knowledge.sakura.ad.jp/tech/867/)

+ [Chef Soloの正しい始め方](http://tsuchikazu.net/chef_solo_start/)

+ [knife-soloによるChefの実行](http://qiita.com/kidachi_/items/b222fb2892e6108c46d5)

+ [インフラストラクチャ自動化フレームワーク「Chef」の基本](http://www.atmarkit.co.jp/ait/articles/1305/24/news003.html)

そのほか、伊藤直也さんのブログでchef-soloを紹介され、[入門Chef Solo – Infrastructure as Code](http://tatsu-zine.com/books/chef-solo)が達人出版からも発売しています。

上の記事を読んでから、Chefの[公式ドキュメント](http://docs.opscode.com/)を読みやすくなると思います。結構の量がありますので、最初からドキュメントを読むとめちゃくちゃわかりにくいんです。

## Chefとは？
Chefは[Opscode社](http://www.opscode.com)によりRubyで開発されたインフラ自動化ツールです。ソースは[Github](https://github.com/opscode/chef)に公開サれています。利用者はRubyのDSLでサーバ環境構築の手順を定義し、Rubyスクリプトを実行してサーバ環境を構築できます。

Chefは「料理人」という言葉が由来になります。ChefはKnifeを使って、Recipeに従って、料理（サーバー構築）を作り上げます。構築対象サーバーは「Node」と呼ばれます。そのRecipeの管理単位を「Cookbook」と呼びます。CookbookにはRecipeを加え、設定ファイルのテンプレート「Template」や、環境に応じてその値を変更できる変数を定義した「Attribute」などが含まれています。後ほどそれらの要素を詳細に解説します。

### ClientとServer
大規模のサーバ群の設定を一括管理する場合、C/S構造のChefを利用します。Chef Serverは構築対象の情報とCookbookを集中管理し、Chef clientはChef serverからCookbookをダウンロードして実行することで自分の環境を構築します。事前にChef clientを構築対象サーバー（Node）でインストールします。

{% img /images/chef/chef_server_client.png 500 [Chef server and client architecture] %}

上記の図を示すようにインフラ作業者は自分のマシンでNode情報、Cookbookなどを作成し、Gitで管理します。作成したCookbookをChef serverに配布して、各NodeはChef serverから最新のCookbookなどを取得して、ローカルでChef clientを実行し、環境を構築します。

Chef serverはCouchDB、RabbitMQ、Solrなどのミドルウェアを使用しますが、構築手間がかかります。Opscode社はChefにいくつかの機能を追加した有償版の「Private Chef」やクラウド型の「Hosted Chef」といったサービスを提供しています。

### Chef soloとKnife solo
Chef soloはChef serverが不要、構築対象サーバー（Node）でCookbookの作成管理、実行などをすべて完結させるスタンドアロンChef clientです。多くの場合、構築対象サーバーはデータセンターに置かれますので（あるいはクラウドVMなど）、Cookbook作成管理はローカルマシンで行って、SSH経由でNodeに配布し、NodeにChef clientコマンドでCookbookなどを実行します。Knife soloはKnife機能を拡張し、Cookbookのリポジトリ作成、SSH経由のNodeに配布、リモートでNodeにChef client実行等を追加します。

{% img /images/chef/chef_solo.png 460 [Chef server and client architecture] %}

## 今回の環境
今回knife soloとchef soloで開発環境を構築してみましょう。

ローカル環境：

* Mac osx 10.9
  * Ruby: 1.9.3p448
  * Chef: 11.6.0 
  
構築対象サーバー(ホスト名： tcserver1)：

* Debian GNU/Linux 7.1 (wheezy) 32bit
  * Ruby: 1.9.3p194

また、SSH鍵認証およびdebain作業アカウントのsudo設定（NOPASSWD)を事前に実施しました。

## インストール
まず、ローカルマシンでchef soloとknife soloをいれましょう。

{% codeblock lang:bash %}
$ ruby -v
ruby 1.9.3p448 (2013-06-27 revision 41675) [x86_64-darwin12.4.0]
$ gem i chef --no-ri --no-rdoc            # chefをインストール（knifeを含む）
$ gem i knife-solo --no-ri --no-rdoc      # knife soloをインストール
$ gem i berkshelf --no-ri --no-rdoc       # cookbookを管理するツールBerkshelfをインストール
{% endcodeblock %}

### 作業手順
Knife soloは以下のコマンドがあります。init、prepareとcookをよく使っています。

{% codeblock lang:bash %}
$ knife solo
FATAL: Cannot find sub command for: 'solo'
Available solo subcommands: (for details, knife SUB-COMMAND --help)

** SOLO COMMANDS **
knife solo bootstrap [USER@]HOSTNAME [JSON]  (options)
knife solo clean [USER@]HOSTNAME
knife solo cook [USER@]HOSTNAME [JSON]  (options)          # 対象サーバーのChef clientをリモートで実行
knife solo init DIRECTORY                                  # Chefリポジトリの雛形を作成
knife solo prepare [USER@]HOSTNAME [JSON]  (options)       # 対象サーバーのChef client実行を準備（初回にChef clientをインストール）
{% endcodeblock %}

#### リポジトリの作成
「knife solo init リポジトリ名」でリポジトリの雛形を生成します。

{% codeblock lang:bash %}
$ knife solo init hellochef
Creating kitchen...
Creating knife.rb in kitchen...
Creating cupboards...
[wang@wang-mbp]-[0]  [/Users/wang/work]
$ tree hellochef/
hellochef/
├── cookbooks             # ライブラリなCookbook（Opscode communityからダウンロードされたもの）はこの中に入ります。
├── data_bags             # 主にパスワード情報などを保存しておく場所
├── nodes                 # 構築対象サーバーの情報はここに書いておきます
├── roles                 # 役割（Webサーバー、DBサーバーなど）ごとの定義を書きます
└── site-cookbooks        # 自分で作ったCookbook（セットアップ手順などを書いたRecipe）はこの中に入ります。
{% endcodeblock %}


#### 構築対象サーバー設定
「knife solo prepare wang@tcserver1」でtcserver1を構築対象サーバーとしてChef clientをインストールし、構築設定ファイルnodes/tcserver1.jsonを作成します。

{% codeblock lang:bash %}
$ knife solo prepare wang@tcserver1
Bootstrapping Chef...
Updating apt caches...
Installing required packages...
Installing rubygems from source...
......
Generating node config 'nodes/tcserver1.json'...
$ tree
.
├── cookbooks
├── data_bags
├── nodes
│   └── tcserver1.json
├── roles
└── site-cookbooks

5 directories, 1 file
$ cat nodes/tcserver1.json
{"run_list":[]}               # tcserver1.jsonの初期内容は空です
{% endcodeblock %}


#### サーバー構築
「knife solo cook wang@tcserver1」でtcserver1を構築してみましょう。（tcserver1.jsonは空なので、実際に何もしません）

{% codeblock lang:bash %}
$ knife solo cook wang@tcserver1
Running Chef on tcserver1...
Checking Chef version...
Uploading the kitchen...
Generating solo config...
Running Chef...
Starting Chef Client, version 11.6.0
Compiling Cookbooks...
Converging 0 resources
Chef Client finished, 0 resources updated
{% endcodeblock %}

##### Opscode communityに公開されたCookbookを取得
Cookbookを一から作成する必要ではありません。[Opscode community](http://community.opscode.com/cookbooks)に数多くのCookbookを公開しています。「git clone」でGithubから取得しますが、Berkshelfというツールを利用すれば、Rubyのbundler風に扱うことができます。

> Opscode CommunityからCookbookをダウンロードするため、ユーザ登録し、秘密鍵をダウンロードしておく必要があります。
> 秘密鍵をDownloadしたら、~/.chef/username.pemにパーミッション600で保存しておきましょう。更にリポジトリの.chef/knife.rbに以下の記述を追記します。
>
> > client_key    "/Users/wang/.chef/username.pem"
>

* まずリポジトリにBerksfileを作成します。

{% codeblock lang:bash %}
$ cat Berksfile
site:opscode

cookbook 'apt'
cookbook 'build-essential'
{% endcodeblock %}

* Berkshelfでcookbookをダウンロードします。

{% codeblock lang:bash %}
$ berks --version
Berkshelf (2.0.10)
$ berks install --path cookbooks/
Using apt (2.3.0)
Installing build-essential (1.4.2) from site: 'http://cookbooks.opscode.com/api/v1/cookbooks'
[wang@wang-mbp]-[0]  [/Users/wang/work/hellochef]
$ tree -d cookbooks/
cookbooks/
├── apt
│   ├── attributes
│   ├── files
│   │   └── default
│   ├── libraries
│   ├── providers
│   ├── recipes
│   ├── resources
│   └── templates
│       ├── debian-6.0
│       ├── default
│       └── ubuntu-10.04
└── build-essential
    ├── attributes
    └── recipes
{% endcodeblock %}


##### Cookbook新規作成
「knife cookbook create demo -o site-cookbooks」でdemoというCookbookの雛形をsite-cookbooksフォルダに新規作成します。

{% codeblock lang:bash %}
$ knife cookbook create demo -o site-cookbooks
** Creating cookbook demo
** Creating README for cookbook: demo
** Creating CHANGELOG for cookbook: demo
** Creating metadata for cookbook: demo
$ tree site-cookbooks/
site-cookbooks/
└── demo
    ├── CHANGELOG.md
    ├── README.md
    ├── attributes
    ├── definitions
    ├── files
    │   └── default
    ├── libraries
    ├── metadata.rb
    ├── providers
    ├── recipes
    │   └── default.rb
    ├── resources
    └── templates
        └── default
{% endcodeblock %}

demoにいくつかの子フォルダーを生成しました。各フォルダーの内容を以下に紹介しましょう。

## アーキテクチャと構成要素
上記のdemo cookbookのフォルダー構造によって、Chef soloの各要素を解説します。

### リポジトリ
リポジトリは「knife solo init」コマンドにより生成したChef client実行用のファイルの置く場です。料理人に対して、料理を作る場所（キチン）の意味です。
リポジトリに構築対象サーバーの情報（Nodes）と構築手順（Cookbooks）を含めます。

### Cookbooks
サーバー構築の作業手順（Recipe、Attribute、Template）をまとめるものです。サーバー環境を構築する時、様々なミドルウェア、システム設定及びアプリケーション・インストールの作業を行います。一般にあるソフトのインストールと設定を１つのRecipeに集約します。

ソフトの設定ファイルがNodeの情報により動的に生成される場合は多いですので、Rubyのテンプレート・ツールeRubyを利用して、設定ファイルの雛形を定義します。Chef clientを実行する時にNode環境と合わせる設定ファイルを生成します。

JavaのビルドツールAntと比較すると、build.xml(&lt;project /&gt;タグ)と同等なものです。

上記のdemoのフォルダーにいくつかの子フォルダーはありますが、その中にattributes、recipes、templatesをよく使っています。

#### Recipe
Recipeはサーバー構築手順を定義するものです。Cookbookに複数のRecipeを定義できますが、保守性の観点からRecipeにNode構築作業を適当に分割して、それぞれのRecipeに手順をまとめます。

例えば、LinuxでMysqlサーバーを構築する場合、システム設定（ベースシステムソフトGccなどのインストールと設定、ユーザアカウント管理）とMysqlのインストール、設定、サービス起動を別々のRecipeに記述したほうが良いかと思います。

Antと対照すると、Recipeは&lt;task /&gt;と同等なものです。&lt;task /&gt;に&lt;mkdir /&gt;、&lt;copy /&gt;、&lt;javac /&gt;などの内部[タスク](http://ant.apache.org/manual/tasklist.html)を組み合わせて、ビルド作業を記述しますが、Recipeでは様々な[Resource](http://docs.opscode.com/resource.html#resources)を組み合わせて、構築作業を記述します。

#### resource
構築作業の１つのステップです。ファイルコピー、パッケージ・インストール等の作業を定義します。ChefにあらゆるResourceを用意しますが、自分でRubyの関数を作成し、カスタマイズなResourceとして利用できます。

構築作業は冪等性（べきとうせい）を求めます。同じ作業を何度実行しても同じ結果になるべき特性です。それで、自分が作成したResourceまたはbashスクリプト再利用の場合、作業前のサーバー状態と作業後の状態をきちんと管理する必要です。

#### attribute
RecipeはRubyのコードですので、自由にRuby変数を定義できます。複数のRecipeの間に変数を共有する場合、Attributeを利用します。

例えば：

attributesにdefault連想配列を定義します。

{% codeblock lang:ruby site-cookbooks/demo/attributes/default.rb %}
  default["mysql"]["package_name"] = "mysql5-server"
{% endcodeblock %}

recipeにnode連想配列からdefaultの値を取得します。

{% codeblock lang:ruby site-cookbooks/demo/recipes/default.rb %}
  package node["mysql"]["package_name"] do
    action :install
  end
{% endcodeblock %}

Chefを実行する時に値を代入して、構築作業を行います。

{% codeblock lang:ruby 実行時のコード %}
package "mysql5-server" do
  action :install
end
{% endcodeblock %}

#### template
[eRuby](http://magazine.rubyist.net/?0017-BundledLibraries)で記述したソフトの設定ファイルの雛形です。Chef clientを実行する時、&lt;%= &gt;で囲む変数の値を代入し、構築対象サーバーの環境と合わせる設定ファイルを生成します。

例えば：

{% codeblock lang:xml site-cookbooks/demo/templates/hadoop/core-site.erb %}
  <property>
    <name>hadoop.tmp.dir</name>
    <value><%= @hadoop_tmp %></value>
    <description>A base directory for temporary directories.</description>
  </property>
{% endcodeblock %}

recipeのtemplate resourceの定義に変数hadoop_tmpの値を渡します。

{% codeblock lang:ruby site-cookbooks/demo/recipes/hadoop.rb %}
    template "/opt/hadoop/etc/hadoop/core-site.xml" do
        source "hadoop/core-site.erb"
		......

       variables(
           :hadoop_tmp  =>  "/opt/hadoop/tmp",
		    ......
       )

       not_if { File.exists?("/opt/hadoop/etc/hadoop/core-site.xml")}
    end
{% endcodeblock %}

Chefを実行して、生成した設定ファイルの内容は以下のようです。

{% codeblock lang:xml /opt/hadoop/etc/hadoop/core-site.xml %}
  <property>
    <name>hadoop.tmp.dir</name>
    <value>/opt/hadoop/tmp</value>
    <description>A base directory for temporary directories.</description>
  </property>
{% endcodeblock %}


### nodes
構築対象サーバーの情報を置く場所です。サーバーごとにjsonファイルで構築内容を記述します。



> 他にRoles、data_bags、Enviroments等のフォルダーも存在しますが、[公式ドキュメント](http://docs.opscode.com/)を参考すれば良いです。ここで割愛します。
> Rolesはサーバー役割（DBサーバかWebサーバー化）を分けて定義します。Enviromentは環境種類（開発環境か本番環境かテスト環境か）によって構築内容を別々記述します。
> data_bagsは暗号化したいDB接続パスワードなど情報を定義する場所です。



## サンプル
これから環境構築の一般な作業をChef soloで試してみましょう。

### Hello Chef

メッセージを出力します。

{% codeblock lang:ruby site-cookbooks/demo/recipes/default.rb %}
log 'message' do
    message "hello chef for tcserver setup"
    level :info
end
{% endcodeblock %}


### パッケージのインストール
ソフトをインストールします。Chef内部でDebian系（apt-get）とRedhat系(yum)の差異を吸収します。

{% codeblock lang:ruby site-cookbooks/demo/recipes/default.rb %}
%w(curl unzip tree wget nkf ctags).each do |pkg|
    package pkg do
        action :install
    end
end
{% endcodeblock %}


### JDKのインストール
JDKのインストールファイルをコマンドライン（wgetまたはcurl）でダウンロードできませんので、事前にブラウザでダウンロードして、site-cookbooks/demp/files/defaultに置いて、cookbook_file resourceを使用して、どこかにコピーして、JDKをインストールします。

例：
{% codeblock lang:bash %}
$ ls -l site-cookbooks/demo/files/default/
total 133M
drwxr-xr-x 4 wang  136 10 25 18:11 ./
drwxr-xr-x 4 wang  136 10 25 16:01 ../
-rw-r--r-- 1 wang 133M 10 25 18:12 jdk-7u45-linux-i586.tar.gz
{% endcodeblock %}

{% codeblock lang:ruby site-cookbooks/demo/recipes/default.rb %}
cookbook_file "/opt/jdk-7u45-linux-i586.tar.gz" do
    source           "jdk-7u45-linux-i586.tar.gz"
    mode             0644
    owner            "hadoop"
    group            "hadoop"
    action           :create_if_missing
    not_if           {File.exists?("/opt/jdk7")}
end
{% endcodeblock %}

### シェールスクリプトの例
[Resource](http://docs.opscode.com/resource.html#resources)に存在しない処理を行うため、bashスクリプトで構築作業を実現するのは普通の考えです。
上記のJDKをbashスクリプトでインストールしてみます。

{% codeblock lang:ruby site-cookbooks/demo/recipes/default.rb %}
bash "install_java_packages" do
    user "hadoop"
    cwd /opt
    code <<-EOH
        if ! ls /opt/jdk1.7.0_45 > /dev/null 2>&1; then
            tar zxf jdk-7u45-linux-i586.tar.gz
            ln -s /opt/jdk1.7.0_45 /opt/jdk7
            sudo chown -R hadoop:hadoop /opt/jdk1.7.0_45 /opt/jdk7
            rm -f jdk-7u45-linux-i586.tar.gz
        fi
	EOH
end
{% endcodeblock %}

> ここで冪等性を考慮する必要です。スクリプトを何回実行しても同じ結果を得るのは重要です。

Chefでは様々なResourceを用意しました。それぞれを組み合わせて、ほとんどのサーバー構築の作業を対応できます。以下ではよく使う便利なResourceです。

Resource           | 機能     
:------------------|:----------
users              | OSのユーザ管理      
group              | OSのグループ管理      
cookbook_file      | ファイル配布    
directory          | ディレクトリ管理
template           | テンプレートファイルを扱う
package            | パッケージ管理
service            | OSサービスの起動・停止・有効化・無効化
bash               | bashスクリプトの実行
cron               | cronジョブの管理

### Recipeの呼び出し
site-cookbooks/demo/recipes/default.rbはcookbook名（demo）を直接に呼びます。

{% codeblock lang:json nodes/tcserver1.json %}
{"run_list":["recipe[demo]"]}
{% endcodeblock %}

default以外のrecipe(site-cookbooks/demo/recipes/hadoop.rb)を::で引用します。

{% codeblock lang:json nodes/tcserver1.json %}
{"run_list":["recipe[demo::hadoop]"]}
{% endcodeblock %}

## まとめ
以上、Chefの概要や基本的な使い方を一通り説明しましたが、ここで紹介した機能はChefの機能のほんの一部です。また経験不足なので、今までのChef soloを使った感想をまとめます。

+ Chefを使って、文字記述の手順を省け、Rubyコードで環境構築を手順化します。大量サーバーの構築は楽になります。

+ Chefが便利なのは、Opscode communityやサードパーティが公開しているCookbookを流用することで、設定ファイルの記述を少なくできる点です。

+ ResourceやCookbookはOSやLinuxディストリビューションの種別が異なっても同じように利用できることから、管理対象のOSを問わず同じ手順で管理が行えます。

+ 公開されているCookbookやChefのソースは複雑になっており、Cookbookの依存関係を管理するツールもありませんので、動作内容の把握や問題発生時の解決に手間取ることもあります。

+ 個人的にすべてのファイルをRubyで記述し、jsonファイルを使わない方が良いかなと思います。なお、フォルダ構造は複雑すぎで、もっと簡単な構成を改善できるはずです。

+ attributesの変数をroles, recipe等に上書きできるのは大変わかりにくい仕組みです。

+ Recipeの記述自由度は高すぎて何が正しいか、あるいは何かベスト・プラクティスであるかよくわかっていません。

以上個人の見解ですが、間違い等あれば指摘していただきたいです。もっと深く勉強すれば、ぜひ[naoyaさんの本](http://tatsu-zine.com/books/chef-solo)を読んで頂ければと思います。


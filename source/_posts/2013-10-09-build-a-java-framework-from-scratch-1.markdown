---
layout: post
author: Haidong Wang
title: "Build a Java framework from scratch(1)"
date: 2013-10-09 15:02
comments: true
categories: [Java]
---

> 本連載はスクラッチで軽量Javaフレームワークの設計、実現方法を解説します。Javaの知識を深めながら、Spring FrameworkのようなAOPxDIフレームワークをゼロから作成してみます。<br />

自力でAOPとDI機能を実現するため、ある程度のJVM知識を習得する必要です。ですから、本題の前にJVM知識を紹介していきたいです。Java class file formatの解説を始め、JVMランタイム仕組みを紹介し、ASMフレームワークでJava classを操作する方法からAOPとDI機能の実装を展開します。<br />

<!-- more -->

では、早速[Java class file format](http://docs.oracle.com/javase/specs/jvms/se7/html/jvms-4.html)を解説します。以下の内容は[The Java® Virtual Machine Specification Java SE 7 Edition](http://docs.oracle.com/javase/specs/jvms/se7/html/index.html)を参照します。一部用語を英語のまま使用します。

## テスト環境
+ OS: Mac OSX 10.8.5
+ Java: Oracle 1.7.0_40(64bit)

## Java class format
バイナリJavaクラスファイルは以下の特徴があります。<br />

* ファイルは8ビット（1バイト）のストリームで構成されます。8ビット以上のデータはBig-Endianの順番で保存します。いわば、高いバイトは低いアドレスに保存されます。（IBMのPowerPCプロセッサはこの順番を採用します。Intelのx86プロセッサは逆順番のLittle-Endianを採用します）。
* クラスのレイアウトはC言語の構造体のような可変長配列で構成されます。主に2つのデータ・タイプ（符号なし整数とテーブル）があります。
    - **u1**: 符号なし8ビット整数
    - **u2**: Big-Endianバイト順の符号なし16ビット整数
    - **u4**: Big-Endianバイト順の符号なし32ビット整数
    - **テーブル**: いくつかの型の可変長の配列。テーブルのテーブル内の項目数はカウント数により識別されるが、テーブルのバイト内のサイズは項目それぞれを調査することのみで決定される。

specificationに記載されたJavaクラスの構造は以下のようです。

{% codeblock lang:c %}
ClassFile {
    u4             magic;                                    // マジックナンバー : 0xCAFEBABE
    u2             minor_version;                            // フォーマットのマイナーバージョン
    u2             major_version;                            // クラスファイルのメジャーバージョン
    u2             constant_pool_count;                      // クラス定数のプールの数量
    cp_info        constant_pool[constant_pool_count-1];     // クラス定数のプール
    u2             access_flags;                             // アクセスフラグ : 例えばクラスがabstractかstaticかなど
    u2             this_class;                               // 現在のクラス名
    u2             super_class;                              // スーパークラスの名前
    u2             interfaces_count;                         // 実現したインタフェースの数量
    u2             interfaces[interfaces_count];             // 実現したインタフェースの名前
    u2             fields_count;                             // クラスまたインスタンス変数の数量
    field_info     fields[fields_count];                     // クラスまたインスタンス変数
    u2             methods_count;                            // メソッド数量（親クラスからのメソッドが含まない）
    method_info    methods[methods_count];                   // メソッド
    u2             attributes_count;                         // クラス内の任意の属性の数量
    attribute_info attributes[attributes_count];             // クラス内の任意の属性（例えばソースファイル名、行番号など）
}
{% endcodeblock %}

コメントを見ればわかるかもしれませんが、少し説明します。constant_pool[constant_pool_count-1]を一見すると配列ですが、各要素の長さは必ず同じではありません。

* u4 magic: マジックナンバーです。このファイルはpng画像ファイルではなく、JavaソースをコンパイルしたJava classファイルであることを示します。4バイトの0xCAFEBABEで固定です。2.
* u2 major_version: 使用されるクラスファイルフォーマットのメジャーバージョン数です。
    * J2SE 7 = 51（0x33 十六進）
    * J2SE 6.0 = 50（0x32 十六進）
    * J2SE 5.0 = 49（0x31 十六進）
    * JDK 1.4 = 48（0x30 十六進）
* u2 constant_pool_count: 定数プールのカウントです。
* cp_info constant_pool[constant_pool_count-1]: 定数プールテーブル、リテラル数、文字列、そしてクラスやメソッドへの参照といった項目を含む、可変長の定数プールエントリです。合計エントリ（定数テーブルカウント - 1）数を含む、1から始まり索引付けされます。Java SE 7 Editionに14種類のcp_infoはあります。tagの値で区別します。

    種類                              | tag | 内容
   :----------------------------------|:---:|:------
   CONSTANT_Utf8_info                 | 1   | UTF-8 (Unicode) 文字列
   CONSTANT_Integer_info              | 3   | Integer : ビッグエンディアンフォーマットによる符号付き32ビット2の補数
   CONSTANT_Float_info                | 4   | Float : 32ビット単精度IEEE 754浮動小数点数
   CONSTANT_Long_info                 | 5   | Long : ビッグエンディアンフォーマットによる符号付き64ビット2の補数（定数テーブルの2つのスロットを占める）
   CONSTANT_Double_info               | 6   | Double : 64ビット倍精度IEEE 754浮動小数点数（定数テーブルの2つのスロットを占める）
   CONSTANT_Class_info                | 7   | クラス参照 : （内部フォーマットによる）完全修飾型クラス名を含むUTF-8文字列による定数テーブル内のインデックス（ビッグエンディアン）
   CONSTANT_String_info               | 8   | 文字列参照 : UTF-8による定数プール内のインデックス（ビッグエンディアン）
   CONSTANT_Fieldref_info             | 9   | フィールド参照 : 定数プール内にある2つのインデックス、最初はクラス参照で次は名前および型の記述（ビッグエンディアン）
   CONSTANT_Methodref_info            | 10  | メソッド参照 : 定数プール内にある2つのインデックス、最初はクラス参照で次は名前および型の記述（ビッグエンディアン）
   CONSTANT_InterfaceMethodref_info   | 11  | インタフェース参照 : 定数プール内にある2つのインデックス、最初はクラス参照で次は名前および型の記述（ビッグエンディアン）
   CONSTANT_NameAndType_info          | 12  | 名前および型の記述 : UTF-8による定数プール内のインデックス、最初は名前（識別子）を表し次は特別にエンコードされた型
   CONSTANT_MethodHandle_info         | 15  | Java SE 7からinvokedynamicの対応
   CONSTANT_MethodType_info           | 16  | Java SE 7からinvokedynamicの対応
   CONSTANT_InvokeDynamic_info        | 17  | Java SE 7からinvokedynamicの対応

   各cp_infoの詳細は後ほど使われる際に説明します。

* u2 access_flags: ビットマスクによるアクセスフラグです。

    フラグ         | 値     |  キーワード
    :--------------|:-------|:-------------
    ACC_PUBLIC     | 0x0001 | public
    ACC_FINAL      | 0x0010 | final
    ACC_SUPER      | 0x0020 | super
    ACC_INTERFACE  | 0x0200 | interface
    ACC_ABSTRACT   | 0x0400 | abstract
    ACC_SYNTHETIC  | 0x1000 | synthetic
    ACC_ANNOTATION | 0x2000 | annotation
    ACC_ENUM       | 0x4000 | enum

* u2 this_class: 定数プールにthisクラスの参照(CONSTANT_Class_info)

{% codeblock lang:c %}
CONSTANT_Class_info {
    u1 tag;                    // 7
    u2 name_index;             // 定数プールのインデックス
}
{% endcodeblock %}

* u2 super_class: 親クラスの参照(CONSTANT_Class_info)
* u2 interface_counts: 実現したインタフェース数
* u2 interface[interface_counts]: インタフェース参照(CONSTANT_Class_info)
* u2 fields_count:クラス変数とインスタンス変数の個数
* field_info fields[fields_count]:フィールド参照(field_info)

{% codeblock lang:c %}
field_info {
    u2             access_flags;                 // 変数のアクセスフラグ
    u2             name_index;                   // 変数名の定数プールのインデックス(CONSTANT_Utf8_info)
    u2             descriptor_index;             // 変数タイプの定数プールのインデックス(CONSTANT_Utf8_info)
    u2             attributes_count;             // 属性数
    attribute_info attributes[attributes_count]; // 属性参照(annotationなど)
}
{% endcodeblock %}

アクセスフラグは[ここ](http://docs.oracle.com/javase/specs/jvms/se7/html/jvms-4.html#jvms-4.5)を参照できます。

* u2 methods_count: メソッド数
* method_info methods[methods_count]: メソッド参照(method_info)

{% codeblock lang:c %}
method_info {
    u2             access_flags;                  // メソッドのアクセスフラグ
    u2             name_index;                    // メソッド名の定数プールのインデックス
    u2             descriptor_index;              // メソッド定義文字列の定数プールのインデックス
    u2             attributes_count;              // 属性数
    attribute_info attributes[attributes_count];  // 属性参照（annotation, excpetionなど)
}
{% endcodeblock %}

アクセスフラグは[ここ](http://docs.oracle.com/javase/specs/jvms/se7/html/jvms-4.html#jvms-4.6)を参照できます。

* u2 attributes_count: 任意の属性数
* attribute_info attributes[attributes_count]: 任意の属性参照

  attribute_infoはクラスファイルの最後に置かれます。例外、ソース行番号、デバッグ情報、annotationなどの標準属性以外、ユーザ定義の属性もあります。

  そして属性の長さは固定ではありません。属性にネスト属性を含むことも可能です。

## サンプル
話はやや複雑になりますが、簡単な例をあげます。
以下の簡単なクラスを作成します。シンプルなクラスなので、annotation, 例外処理などがありません。

{% codeblock 簡単なクラス - Sample.java %}
package net.codemelon.brisk.demo.jvm;

/**
 * Sample class for interpret JVM class file structure.
 *
 * @author : Haidong Wang
 */
public class Sample {

    private String name;

    public static final String AOP_CLASS_SUFFIX = "$$_brisk_aop_enhanced";

    protected int age = 30;

    public void init(String name, int age) {
        this.name = name;
        this.age = age;
    }

    public String getName() {
        return this.name;
    }

    public static String getAopClassSuffix() {
        return Sample.AOP_CLASS_SUFFIX;
    }
}
{% endcodeblock %}


jdk_1.7.0_40でコンパイルしたクラスは以下のようです。

{% img /images/brisk/sample_class_hex.png [Sample.class] %}


[Java仮想マシン仕様](http://docs.oracle.com/javase/specs/jvms/se7/html/jvms-4.html#jvms-4.6)を参照しながら、上記のバイトコードを解読してみましょう。

+ マジック・ナンバー<br />
クラスファイルの先頭4バイトは、Javaのクラスファイルであることを示すマジックナンバーで、0xCAFEBABE固定です。

    0000000: **cafe babe** 0000 0033 0021 0a00 0600 1b09  .......3.!......

+ バージョン番号<br />
次の4バイトは、クラスファイルが実行対象とするJavaバージョンを識別するバージョン番号です。前半2バイトがマイナー・バージョンで後半2バイトがメジャーバージョンとなります。<br />
以下は、マイナーバージョンが0（0x0000）、メジャーバージョンが51（0x0033）を表します。上のテーブルによって、Java SE 7のバージョン番号は51ですね。

    0000000: cafe babe **0000 0033** 0021 0a00 0600 1b09  .......3.!......

+ 定数プールの個数<br />


## まとめ





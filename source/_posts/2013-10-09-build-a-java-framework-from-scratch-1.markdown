---
layout: post
author: Haidong Wang
title: "Build a Java framework from scratch(1)"
date: 2013-10-09 15:02
comments: true
categories: [Java]
---

> 本連載はスクラッチで軽量Javaフレームワークの設計、実現方法を解説します。Javaの知識を深めながら、Spring FrameworkのようなAOPxDIフレームワークをゼロから作成してみます。<br />

自力でAOPとDI機能を実現するため、ある程度のJVM知識を習得する必要です。ですから、本題の前にJVM知識を紹介していきたいです。クラスレイアウト定義の解説を始め、JVMランタイム仕組みを紹介し、ASMフレームワークでJava classを操作する方法からAOPとDI機能の実装を展開します。<br />

<!-- more -->

では、早速[クラスレイアウト仕様](http://docs.oracle.com/javase/specs/jvms/se7/html/jvms-4.html)を解説します。以下の内容は[Java仮想マシン仕様SE 7版](http://docs.oracle.com/javase/specs/jvms/se7/html/index.html)を参照します。一部用語を英語のまま使用します。

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
    u2             major_version;                            // フォーマットのメジャーバージョン
    u2             constant_pool_count;                      // 定数プール数
    cp_info        constant_pool[constant_pool_count-1];     // 定数プール情報配列
    u2             access_flags;                             // アクセスフラグ : 例えばクラスがabstractかstaticかなど
    u2             this_class;                               // 現在のクラス名
    u2             super_class;                              // スーパークラスの名前
    u2             interfaces_count;                         // インタフェース数
    u2             interfaces[interfaces_count];             // インタフェース名前の配列
    u2             fields_count;                             // クラスまたインスタンス変数の個数
    field_info     fields[fields_count];                     // クラスまたインスタンス変数情報配列
    u2             methods_count;                            // メソッド数量（親クラスからのメソッドが含まない）
    method_info    methods[methods_count];                   // メソッド情報配列
    u2             attributes_count;                         // クラス内の任意属性の数量
    attribute_info attributes[attributes_count];             // クラス内の任意属性の情報配列（例えばソースファイル名、行番号など）
}
{% endcodeblock %}

コメントを見ればわかるかもしれませんが、少し説明します。constant_pool[constant_pool_count-1]を一見すると配列ですが、各要素のタイプと長さは異なっています。

* u4 magic: マジックナンバーです。このファイルはpng画像ファイルではなく、JavaソースをコンパイルしたJava classファイルであることを示します。4バイトの0xCAFEBABEで固定です。2.
* u2 major_version: 使用されるクラスファイルフォーマットのメジャーバージョン数です。
    * J2SE 7 = 51（0x33 十六進）
    * J2SE 6.0 = 50（0x32 十六進）
    * J2SE 5.0 = 49（0x31 十六進）
    * JDK 1.4 = 48（0x30 十六進）
* u2 constant_pool_count: 定数プールのカウントです。
* cp_info constant_pool[constant_pool_count-1]: 定数プールテーブル、リテラル数、文字列、そしてクラスやメソッドへの参照といった項目を含む、可変長の定数プールエントリです。
合計エントリ（定数テーブルカウント - 1）数を含む、1から始まり索引付けされます。Java SE 7 Editionに14種類のcp_infoはあります。tagの値で区別します。

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

    0000: **CA FE BA BE** 00 00 00 33 00 21 0A 00 06 00 1B 09 .......3.!......

+ バージョン番号<br />
次の4バイトは、クラスファイルが実行対象とするJavaバージョンを識別するバージョン番号です。前半2バイトがマイナー・バージョンで後半2バイトがメジャーバージョンとなります。<br />
以下は、マイナーバージョンが0（0x0000）、メジャーバージョンが51（0x0033）を表します。上のテーブルによって、Java SE 7のバージョン番号は51ですね。

    0000: CA FE BA BE **00 00 00 33** 00 21 0A 00 06 00 1B 09 .......3.!......

+ 定数プール数<br />
リテラル、実行時に解決するメソッド、フィールド参照、などの各種定数を持つ定数プールの個数です。定数プールは1から数えますので、Sampleクラスは32(0x21 - 1)個の定数があります。

    0000: CA FE BA BE 00 00 00 33 **00 21** 0A 00 06 00 1B 09 .......3.!......

+ 定数プールの情報配列<br />
定数プールのフォーマットは種類により異なります。種類は先頭1バイトのタグで決まります。<br />
定数プールの詳細をみってみましょう。まず1番目の定数をみます。

    0000: CA FE BA BE 00 00 00 33 00 21 **0A** 00 06 00 1B 09 .......3.!......

0x0aは10ですので、上のテーブルによってConstant_Methodref_infoの定数です。Constant_Methodref_infoの構造は以下のようです。


{% codeblock lang:c %}
CONSTANT_Methodref_info {
    u1 tag;                 // 10
    u2 class_index;         // メソッド所属クラスのインデックス
    u2 name_and_type_index; // メソッド定義の定数のインデックス
}
{% endcodeblock %}

class_indexは0x06です。定数プールの6番目のCONSTANT_Class_info定数を指します。

0000: CA FE BA BE 00 00 00 33 00 21 0A **00 06** 00 1B 09 .......3.!......

name_and_type_indexは0x1bです。定数プールの27番目のCONSTANT_NameAndType_info定数を参照します。<br />

0000: CA FE BA BE 00 00 00 33 00 21 0A 00 06 **00 1B** 09 .......3.!......

定数プールの27番のデータは以下のようです。

0140: 6C 65 2E 6A 61 76 61 **0C 00 0D 00 0E** 0C 00 0B 00 le.java.........

CONSTANT_NameAndType_infoの構造は以下のようです。

{% codeblock lang:c %}
CONSTANT_NameAndType_info {
    u1 tag;                    // 12
    u2 name_index;             // メソッド名または変数名の定数プールのインデックス
    u2 descriptor_index;       // メソッドまたは変数のメソッドの引数の型と個数，及び戻り値の型（voidを含む）
}
{% endcodeblock %}

上記の方法を従って、すべての定数を解読できます。バイナリは読みづらいですが、JDKに便利なのツールを提供しています。<br />
javap -version Sample.classでバイナリデータをJVMのアセンブリコードに変換します。定数プールの内容もリストアプされます。

{% codeblock lang:java [Sampleクラスの定数プール] %}
public class net.codemelon.brisk.demo.jvm.Sample
  SourceFile: "Sample.java"
  minor version: 0
  major version: 51
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #6.#27         //  java/lang/Object."<init>":()V
   #2 = Fieldref           #5.#28         //  net/codemelon/brisk/demo/jvm/Sample.age:I
   #3 = Fieldref           #5.#29         //  net/codemelon/brisk/demo/jvm/Sample.name:Ljava/lang/String;
   #4 = String             #30            //  $$_brisk_aop_enhanced
   #5 = Class              #31            //  net/codemelon/brisk/demo/jvm/Sample
   #6 = Class              #32            //  java/lang/Object
   #7 = Utf8               name
   #8 = Utf8               Ljava/lang/String;
   #9 = Utf8               AOP_CLASS_SUFFIX
  #10 = Utf8               ConstantValue
  #11 = Utf8               age
  #12 = Utf8               I
  #13 = Utf8               <init>
  #14 = Utf8               ()V
  #15 = Utf8               Code
  #16 = Utf8               LineNumberTable
  #17 = Utf8               LocalVariableTable
  #18 = Utf8               this
  #19 = Utf8               Lnet/codemelon/brisk/demo/jvm/Sample;
  #20 = Utf8               init
  #21 = Utf8               (Ljava/lang/String;I)V
  #22 = Utf8               getName
  #23 = Utf8               ()Ljava/lang/String;
  #24 = Utf8               getAopClassSuffix
  #25 = Utf8               SourceFile
  #26 = Utf8               Sample.java
  #27 = NameAndType        #13:#14        //  "<init>":()V
  #28 = NameAndType        #11:#12        //  age:I
  #29 = NameAndType        #7:#8          //  name:Ljava/lang/String;
  #30 = Utf8               $$_brisk_aop_enhanced
  #31 = Utf8               net/codemelon/brisk/demo/jvm/Sample
  #32 = Utf8               java/lang/Object
{% endcodeblock %}

定数プールに32個定数があることをわかりますね。#1から数えますが、#0はクラスが定数プールを参照しないことを示します。
ですから、constant_pool_countの値は33(0x21)です。

上の1番目の定数のclass_indexは#6ですが、java.lang.Objectのメソッドをわかります。

> 完全修飾されたJavaのクラス名は、「java.lang.Object」のように慣例的にドットで区分けされますが、Java仮想マシン内部形式は「java/lang/Object」のように、代わりにスラッシュを使用します。

name_and_type_indexは#27のCONSTANT_NameAndType_info(上記のコードによると&lt;init&gt;メソッド)です。コンパイルの時に自動生成したデフォールトコンストラクターですね。<br />
メソッドの引数と戻り値は()Vを指します。Java仮想マシン内部はデータ・タイプを1文字で表します。

BaseType Character | Type      | Interpretation
:------------------|:----------|:--------------
B                  | byte      | signed byte
C                  | char      | Unicode character
D                  | double    | double-precision floating-point value
F                  | float     | single-precision floating-point value
I                  | int       | integer
J                  | long      | long integer
LClassname;        | reference | an instance of class Classname
S                  | short     | signed short
Z                  | boolean   | true or flase
[                  | reference | one array dimension(一次元配列)
V                  | void      | return void


()Vは引数無し、戻り値無しのメソッドを表します。<br />
複雑な例をあげます。二次元配列String[][]は[[Ljava/lang/Stringを表します。int[][]なら[[Iを表します。

+ アクセスフラグ<br />
クラス宣言またはインタフェース宣言で使用する修飾子のビットマスクを表します。

01A0: 2F 4F 62 6A 65 63 74 **00 21** 00 05 00 06 00 00 00 /Object.!.......

Sampleクラスのアクセスフラグは0x0021 = 0x0001|0x0020（すなわち、ACC_PUBLIC|ACC_SUPER）です。<br />
ACC_PUBLICはpublicですが、ACC_SUPERはJDK 1.2以降強制的に追加された修飾子です。

+ this_class<br />
Sampleクラス情報のインデックスです。

01A0: 2F 4F 62 6A 65 63 74 00 21 **00 05** 00 06 00 00 00 /Object.!.......

定数プールの#5はCONSTANT_Class_info定数です。クラス名name_indexは#31のCONSTANT_Utf8_info定数を参照します。<br />
それによって、クラスの名がnet/codemelon/brisk/demo/jvm/Sampleであることはわかります。

+ 親クラス<br />

01A0: 2F 4F 62 6A 65 63 74 00 21 00 05 **00 06** 00 00 00 /Object.!.......

定数#6はjava/lang/Objectです。宣言していない場合、暗黙でjava.lang.Objectを継承しますね。


+ インタフェース<br />
インタフェース数とインタフェース情報配列はクラスを実現したインタフェース情報です。Sampleクラスはinterfaceがありませんので、interfaces_countは0x00です。


01A0: 2F 4F 62 6A 65 63 74 00 21 00 05 00 06 **00 00** 00 /Object.!.......

+ インスタンス変数とクラス変数<br />
Sampleクラスは2つのインスタンス変数と1つのクラス変数（合わせて3つ）があります。親クラスの変数を含まないことを注意してください。<br />
変数の構造は上のfield_infoです。

01A0: 2F 4F 62 6A 65 63 74 00 21 00 05 00 06 00 00 **00** /Object.!.......<br />
01B0: **03** 00 02 00 07 00 08 00 00 00 19 00 09 00 08 00 ................

以下は変数nameの内容を示します。

{% img /images/brisk/name_field_info.png [private String name] %}

もっと複雑な例としてクラス変数AOP_CLASS_SUFFIXを解析しましょう。

    public static final String AOP_CLASS_SUFFIX = "$$_brisk_aop_enhanced";

AOP_CLASS_SUFFIXのaccess_flagsはACC_PUBLIC | ACC_STATIC | ACC_FINAL (0x0001 | 0x0008 | 0x0010) = 0x19です。

01B0: 03 00 02 00 07 00 08 00 00 **00 19** 00 09 00 08 00 ................

name_indexは0x09です。上の定数一覧によって、#9 = Utf8 AOP_CLASS_SUFFIXです。

01B0: 03 00 02 00 07 00 08 00 00 00 19 **00 09** 00 08 00 ................

descriptor_indexは0x08ですが、上の定数一覧によって、#8 = Utf8 Ljava/lang/String;(java.lang.Stringインスタンス)です。

01B0: 03 00 02 00 07 00 08 00 00 00 19 00 09 **00 08** 00 ................

次のattributes_countは0x0001です。1つの属性はあります。

01B0: 03 00 02 00 07 00 08 00 00 00 19 00 09 00 08 **00** ................<br />
01C0: **01** 00 0A 00 00 00 02 00 04 00 04 00 0B 00 0C 00 ................

次の2バイトは0x000aです。上の定数プールの#10によると"ConstantValue"の属性です。<br />
[ConstantValue](http://docs.oracle.com/javase/specs/jvms/se7/html/jvms-4.html#jvms-4.7.2)の構造は以下のようです。

{% codeblock lang:c %}
ConstantValue_attribute {
    u2 attribute_name_index;
    u4 attribute_length;
    u2 constantvalue_index;
}
{% endcodeblock %}

attribute_name_indexはConstantValueの定数プールのインデックです。attribute_lengthは固定で2です。

01C0: 01 00 0A **00 00 00 02** 00 04 00 04 00 0B 00 0C 00 ................

constantvalue_indexは0x04です。定数プールによると、#4 = String #30 //  $$_brisk_aop_enhancedです。AOP_CLASS_SUFFIXの初期値ですね。

01C0: 01 00 0A 00 00 00 02 **00 04** 00 04 00 0B 00 0C 00 ................

他の変数は同じな方法で解析できます。

+ メソッド<br />
コンパイラで自動生成したコンストラクターを含めて、methods_countは0x04です。

01D0: 00 **00 04** 00 01 00 0D 00 0E 00 01 00 0F 00 00 00 ................

一番目のmethod_infoの内容を見てみましょう。method_infoの構造を上に参照できます。<br />
access_flagsは0x01です。

01D0: 00 00 04 **00 01** 00 0D 00 0E 00 01 00 0F 00 00 00 ................

[メソッドのアクセスフラグ一覧](http://docs.oracle.com/javase/specs/jvms/se7/html/jvms-4.html#jvms-4.6)によると、ACC_PUBLICは0x0001です。

01D0: 00 00 04 00 01 **00 0D** 00 0E 00 01 00 0F 00 00 00 ................

name_indexは0x0dです。上の定数プールによると、#13 = Utf8 <init>です。自動生成したデフォルト・コンストラクターです。

01D0: 00 00 04 00 01 00 0D **00 0E** 00 01 00 0F 00 00 00 ................

descriptor_indexは0x0eなので、定数プールの#14 = Utf8  ()Vです。デフォルト・コンストラクターはパラメータ無し、戻り値voidです。<br />
次のattributes_countは0x01です。

01D0: 00 00 04 00 01 00 0D 00 0E **00 01** 00 0F 00 00 00 ................

次の2バイトはattribute_name_indexです。定数プールの0x0fは#15 = Utf8 Codeです。<br />
それによって、属性タイプは[Code_attribute](http://docs.oracle.com/javase/specs/jvms/se7/html/jvms-4.html#jvms-4.7.3)です。

01D0: 00 00 04 00 01 00 0D 00 0E 00 01 **00 0F** 00 00 00 ................

Code_attributeの構造は以下のようです。

{% codeblock lang:c %}
Code_attribute {
    u2 attribute_name_index;
    u4 attribute_length;
    u2 max_stack;
    u2 max_locals;
    u4 code_length;
    u1 code[code_length];
    u2 exception_table_length;
    {   u2 start_pc;
        u2 end_pc;
        u2 handler_pc;
        u2 catch_type;
    } exception_table[exception_table_length];
    u2 attributes_count;
    attribute_info attributes[attributes_count];
}
{% endcodeblock %}

かなり複雑な構造ですね。メソッドに仮想マシンの命令を表す構造体です。<br />

01D0: 00 00 04 00 01 00 0D 00 0E 00 01 00 0F **00 00 00** ................<br />
01E0: **39** 00 02 00 01 00 00 00 0B 2A B7 00 01 2A 10 1E 9........*...*..

attribute_lengthはCode_attributeにattribute_name_indexとattribute_lengthを除いたバイト数です。<br />
上のバイトデータによると、&lt;init&gt;のCode_attributeの長さは0x39バイトです。

01E0: 39 **00 02 00 01** 00 00 00 0B 2A B7 00 01 2A 10 1E 9........*...*..

次のmax_stackは0x02です。max_localsは0x01です。JVMでメソッドをframeに実行されます。<br />
frameにローカル変数用の配列と操作命令スタックはあります。変数配列はメソッドパラメータ、ローカル変数（中間結果）を保存します。<br />
操作スタックは仮想マシンの命令と操作数（変数配列からロードされる）を順番でロードして実行します。結果を変数配列仁保存し、命令と操作数をクリアし、次の命令を処理します。<br />
メソッドのすべてのコードを実行する時、変数配列の最大長さはmax_localsと呼ばれます。操作スタックの最大長さはmax_stackと呼ばれます。<br />
doubleとlongのデータは64ビットなので、max_stackとmax_localsを計算するときに注意しなければなりません。<br />
詳しいJVMのランタイム仕組みは次回に解説させて頂きます。

01E0: 39 00 02 00 01 **00 00 00 0B** 2A B7 00 01 2A 10 1E 9........*...*..

code_lengthは0x0bです。メソッドコードはcode[11]に置かれます。Code_attributeの構成によって、codeタイプはu1です。<br />
u1の範囲は0x00 ~ 0xff(0 ~ 255)です。現在約200個のJVM命令を定義しています。<br />
exception_table_lengthとexception_tableは例外情報です。&lt;init&gt;は例外宣言がありませんので、exception_table_lengthは0です。

01F0: B5 00 02 B1 **00 00** 00 02 00 10 00 00 00 0A 00 02 ................

次のattributes_countは0x02です。

01F0: B5 00 02 B1 00 00 **00 02** 00 10 00 00 00 0A 00 02 ................

1つ目の属性のインデックは0x10です。定数プールの#16はUtf8 LineNumberTableです。

01F0: B5 00 02 B1 00 00 00 02 **00 10** 00 00 00 0A 00 02 ................

[LineNumberTable](http://docs.oracle.com/javase/specs/jvms/se7/html/jvms-4.html#jvms-4.7.12)の構造は以下のようです。


{% codeblock lang:c %}
LineNumberTable_attribute {
    u2 attribute_name_index;
    u4 attribute_length;
    u2 line_number_table_length;
    {   u2 start_pc;
        u2 line_number;
    } line_number_table[line_number_table_length];
}
{% endcodeblock %}

attribute_lengthはattribute_name_indexとattribute_length以外のバイト数です。<br />
バイトデータによって、attribute_lengthは10(0x0a)です。

01F0: B5 00 02 B1 00 00 00 02 00 10 **00 00 00 0A** 00 02 ................

line_number_table_lengthは0x02です。

01F0: B5 00 02 B1 00 00 00 02 00 10 00 00 00 0A *00 02** ................

line_number_tableは次の8バイトとです。start_pcはJVM命令の番号です。line_numberはソースの行番号です。

0200: **00 00 00 08 00 04 00 0E** 00 11 00 00 00 0C 00 01 ................


## まとめ





---
title: "【初心者向け】“探す”時間を減らしませんか？VS Code検索機能の活用方法"
emoji: "🐣"
type: "tech"
topics: ["vscode", "初心者", "拡張機能", "エディタ", "効率化"]
published: true
published_at: 2025-07-14 05:00
publication_name: "secondselection"
---

## はじめに

この記事では、プログラミング初心者でも安心して始められる無料のコードエディタ「Visual Studio Code（VS Code）」の**検索機能**の基本的な使い方をまとめました。
私自身エンジニア1年目で初学者の方の役に立てれば幸いです。

定番の機能や自然と発見できそうな機能から、私が会社の上司や先輩エンジニアにおすすめしてもらった機能・こっそり使い方を真似して覚えた便利な機能も含めて、今回ご紹介していきます。

> ✅ **この記事を読むのにかかる目安の時間：** およそ3～5分
> ✅ **一番読んでほしい項目：** Step 2

:::message

## 対象読者

- **プログラミング初心者・新人プログラマ・駆け出しエンジニア**
  あるいは **VS Codeをまだまだ使いこなせてない**という人向けです。

- 少し慣れてきた方は、**チェックリストとして**活用してもらえれば嬉しいです。

:::

## 前提条件

- VS Codeをインストールしていること。
- ショートカットキーはWindows環境を対象としています。
- テキストファイルが複数入っているフォルダを開いた状態でお試しください。

## 機能・用途・ショートカット(Windows)

| 機能               | Windows      | 主な用途                                   |
| ------------------ | ------------ | ------------------------------------------ |
| 単一ファイル内検索  | Ctrl+F        | 現在開いているファイル内をキーワードで検索 |
| 検索単語移動       | Shift+F3     | 検索した単語が複数ある場合移動が可能 　    |
| 全文検索           | Ctrl+Shift+F | フォルダ全体をキーワードで横断検索     |
| ファイル検索       | Ctrl+P       | ファイル名で全フォルダを高速ジャンプ       |

### 💡 Step 1: 単一ファイル内検索

- **ショートカットキー操作** `Ctrl+F` (Windows)
- 検索したい文字列を入力し、Enterを押すと一致する箇所にジャンプします。
- 一致する箇所が複数ある場合は、`F3` で次に移動、`Shift + F3` で前に戻ることができます。
- 検索は**現在開いているファイル内のみ**に適用されます。

![画像](/images/beginner-vscode/HelloWorld.png)
*[参照:公式リファレンス](https://code.visualstudio.com/docs/editing/codebasics#_seed-search-string-from-selection)*

:::details 📝Tips1：同一単語を複数選択できるショートカットもあります！

- 単語を選択した状態で、`Ctrl+D`を押すと一気に複数選択が可能です

:::

### 💡 Step 2: フォルダ全体を検索する方法

- **ショートカットキー操作** `Ctrl+Shift+F` (Windows)
- **サイドバー**「🔍」アイコンから「検索ビュー」を開くことも可能です

- 現在開いているフォルダ内のすべてのファイルが検索対象
- 他ファイルも横断して探したいときに使います
- コード・文章・コメント等あらゆる文字列を素早く検索可能です

![画像](/images/beginner-vscode/search_across_files.png)
*[参照:公式リファレンス](https://code.visualstudio.com/docs/editing/codebasics#_search-across-files)*

:::message

#### 検索履歴について

- 検索結果欄には履歴が残るので、↑↓キーで呼び出し・再利用可能です
- 関数名を思い出す手間も少し減らせます
- 度々コピー&ペーストを行ったり、打ち直す手間も減らすことができます

:::

> **【おすすめ度：⭐⭐⭐】**
> 左側の一覧画面から選択すれば、**行をクリック**して、
> 該当するファイルにジャンプできます。
> ![画像](/images/beginner-vscode/oneclick.png)

> **【おすすめ度：⭐⭐】**
> 画面左上の水色文字の"エディタで開く"を押すと、
> 右側に一覧が表示され、**ファイル名を選択してCtrl＋クリック**で
> 該当するファイルにジャンプできます。
> ![画像](/images/beginner-vscode/sample_search.png)

:::details 📝Tips2：検索ボックスの右側にある3つの検索オプションボタンを確認しよう！

![画像](/images/beginner-vscode/search_box.png)

(画面左) **大文字と小文字を区別**

- 大文字と小文字を区別します。

(画面中) **単語単位で検索**

- 検索語と完全に一致する単語のみを対象にします。
    (例：「apple」で検索した場合、「apples」はマッチしません)

(画面右) **正規表現**

- 文字列の規則性に応じて、記号で検索条件を指定できます。
    (例：「単語の形」「電話番号」「メールアドレス」など)
  
- 正規表現の基本の記号

| 記号   | 意味         | 例                      |
| ---- | ---------- | ------------------------------ |
| `.`  | どんな1文字でもOK | `a.c` → “abc” や “adc”など     |
| `*`  | 前の文字 0回以上  | `go*` → “g”, “go”, “goo”…      |
| `+`  | 前の文字 1回以上  | `go+` → “go”, “goo”、でも “g” はNO |

![画像](/images/beginner-vscode/search_regax.png)

:::

:::details 📝Tips3：どんなアイコンがあるか確認しよう！

**1. 検索アイコンの右上のボタンで検索がもっと便利になります。**

![画像](/images/beginner-vscode/search_icon.png)
> (画面左) 最新の情報に更新
> (画面中) 表示方法の切り替え **左:ツリー/一覧 右:折り畳み/展開**
> (画面右) 新しいエディターを開く

---

**2.『…』をクリックすると詳細設定ボックスの開閉ができ、さらに絞り込みができます。**

![画像](/images/beginner-vscode/openclose.png)

![画像](/images/beginner-vscode/search_more.png)
> (画面左) 含めるファイル：ワイルドカードをつかい**検索対象に含める**ファイルやフォルダーを指定。
> (画面中) 開いているエディターでのみ検索。
> (画面右) 除外するファイル：ワイルドカードをつかい**検索対象外の**ファイルやフォルダーを指定。
:::

### 💡 Step 3: ファイル名検索

- **ショートカットキー操作**：`Ctrl+P` (Windows)
- フォルダ名やファイル名での検索に便利な機能です

- VS Code内の画面真ん中上部に小窓が出現して、文字を入力すると候補が表示されます
- **部分一致やディレクトリ指定**（例:`src/app`）ですばやく絞り込みが可能
- サジェストも効くので、ディレクトリ階層を覚えてなくても目的のファイルへ行けます

![画像](/images/beginner-vscode/QuickFile.png)
*[参照:公式リファレンス](https://code.visualstudio.com/docs/editing/editingevolved#_quick-file-navigation)*

:::details 📝Tips4：ファイル名をすべて入力しなくてもOK！

🧩 例: config.yamlの検索。

- ショートカットキー操作:`Ctrl+P`(Windows)
- 'conf'と入力します
- `config.yaml`が提案されます

![画像](/images/beginner-vscode/search_config.png)

:::

### 💡 Step 4: 検索履歴の活用

- **検索BOXでの履歴呼び出し**
  - ↑↓を押すと過去の検索キーワードを遡ることができます。
  - 検索した単語が自動保存されているので、再入力不要ですぐ再検索ができます。

## 拡張機能おすすめ2選

「コードを読むときの便利な機能があれば...」「フォルダを履歴で探すのも量が増えて大変...」
そんなときに、情報を整理するために活用できるおすすめの拡張機能を2つご紹介します。

### 🌱 **1. Bookmarks** 【おすすめ度：⭐⭐】

![Bookmark](/images/beginner-vscode/Bookmarks.png)
> (画面左) 登録されたしおりの表示イメージ
> (画面右) ファイル内の行にしおりをつけたとき

- **登録方法**
  - 登録したい行を選択し、**ショートカット**`Alt+Ctrl+K` (Windows)
- **登録した情報の検索方法**
  - サイドバーに表示された「🔖」マークを選択後、行ごとに表示されます
  - 行にカーソルをあてて右横の「🖌」マークを選択後、名前の変更が可能です

:::message

**【活用事例】**

コード内に付箋や栞を付けたり、
マーカーを引くようなイメージで活用しています。

:::

:::details Visual Studio │ Marketplace

![Bookmark](/images/beginner-vscode/vscode_bookmarks.png)
*[参照:Visual Studio │ Marketplace](https://marketplace.visualstudio.com/items?itemName=alefragnani.Bookmarks)*
:::

### 🌱 **2. Project Manager**【おすすめ度：⭐⭐⭐】

![ProjectManager](/images/beginner-vscode/ProjectManager.png)
> (画面左) ワークスペース・フォルダを整理したときの表示イメージ
> (画面右) フォルダを設置するための設定ファイル

- **登録方法**
  - サイドバーの「📁」のアイコンをクリック
  - FAVORITESの行にマウスを移動させ「💾」マークをクリック
  - 画面上部真ん中の小窓内に、ファイル名が表示されることを確認し
    (登録名を変更したい場合は編集を行い) Enterを押す
  - タグ名は`no tags`として保存されるので、グルーピングを希望する場合は次の項目へ
  - FAVORITESの行にマウスを移動させ「🖌」マークをクリック
  - jsonファイルが表示されるので、`tags`の項目にタグ名を設定すればグルーピング可能

![json](/images/beginner-vscode/json_projectmanager.png)

:::message

**【活用事例】**

ディレクトリの構造とは別に、自由にカテゴリ分けができます。

お気に入り・使用頻度・顧客別など
好みに合わせて名前をつけて整理ができてすごく便利です。

:::

:::details Visual Studio │ Marketplace

![Project Manager](/images/beginner-vscode/vscode_ProjectManager.png)
*[参照:Visual Studio │ Marketplace](https://marketplace.visualstudio.com/items?itemName=alefragnani.project-manager)*
:::

## おわりに

初心者として学びながら書きました。どなたかの学びの助けになれば嬉しいです。
最後までご覧いただきありがとうございました。

## 参考

- Visual Studio Code（[Documentation for Visual Studio Code](https://code.visualstudio.com/docs/editor/codebasics#_search-across-files)）
- Visual Studio Marketplace([Extensions for Visual Studio Code](https://marketplace.visualstudio.com/vscode))

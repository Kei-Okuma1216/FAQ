# 質問・回答

<details><summary>ソース "NortonSecurity" からのイベント ID 36 の説明が見つかりません。</summary>

#### Q.
OSのイベントビューアに以下の情報レベルのアプリケーションログが取得された。これはどういう意味ですか？
ソース "NortonSecurity" からのイベント ID 36 の説明が見つかりません。このイベントを発生させるコンポーネントがローカル コンピューターにインストールされていないか、インストールが壊れています。ローカル コンピューターにコンポーネントをインストールするか、コンポーネントを修復してください。

イベントが別のコンピューターから発生している場合、イベントと共に表示情報を保存する必要があります。

イベントには次の情報が含まれています: 

NortonSecurity

メッセージ リソースは存在しますが、メッセージがメッセージ テーブルに見つかりませんでした。

#### A.
 このイベントビューアのメッセージは、**Norton Security に関連するイベント（ID 36）を表示しようとしたが、必要な「説明テキスト」が見つからなかった**ということを示しています。これは **エラーではなく、ログ表示の仕様に関する情報レベルの通知**です。

---

## 🔍 内容の分解と意味

### メッセージ概要：

> **「ソース 'NortonSecurity' からのイベント ID 36 の説明が見つかりません」**

このメッセージの意味は：

* **イベント自体は発生しています**（例: NortonSecurity の何らかの動作）
* しかし、**Windows 側にこのイベントの「表示用の翻訳情報（メッセージ定義）」が無い**ため、

  * イベントの**内容が人間に読める形で表示できない**
  * イベントログに保存された「NortonSecurity」というキーワードだけが表示される

---

## 🧾 原因のよくあるパターン

1. **Norton Security がアンインストールされているか、破損している**

   * イベントを生成したが、必要な「表示用DLL」が削除されている

2. **別のPCからリモートでイベントを取得している**

   * イベント発生元とログ表示しているPCが異なり、表示定義がない

3. **システムにメッセージDLLが登録されていない**

   * インストールされていても、正しくレジストリに登録されていないケース

---

## 🟡 これは問題なのか？

* **基本的には "情報" レベルのログであり、システムトラブルではありません**
* ただし、**Norton Security の設定や挙動が気になる場合**は以下の対応が有効です。

---

## ✅ 対処法

### 1. **Norton Security の修復／再インストール**

* コントロールパネル → プログラム → Norton を選択 → 修復 or 再インストール
* または、公式アンインストールツール＋再インストール：

  * 🔗 [https://support.norton.com/](https://support.norton.com/)

### 2. **イベントを無視する**

* このようなログは無害なことが多く、「記録はされているが詳細は読めないだけ」なので無視可能です。

---

## 🛡 まとめ

| 項目   | 内容                                        |
| ---- | ----------------------------------------- |
| 原因   | NortonSecurity のメッセージ定義ファイルが見つからない（または破損） |
| 影響   | イベント内容が読めないだけ。実害はなし（多くの場合）                |
| 推奨対応 | 気になる場合は Norton の修復または再インストール              |

---

このログが特定のタイミング（例えば Docker 起動時）と連動していた場合は、その時点の Norton の干渉ログとして意味があるかもしれません。そのような関連性を調べたい場合もお知らせください。


</details>

<details><summary>別のPCからリモートでイベントを取得している</summary>

#### Q.
別のPCからリモートでイベントを取得している

イベント発生元とログ表示しているPCが異なり、表示定義がない
これはどういう意味ですか？

#### A.
ご質問の「**イベント発生元とログ表示しているPCが異なり、表示定義がない**」について、以下に詳しく解説します。

---

## 💡 イベントビューアと「リモートのイベント」

### Windowsイベントビューアの基本動作

Windowsのイベントビューア（`eventvwr.msc`）は、通常その**ローカルコンピュータ上で発生したログ**を表示します。

しかし、以下のような操作を行うと「**リモートPCのログを取得して表示する**」こともできます：

* 同じネットワーク上の他のPCのイベントログを参照する（管理者権限が必要）
* 管理ツール（例：System Center、Remote Server Admin Tools など）でログを集約して表示する

---

## 🧩 「表示定義がない」とは何か？

Windowsのイベントログは2つの部分に分かれています：

1. **イベントのデータ本体（発生したエラーや情報そのもの）**
2. **メッセージ定義（人間が読める形式に翻訳するテンプレート）**

メッセージ定義は、通常は**ログを発生させたアプリケーションがローカルに持つ DLL や EXE に埋め込まれています**。

### 問題：

リモートPCで発生したログを、別のPCからイベントビューアで開くと…

➡ 表示用のメッセージ定義が**ローカルPC側に存在しない場合**、以下のような表示になります：

```text
ソース "NortonSecurity" からのイベント ID 36 の説明が見つかりません。
```

これは：

* ログ本体は取得できたが
* 表示に必要なリソースが自分のPCに無いため
* **意味の分からない数値や「説明が見つかりません」だけが出る**

---

## 📦 たとえ話

これは、**外国語の本だけ渡されて、翻訳辞書が無い**のと同じです。

* 本（イベントログの本体）は見えている
* 翻訳辞書（メッセージ定義DLL）が手元に無いので意味が分からない

---

## ✅ 対策（ただし高度）

この現象を解消したい場合：

1. **イベント発生元のPCにログインして直接イベントビューアで見る**
2. または、そのPCにインストールされているソフト（例：Norton）の **メッセージDLLをコピー・登録** する（非推奨・複雑）

一般的には「**そのまま無視**」か「**イベント発生PCで確認**」するのが現実的です。

---

## ✅ まとめ

| 項目       | 内容                               |
| -------- | -------------------------------- |
| 何が起きているか | 他のPCのログを参照しているが、表示に必要なリソースが手元に無い |
| 結果       | イベント内容が人間に読めない形式になる              |
| 対応策      | 無視してもよい。必要ならイベント発生元のPCで確認するのが確実  |

---

もし「そのログがどのPCのものか分からない」や、「ログ内容を正しく見る方法を知りたい」といったご希望があれば、さらに具体的にサポート可能です。


</details>

<details><summary>このPCにアクセスしてきたリモートPCの接続ログはどこにありますか？</summary>

#### Q.
このPCにアクセスしてきたリモートPCの接続ログはどこにありますか？

#### A.
Windowsにおいて、**このPCにアクセスしてきたリモートPCの接続ログ**を確認するには、**イベントビューアを使って特定のログカテゴリを調べる必要があります**。以下に、代表的なリモート接続ログの記録場所と確認方法を説明します。

---

## ✅ よく使われる「リモートアクセスの種類」

| アクセス手段              | 確認すべきログカテゴリ                              |
| ------------------- | ---------------------------------------- |
| リモートデスクトップ（RDP）     | セキュリティログ（ログオンイベント）                       |
| 共有フォルダアクセス          | セキュリティログ（ログオン + ファイルアクセス）                |
| PowerShell Remoting | Microsoft-Windows-PowerShell/Operational |
| 管理共有（\PC\C\$など）     | 同上、またはセキュリティログ                           |

---

## 📍 イベントビューアでの確認手順（RDPログインなど）

### ステップ 1：イベントビューア起動

* `Windows`キー → `eventvwr.msc` と入力して実行

---

### ステップ 2：**セキュリティログ**を開く

* 左側メニューから：

  ```
  Windowsログ > セキュリティ
  ```

---

### ステップ 3：ログオンイベントをフィルタする

#### 🔍 主なイベントID一覧

| イベントID   | 内容                  |
| -------- | ------------------- |
| **4624** | 成功したログオン（誰かがアクセス）   |
| **4625** | 失敗したログオン（パスワードミスなど） |
| **4648** | 明示的な資格情報の使用ログオン     |
| **4672** | 特権アカウントによるログオン      |

#### フィルタ例（成功ログオン）：

1. 「現在のログをフィルター」を選択
2. `イベントID`に `4624` を入力
3. 「OK」で絞り込み

---

### ステップ 4：ログの詳細を見る

ログを開くと、以下のような情報が確認できます：

| フィールド          | 意味                |
| -------------- | ----------------- |
| **アカウント名**     | 接続してきたユーザーアカウント   |
| **アカウントドメイン**  | ワークグループ名やドメイン     |
| **ログオンタイプ**    | 接続方法（RDP、ネットワーク等） |
| **ネットワークアドレス** | リモートPCのIPアドレス     |

#### 💡 ログオンタイプの意味（抜粋）

| タイプ番号 | 意味                 |
| ----- | ------------------ |
| `2`   | ローカルの対話型ログオン       |
| `3`   | ネットワーク経由（共有フォルダなど） |
| `10`  | リモートデスクトップ（RDP）    |

---

## ✅ 補足：PowerShell で確認する方法

以下のコマンドでもリモートアクセス履歴（ログオンイベント）を取得可能です：

```powershell
Get-WinEvent -LogName Security | Where-Object {
    $_.Id -eq 4624 -or $_.Id -eq 4625
} | Format-List -Property TimeCreated, Id, Message
```

---

## 🔐 注意点

* **監査ポリシーが有効でない場合、セキュリティログが記録されません。**
* ログが一定期間でローテーションされている場合、古い接続履歴は消えていることもあります。

---

## 📝 まとめ

| 接続種類         | 調べるログカテゴリ                 | 主なイベントID            |
| ------------ | ------------------------- | ------------------- |
| RDP・共有フォルダなど | Windowsログ > セキュリティ        | 4624, 4625, 4648 など |
| PowerShell経由 | PowerShell/Operationalログ  |                     |
| どこから来たか（IP）  | イベントログ内の「ネットワークアドレス」フィールド |                     |

---

ご希望があれば、**イベントIDの具体的な意味**や、\*\*接続ユーザーの追跡方法（監査ログの設定）\*\*も詳しくサポート可能です。


</details>

<details><summary>PowerShellコマンド実行をパイプしてテキスト保存したい</summary>

#### Q.
このコマンド実行後、eventlog.txtにパイプして保存したい。どのように記述すればよいですか？
Get-WinEvent -LogName Security | Where-Object {
    $_.Id -eq 4624 -or $_.Id -eq 4625
} | Format-List -Property TimeCreated, Id, Message

#### A.
`Get-WinEvent` コマンドの結果を `eventlog.txt` に保存するには、**リダイレクト（`>`）ではなく `Out-File` を使う**のが最も確実です。以下のようにコマンドを書いてください：

---

## ✅ 保存コマンドの完全な形式

```powershell
Get-WinEvent -LogName Security | Where-Object {
    $_.Id -eq 4624 -or $_.Id -eq 4625
} | Format-List -Property TimeCreated, Id, Message | Out-File -Encoding UTF8 -FilePath eventlog.txt
```

---

## 🧾 補足説明

| 部分                       | 説明                                                |
| ------------------------ | ------------------------------------------------- |
| `Out-File`               | 出力をファイルに書き込む PowerShell 専用コマンド（標準リダイレクト `>` より安定） |
| `-Encoding UTF8`         | 日本語などが含まれる場合に文字化けを防ぐ                              |
| `-FilePath eventlog.txt` | 保存先ファイル名（相対パスでも絶対パスでも可）                           |

---

## ✅ 例：デスクトップに保存する場合

```powershell
$path = "$env:USERPROFILE\Documents\eventlog.txt"
Get-WinEvent -LogName Security | Where-Object {
    $_.Id -eq 4624 -or $_.Id -eq 4625
} | Format-List -Property TimeCreated, Id, Message | Out-File -Encoding UTF8 -FilePath $path
```

---

## ✅ 出力を追記したい場合（オプション）

既存のファイルに追記したいときは `-Append` オプションを追加します：

```powershell
... | Out-File -Encoding UTF8 -FilePath eventlog.txt -Append
```

---

何かフィルタを追加したい（例: 日付指定やIPアドレスを含む行のみ）などのニーズがあれば、それも対応可能です。お気軽にどうぞ。


</details>

<details><summary>コマンドレットに期間の条件追加</summary>

#### Q.
$path = "$env:USERPROFILE\Documents\eventlog.txt"



Get-WinEvent -LogName Security | Where-Object {



    $_.Id -eq 4624 -or $_.Id -eq 4625



} | Format-List -Property TimeCreated, Id, Message | Out-File -Encoding UTF8 -FilePath $path



このコマンドレットに条件として、いまより1日前までという条件を追加してほしい。

#### A.
はい、PowerShellの`Get-WinEvent`コマンドレットに、現在時刻から1日前までの条件を追加します。

イベントの`TimeCreated`プロパティを使って日付フィルタリングを行います。

```powershell
$path = "$env:USERPROFILE\Documents\eventlog.txt"
$oneDayAgo = (Get-Date).AddDays(-1) # 1日前の日付と時刻を取得

Get-WinEvent -LogName Security -FilterHashtable @{
    LogName='Security';
    Id=4624,4625;
    StartTime=$oneDayAgo # 1日前の時刻以降のイベントを取得
} | Format-List -Property TimeCreated, Id, Message | Out-File -Encoding UTF8 -FilePath $path
```

**変更点と解説:**

1.  **`$oneDayAgo = (Get-Date).AddDays(-1)`**:
    * `Get-Date`は現在のシステム日付と時刻を取得します。
    * `.AddDays(-1)`メソッドは、取得した日付から1日前の日付を計算します。この結果が変数`$oneDayAgo`に格納されます。

2.  **`-FilterHashtable @{...; StartTime=$oneDayAgo}`**:
    * 以前の説明で提案したように、`Where-Object`で後からフィルタリングするのではなく、`Get-WinEvent`の`-FilterHashtable`パラメータを使用することで、イベントログの取得自体を効率化しています。
    * `StartTime`キーに`$oneDayAgo`変数を指定することで、指定された時刻（現在時刻の1日前）以降に作成されたイベントのみを取得するようになります。これにより、ログの読み込み量が大幅に削減され、コマンドの実行が速くなります。

この修正により、セキュリティログの中から過去1日間のログオン成功（ID: 4624）とログオン失敗（ID: 4625）のイベントのみを抽出し、指定したファイルに書き出すことができます。

</details>

<details><summary>あなたの提示したコマンドレットを実行すると以下のエラーになりました。</summary>

#### Q.
あなたの提示したコマンドレットを実行すると以下のエラーになりました。

PS C:\WINDOWS\system32> $path = "$env:USERPROFILE\Documents\eventlog.txt"

PS C:\WINDOWS\system32> $oneDayAgo = (Get-Date).AddDays(-1) # 1日前の日付と時刻を取得

PS C:\WINDOWS\system32>

PS C:\WINDOWS\system32> Get-WinEvent -LogName Security -FilterHashtable @{

>>     LogName='Security';

>>     Id=4624,4625;

>>     StartTime=$oneDayAgo # 1日前の時刻以降のイベントを取得

>> } | Format-List -Property TimeCreated, Id, Message | Out-File -Encoding UTF8 -FilePath $path

Get-WinEvent : 指定された名前のパラメーターを使用してパラメーター セットを解決できません。

発生場所 行:1 文字:1

+ Get-WinEvent -LogName Security -FilterHashtable @{

+ ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

    + CategoryInfo          : InvalidArgument: (:) [Get-WinEvent]、ParameterBindingException

    + FullyQualifiedErrorId : AmbiguousParameterSet,Microsoft.PowerShell.Commands.GetWinEventCommand

#### A.
提供されたエラーメッセージ「Get-WinEvent : 指定された名前のパラメーターを使用してパラメーター セットを解決できません。」は、`Get-WinEvent`コマンドレットで`-LogName`と`-FilterHashtable`を**同時に使用している**場合に発生する一般的なエラーです。

これは、`Get-WinEvent`には複数のパラメータセット（使用できる引数の組み合わせ）があり、`-LogName`と`-FilterHashtable`の両方を使うと、PowerShellがどちらのパラメータセットを使用すべきか判断できなくなるために起こります。

**解決策:**

`LogName`を`-FilterHashtable`の中に含めることで、この問題を解決できます。以下のようにコマンドを修正してください。

```powershell
$path = "$env:USERPROFILE\Documents\eventlog.txt"
$oneDayAgo = (Get-Date).AddDays(-1) # 1日前の日付と時刻を取得

Get-WinEvent -FilterHashtable @{
    LogName='Security'; # ここにLogNameを含める
    Id=4624,4625;
    StartTime=$oneDayAgo # 1日前の時刻以降のイベントを取得
} | Format-List -Property TimeCreated, Id, Message | Out-File -Encoding UTF8 -FilePath $path
```

**変更点:**

* `Get-WinEvent -LogName Security` の部分から`-LogName Security`を削除しました。
* 代わりに、`-FilterHashtable`の中に`LogName='Security'`を移動させました。

これにより、`Get-WinEvent`は単一のパラメータセット（`-FilterHashtable`を使用するセット）で処理を実行できるようになり、エラーが解消されるはずです。

</details>

<details><summary>eventlog.txtが出力できました。しかし777kBもサイズがあります。これを構造化して見やすくする方法はありますか？</summary>

#### Q.
eventlog.txtが出力できました。しかし777kBもサイズがあります。これを構造化して見やすくする方法はありますか？

#### A.
777KBの`eventlog.txt`が出力できたとのこと、おめでとうございます。しかし、そのサイズのテキストファイルを目視で解析するのは大変ですね。ログデータを構造化して見やすくする方法はいくつかあります。

以下に主な方法と、それぞれのメリット・デメリットを挙げます。

**1. PowerShellでCSV形式に変換する**

最も簡単で、スプレッドシートソフトウェア（Excelなど）で開いてフィルタリングやソートができるようになるため、非常に有効です。

* **コマンドレットの変更例:**
    ```powershell
    $path = "$env:USERPROFILE\Documents\eventlog.csv" # 拡張子を.csvに変更
    $oneDayAgo = (Get-Date).AddDays(-1)

    Get-WinEvent -FilterHashtable @{
        LogName='Security';
        Id=4624,4625;
        StartTime=$oneDayAgo
    } | Select-Object TimeCreated, Id, Message | Export-Csv -Encoding UTF8 -NoTypeInformation -Path $path
    ```
* **メリット:**
    * Excelなどのスプレッドシートで簡単に開ける。
    * 列ごとにデータをフィルタリング、ソートできる。
    * グラフ化や集計が比較的容易。
* **デメリット:**
    * `Message`プロパティは長い文字列なので、CSVの一つのセルに収まり、見出しがずれていないか確認が必要。
    * メッセージ内の改行などがCSVの区切り文字と誤認識される可能性もゼロではない（`Export-Csv`が適切に処理するはずですが）。

**2. PowerShellでJSON形式に変換する**

プログラミング言語での解析や、Webアプリケーションでの利用に適しています。

* **コマンドレットの変更例:**
    ```powershell
    $path = "$env:USERPROFILE\Documents\eventlog.json" # 拡張子を.jsonに変更
    $oneDayAgo = (Get-Date).AddDays(-1)

    Get-WinEvent -FilterHashtable @{
        LogName='Security';
        Id=4624,4625;
        StartTime=$oneDayAgo
    } | Select-Object TimeCreated, Id, Message | ConvertTo-Json -Depth 5 | Out-File -Encoding UTF8 -FilePath $path
    ```
* **メリット:**
    * 機械可読性が高く、プログラミング言語（Pythonなど）でのデータ解析が容易。
    * ネストされたデータ構造も保持できる。
* **デメリット:**
    * 手動で開いて見ても、生のJSONは人間には読みにくい。
    * JSONビューアや整形ツールが必要になる。

**3. PowerShellでカスタム形式に整形する（読みやすさ重視）**

`Format-List`の代わりに、もっと整形されたテキスト形式で出力します。

* **コマンドレットの変更例 (カスタム整形):**
    ```powershell
    $path = "$env:USERPROFILE\Documents\eventlog_formatted.txt"
    $oneDayAgo = (Get-Date).AddDays(-1)

    Get-WinEvent -FilterHashtable @{
        LogName='Security';
        Id=4624,4625;
        StartTime=$oneDayAgo
    } | ForEach-Object {
        "--- イベント開始 ---"
        "発生時刻: $($_.TimeCreated)"
        "イベントID: $($_.Id)"
        "メッセージ:"
        $_.Message
        "--- イベント終了 --`n" # 各イベントの後に改行を追加
    } | Out-File -Encoding UTF8 -FilePath $path
    ```
* **メリット:**
    * テキストエディタで開いたときに、各イベントの開始と終了が明確になり、項目ごとに見やすくなる。
    * 特定の情報だけを抽出しやすい（正規表現などを使って検索する場合）。
* **デメリット:**
    * CSVやJSONほど構造化されていないため、スプレッドシートでの集計は難しい。
    * 依然として大量のテキストになりやすい。

**4. ログ解析ツールを利用する**

長期的、専門的にログを扱う場合は、専用のツールを検討する価値があります。

* **Microsoft Log Parser Studio / Log Parser:**
    * Microsoft製の強力なツールで、SQLライクなクエリでログファイルを解析できます。
    * 学習コストはかかりますが、複雑な条件での抽出や集計が可能です。
* **SIEM (Security Information and Event Management) ツール:**
    * Splunk, Elastic Stack (ELK Stack - Elasticsearch, Logstash, Kibana), Graylogなど。
    * リアルタイムでのログ収集、解析、可視化、アラート機能などを備えています。大規模なシステムやセキュリティ監視に利用されます。
    * 導入と運用には高いコストと専門知識が必要です。
* **テキストエディタの機能:**
    * Notepad++やVS Codeのような高機能なテキストエディタには、正規表現検索、行のソート、特定の文字列のハイライト表示などの機能があり、ある程度の解析に役立ちます。

**777KBのファイルサイズについて:**

ログオン・ログオフイベントが過去1日分で777KBというのは、一般的なアクティブなシステムであれば妥当なサイズです。CSV形式に変換すれば、Excelで十分に開いて解析できるはずです。

まずはCSV形式に変換する試みから始めて、ご自身の目的や使い勝手に合わせて他の方法も検討してみてください。

</details>

<details><summary>このファイルを構造的に読むためのJSONビューアはありますか？</summary>

#### Q.
このファイルを構造的に読むためのJSONビューアはありますか？

#### A.
はい、VSCodeでこのような `eventlog.json` ファイルを**構造的（ツリー形式）に表示・閲覧するための JSON ビューア**拡張機能はいくつか存在します。以下におすすめの方法を紹介します。

---

## ✅ 方法1：VSCode 標準の JSON 表示機能

VSCode は `.json` ファイルを開くと、**標準で折りたたみ・インデント付きのツリー表示**を提供します：

* ファイルを開くと自動で JSON として認識され、整形表示される
* `{` や `[` の横に「折りたたみトグル」が表示される

### 🔍 整形表示にならない場合：

* コマンドパレット（`Ctrl+Shift+P`）で `Format Document` を実行（または右クリック → 「ドキュメントのフォーマット」）
* 拡張子が `.txt` の場合は `.json` に変更する

---

## ✅ 方法2：拡張機能「JSON Viewer」を使う

### 📦 おすすめ拡張：**JSON Viewer**

#### インストール方法：

1. VSCode 左側の拡張機能アイコンをクリック
2. 検索バーに `JSON Viewer` と入力
3. 提供元 `tulios` や `hbenl` などの人気拡張を選択し「インストール」

#### 特徴：

* JSON をブラウザのような「ツリー形式ビュー」で表示
* 値のコピーや検索がGUIで可能
* 大規模JSONでもパフォーマンスが良好

---

## ✅ 方法3：外部Webツール（VSCode以外）

VSCodeで開かずに、以下のようなオンラインツールに `eventlog.json` をドラッグ＆ドロップすることもできます：

* [https://jsonviewer.stack.hu/](https://jsonviewer.stack.hu/)
* [https://jsonformatter.org/json-viewer](https://jsonformatter.org/json-viewer)

---

## ✅ 補足：VSCodeで整形して表示するショートカット

| 操作           | キー                                                   |
| ------------ | ---------------------------------------------------- |
| フォーマット整形     | `Shift + Alt + F`                                    |
| JSON スキーマの確認 | `Ctrl + Shift + P` → `Change Language Mode` → `JSON` |

---

## ✅ まとめ

| 方法               | 特徴                | 備考                |
| ---------------- | ----------------- | ----------------- |
| 標準表示機能（VSCode）   | 自動整形 + 折りたたみ可能    | 拡張不要              |
| JSON Viewer 拡張機能 | GUIツリー表示、コピー・検索可  | ファイルが大きくても扱いやすい   |
| オンラインツール         | ブラウザで開ける直感的なツリー表示 | インストール不要、セキュリティ注意 |

---

VSCodeでの作業を前提にするなら、**拡張機能「JSON Viewer」＋整形表示（`Shift+Alt+F`）の併用**が非常に快適です。

必要であれば、`eventlog.json` の中身を抽出・フィルタ・変換する Python スクリプトの作成もお手伝い可能です。お気軽にどうぞ。


</details>

<details><summary>aaa</summary>

#### Q.


#### A.


</details>

<details><summary>aaa</summary>

#### Q.


#### A.


</details>

<details><summary>aaa</summary>

#### Q.


#### A.


</details>

<details><summary>aaa</summary>

#### Q.


#### A.


</details>

<details><summary>aaa</summary>

#### Q.


#### A.


</details>

<details><summary>aaa</summary>

#### Q.


#### A.


</details>

<details><summary>aaa</summary>

#### Q.


#### A.


</details>

<details><summary>aaa</summary>

#### Q.


#### A.


</details>

<details><summary>aaa</summary>

#### Q.


#### A.


</details>

<details><summary>aaa</summary>

#### Q.


#### A.


</details>

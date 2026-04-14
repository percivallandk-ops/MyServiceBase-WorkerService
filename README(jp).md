# MyServiceBase for .NET 10 WorkerService

## ― Windows Service をより扱いやすくする拡張ベースクラス ―

検索キーワード：.NET10.0 WorkerService,Windows Service,PreShutdown,ServiceBase

## はじめに

本ライブラリは、.NET 10.0 WorkerService で動作する Windows Service 向けに、**.NET Framework4.8 ServiceBase 相当の機能 + PreShutdown 対応機能** を提供する拡張ベースクラス **MyServiceBase** を公開するものです。

本ライブラリは、WorkerService の弱点である**PreShutdownイベントの受信**及び対応を容易に行えるインターフェースを提供し、「開始」「停止」「非同期タスクの終了管理」プロセスの定型処理部分とユーザー処理部分を完全に切り離し **「安全性」「開発効率」「メンテナンス性」** を**大幅に改善**するものです。

---

## 開発の背景

私はWorkerService (.NET 10) で PreShutdown を扱う Windows Service を作成しようとしました。

しかし、.NET10でWindows Serviceを作成するための*Microsoft.Extensions.Hosting.WindowsServices* を導入しただけでは **PreShutdown を正しく受信できない** ことが分かりました。

実測調査の結果、.NET 10 の ServiceBase はHosting内部専用であり、SCM（Service Control Manager）との連携をユーザー側で制御する手段がない、という結論に至りました。

やむなくWin32 API を直接利用して SCM と通信することにしたのですが、アプリケーションを構築するにあたり、この部分だけはブジェクト指向から外れるため、不便、処理漏れ（特に後始末）の危険を感じました。そこで、WorkerService のモデルを維持したまま **Start / Stop / Shutdown / PreShutdown** を容易に扱える独自の **MyServiceBase** を開発しました。

同じ問題で困っている方の助けになればと思い、公開します。

---

## キーフィーチャー

### ✔ SCM イベントに対応したオーバーライド関数

- OnStart() …… サービス開始時
- OnStop() …… サービス停止時
- OnShutdown() …… シャットダウン時
- OnPreShutdown() …… プレシャットダウン時
  ※ Windows の仕様により、Shutdown と PreShutdown はどちらか一方のみ選択可能

---

## 親切機能（実用サービス向け）

WorkerService で実際にサービスを作ると、  「開始後に非同期ループを動かして待機、停止時に安全に終了させる」 という処理が必須になります。
MyServiceBase はこの部分も大幅に簡略化します。

- SCMからの停止要求に対し非同期タスクへ CancellationToken をリレー
- 各タスクはCancellationToken を受信したら終了する。（ユーザーがやるのはこれだけ）
- MyServiceBaseに対し、タスク終了を確認してサービス停止に入る設定が可能（設定は任意です）
- Shutdown / PreShutdown 時も同様に上記トークンを発行
- ログはイベントログではなく ユーザーデータ領域にテキスト出力
  （メモ帳で確認可能。必要なら出力部分はイベントログへの書き出しなどへ差し替え可能）
  これにより、**より確実性が高いサービス開発**が可能になります。

---

## 動作環境（検証環境）

- Windows 11 Pro 24H2
- Visual Studio 2026
- C# (.NET 10.0)
- サービス登録・運用には管理者権限が必要です

---

## 添付データ(ZIP)の説明

いずれも、実際使用した／使用している実働サンプルです。

-  MyServiceBase.zip

        今回公開するMyServiceBaseクラスのC#ソースです。単一クラス単一コードであるため、クラスライブラリではなくソース部品として提供します。

- Sample1-NetTestWorkerService.zip
  
  - MyServiceBase.csを含んだビルド可能なソースコード一式です。ビルドする場合はC#ファイルをすべてプロジェクトに含めてください。
  
  - このサービスはPreShutdown中、いつまで外部ネットと通信できるかテストするために作成したものです。
  
  - 他のプロジェクトで使用する際のテンプレートとして利用できます。MyServiceBase.cs、Program.cs、Helper.csはコピーしてそのままお使いいただけます。MyWorker.csはテンプレートとして使用可能です。
  
  - パワーシェルスクリプトは、ビルドしたexeをサービスに登録・起動するscコマンドのスクリプトのサンプルです。適宜環境に合わせ変更してください。管理者権限が必要です。

- Sample2-Watcher1.zip
  
  - Workerサンプルです。sample1のMyServiceBase.cs、Program.cs、Helper.csをプロジェクトに含めることでビルド可能になります。Worker以外は全く同じものが使用できます。MyServiceBaseの独立性の高さが分かります。
  
  - 他のサービスのSCMステータスを監視し、ステータスがどのように変化するかをログに残すサービスです。サービスのデバッグ、特にシャットダウンギリギリのタイミングで動作させる場合に役立つと思われます。実際にお使いになる際は、TARGET_SERVCEの監視対象サービスを適切な名称にしてビルドしてください。
  
  - 本サンプルは、終了待ちループRunAsyncに直接コードを書いた例です。

- Sample3-SerialCom.zip
  
  - Workerサンプルです。sample1のMyServiceBase.cs、Program.cs、Helper.csをプロジェクトに含めることでビルド可能になります。（通信相手がいないと動作はしません。）
  
  - USBの仮想シリアルポートに接続したマイコンと通信します。
  
  - 本サンプルは、OnStartで、非同期タスクを複数起動して並行稼働させ、自身は素早くリターンするモデルのサンプルです。

## 詳細ドキュメント

以下の内容は docs/ に詳細をまとめています：

- MyServiceBase / MyWorker の使い方 ([使い方.md](DOCS/JP/使い方.md)）
- 技術資料　([テクニカルノート.md](DOCS/JP/テクニカルノート.md))
  - .NET Core と .NET Framework の違い
  - PreShutdown の挙動
  - 実験結果
  - 実装方法
- ソースコード内の注釈を参照（より実務的なコーディングアドバイス）

---

## 署名

- 著者: K.IYO
- 本ライブラリおよび記事は、著者が設計し、ChatGPT（OpenAI）および Microsoft Copilot の支援を受けて作成しました。

---

## コピーライト、保証等

- 本ソフト及び記事に関するコピーライトは放棄しません。
- 本ソフトの利用、改変については、個人・法人を問わず自由にできるものとします。
- 本ソフト及びドキュメントについて、著作者の許可なく別媒体や別サイトへの転載や販売を行うことを禁止します。
- 本ソフトについて一切の動作保証をいたしません。また、本ソフトの利用により生じた一切の損害について責任を負いません。自己責任でご利用ください。
- Copyright © 2026.4 K.IYO. All rights reserved.

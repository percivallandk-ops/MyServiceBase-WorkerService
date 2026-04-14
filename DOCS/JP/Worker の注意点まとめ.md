//==============================================
// MyServiceBase / Worker 利用時の重要ポイント
//==============================================

// ■ ログ出力
// Helper.SetDebugMSGLevel(name, level):
//   ログレベル設定（0=例外, 1=基本, 2=詳細, -1=無効）
// Helper.Write(message, level):
//   指定レベル以下のログを "C:\ProgramData\MyService\MyService.log" に出力
//   保存先は変更可能（イベントログ出力に差し替えも可）

// ■ タスク監視（任意）
// registerTask(name, task) / registerTasks(...):
//   停止要求時、登録タスクの終了を最大8秒待つ
//   終了しないタスク名はログに記録（ゾンビ化の可能性あり）

// ■ CancellationToken の扱い（重要）
//   停止要求で token.IsCancellationRequested が true になる
//   各タスクはこれを監視して速やかに終了すること
//   OnStart / OnPreShutdown などで token をサブタスクへリレーする

// ■ OnStart の注意
//   OnStart は即時リターン必須（ブロック禁止）
//   実処理は非同期タスクで行うこと
//   遅延すると SCM が「起動失敗」と判断する

// ■ SERVICE_NAME（必須）
//   サービス名。重複不可。SCM・タスクマネージャ・イベントログに表示される

// ■ IsPreShutdown / PreShutdownTimeout
//   IsPreShutdown=true で PreShutdown を受信
//   false の場合は Shutdown を受信（排他仕様）
//   PreShutdownTimeout は待機時間（ms）。上限はシステム依存

// ■ OnXXXX() のオーバーライド
//   必要なものだけ override すればよい（未実装なら空動作）
//   RunAsync() は Worker 側で override 推奨
//   何も処理しない場合は削除してもよい（Shutdown/PreShutdown 専用サービスなど）

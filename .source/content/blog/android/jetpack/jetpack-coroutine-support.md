+++
date = "Sun Mar 10 10:14:21 UTC 2019"
title = "Android: JetpackのCoroutine Supportについて"
tags = ["android", "jetpack", "coroutine", "ktx", "kotlin"]
blogimport = true
type = "post"
draft = false
+++

Jetpackのいくつかのライブラリでは、Kotlin Coroutineのサポートが入っていますが、
どのライブラリで対応が進んでいるか気になったので、軽くまとめます。使い方については言及しません。

以下、2019年3月10日の調査結果になります。
また、これらは、supportライブラリのリポジトリから取ってきたので、現在リリースされているかどうかは不明です。

Lifecycle

```kotlin
val Lifecycle.coroutineScope: CoroutineScope
```

LifecycleOwner

```kotlin
val LifecycleOwner.lifecycleScope: CoroutineScope
```

ViewModel

```kotlin
val ViewModel.viewModelScope: CoroutineScope
```

WorkManager
```kotlin
abstract class CoroutineWorker(
  appContext: Context,
  params: WorkerParameters
) : ListenableWorker(appContext, params) {
  abstract suspend fun doWork(): Result
}
```

Room
```kotlin
// Daoでsuspendメソッドが定義出来る
@Dao
interface HogesDao {
  @Insert
  suspend fun add(hoge: Hoge)

  @Query("SELECT * FROM hoge WHERE id = :id")
  suspend fun get(id: String): Hoge

  ...
}
```

## まとめ

- Lifecycle、LifecycleOwner、ViewModelはそれらのライフサイクルに従う、CoroutineScopeの生成が出来る
- WorkManagerのdoWorkがsuspendメソッドになった
  - doWorkの中で、他のsuspendメソッドがコールできる
- RoomなのDao内でsuspendメソッドが定義出来る

結構対応がされていた😃😃😃

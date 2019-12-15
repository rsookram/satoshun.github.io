+++
date = "Sun Dec 15 05:03:08 UTC 2019"
title = "ViewModelとKotlin Coroutinesの書き方あれこれ"
tags = ["android", "kotlin", "viewmodel", "coroutine"]
blogimport = true
type = "post"
draft = true
+++

ViewModel + Kotlin Coroutineを使う場合、どんな感じでViewModelでCoroutineを実行するかについて考えてみました。
MVVM + Repositoryを想定しており、UIに反映する部分はLiveDataを使っています。

`androidx.lifecycle:lifecycle-viewmodel-ktx`は2.2.0-rc03、Coroutineは1.3.3です。

この記事は次の順序で進んでいきます。

- viewModelScopeとは?
- suspend関数をコールする場合
- FlowをViewModelでコール/購読する場合

## ViewModel.viewModelScopeとは?

`androidx.lifecycle:lifecycle-viewmodel-ktx`ライブラリで、viewModelScope拡張関数が使うことが出来ます。

```kotlin
/**
 * [CoroutineScope] tied to this [ViewModel].
 * This scope will be canceled when ViewModel will be cleared, i.e [ViewModel.onCleared] is called
 */
val ViewModel.viewModelScope: CoroutineScope
```

ViewModelのライフサイクルに合わせたCoroutineScopeを取得することが出来ます。
ViewModelでCoroutineを扱うときは、このスコープで実行しておけば、自動的にdisposeしてくれるので、メモリリークの心配もないです。

また、`viewModelScope`は、メインスレッド上で実行してくれるため、`LiveData.setValue`を使います。

```kotlin
val userLiveData = MutableLiveData(...)

viewModelScope.launch {
    val user = userRepository.getUser() // 適当なsuspend関数をコール

    userLiveData.setValue(user) // メインスレッド上で実行されることが保証されているのでsetValueを使う
    // userLiveData.postValues(user)
}
```

viewModelScopeを使っている限り、postValueメソッドを使うケースは無いと思います。


## suspend関数をコールする場合

ネットワークコールなどのAPIは、基本的にワンショットAPIなので、suspendで定義することになると思います。
Retrofitでは2.6.0から、suspendが使えるようになりました。

なので、Repository層での定義は次のようになると思います。

```kotlin
interface UserApi {
    suspend fun getUser(): User
}

class UserRepository(private val retrofitService: UserApi) {
    suspend fun getUser() : User {
        val user = retrofitService.getUser()
        ...
        return user
    }
}
```

これをViewModelからコールします。

```kotlin
viewModelScope.launch {
    val user = userRepository.getUser()
    ...
}
```

これだけだと、ネットワークの調子が悪い時などに、例外が起きてしまうので、エラーハンドリングをする必要があります。

ViewModel側でハンドリングするなら、try-catch、もしくは`runCatching`を使うのが良いと思います。

```kotlin
viewModelScope.launch {
    try {
        val user = userRepository.getUser()
        ...
    } catch (e: Exception) {
        ...
    }
}
```

```kotlin
viewModelScope.launch {
    runCatching { userRepository.getUser() }
        .onSuccess { ... }
        .onFailure { ... }
}
```

個人的には`runCatching`のほうが好きです😃

また、Repository側で適当な型で包むパターンもあると思います。例えばNetWorkResultのようなクラスがあるとします。

```kotlin
sealed class NetWorkResult<T> {
    class Success<T>(val value: T) : NetWorkResult<T>()
    class Error(val exception: Exception) : NetWorkResult<Nothing>()
    ...
}
```

このクラスをRepository側で使います。

```kotlin
class UserRepository(private val retrofitService: UserApi) {
    suspend fun getUser() : NetWorkResult<User> {
        try {
            val user = retrofitService.getUser()
            return NetWorkResult.Success(user)
        } catch (e: Exception) {
            return NetWorkResult.Error(e)
        }
    }
}
```

こうすることで、ViewModel側では、try-catchを使わなくて良くなり、代わりにwhen式を使うことになります。

ここまでがsuspend関数の説明になります。次にFlowを返すAPIの話です。


## Flow APIの話

Flow APIは、複数の値を流すストリームを表現することが出来ます。RxJavaで言うところのObservableとか、Flowableのようなものと考えられます。

Flowを購読するタイミングは、ViewModelのinitブロックが良いと思います。重複登録の心配がないためです。`SavedStateHandle`を組み合わせることで、多くの場合、initブロックで初期化を行うことが出来ると思います。

```kotlin
class MyViewModel(private val state: SavedStateHandle) : ViewModel() {
    init {
        ...
    }
}
```

Flowの購読方法なんですが、`collect(collectLatest)`もしくは、`launchIn`を使います。

```kotlin
class MyViewModel(...) : ViewModel() {
    init {
        viewModelScope.launch {
            repository.getFlowStream()
                .collect { ... }
        }

        repository.getFlowStream()
            .onEach { ... }
            .launchIn(viewModelScope)
    }
}
```

エラーや、完了イベントをハンドリング必要がある場合、catch、onCompletionメソッドを使います。

```kotlin
repository.getFlowStream()
    .onEach { ... }
    .catch { ...}
    .onCompletion {  }
    .launchIn(viewModelScope)
```

個人的にはネストが少なくなるので、`launchIn`を使う書き方のほうが好みです。


## まとめ

- viewModelScopeメソッドを使うと自動でリソースを解放してくれる
- runCatchingを使うと、いい感じにエラーハンドリングが出来る
- launchInを使うと、いい感じにFlowを購読できる

Coroutineはいいぞ〜

+++
date = "Sun Oct 13 05:23:44 UTC 2019"
title = "ViewPager2で要素をループさせる"
tags = ["android", "viewpager2", "jetpack"]
blogimport = true
type = "post"
draft = false
+++

ViewPager2で要素をループさせる方法の紹介です。いわゆる循環リストです。

ViewPager2ではRecyclerViewを使うことが出来るので、RecyclerViewと同じような実装で実現することが出来ます。

最終的にこんなのが作れます。（a、b、cの要素でループしている）

<img src="/blog/android/jetpack/viewpager2/viewpager2-loop.gif" style="max-width:280px" />

今回の検証に用いたコードは、[satoshun/ViewPager2](https://github.com/satoshun-android-example/ViewPager2/tree/master/app/src/main/java/com/github/satoshun/example/infinite)にあります😃

では、コードを説明していきます。今回は、1つのViewTypeを扱います。

## 実際のコード

まず、`RecyclerView.Adapter`のサイズを決める`getItemCount`メソッドの実装です。
限りなく大きい値、`Int.MAX_VALUE`を返します。

```kotlin
override fun getItemCount(): Int = Int.MAX_VALUE
```

次に、`onBindViewHolder`メソッドを次のように実装します。

```kotlin
private val itemData: List<Data>

override fun onBindViewHolder(holder: InfiniteViewHolder, position: Int) {
  val data = itemData[position % itemData.size]
  ...
}
```

`itemData`にはViewの生成に必要な実際のデータが入っています。`position % itemData.size`とindexを取ることで、ループ中のどこの位置にいるかを特定することが出来ます。

最終的な、RecyclerView.Adapterは次のようになります。（onCreateViewHolderメソッドは重要でないので、省略しています）

```kotlin
class InfiniteAdapter(
  private val itemData: List<Data>
) : RecyclerView.Adapter<InfiniteViewHolder>() {

  override fun getItemCount(): Int = Int.MAX_VALUE

  override fun onCreateViewHolder(parent: ViewGroup, viewType: Int) = ...

  override fun onBindViewHolder(holder: InfiniteViewHolder, position: Int) {
    val data = itemData[position % itemData.size]
    ...
  }
}
```

最後に、ViewPager2のセットアップをします。

```kotlin
with(binding.viewpager) {
  val infiniteAdapter = InfiniteAdapter(itemData = mockAdapterData)
  adapter = infiniteAdapter
  val center = Int.MAX_VALUE / 2
  val start = center - (center % mockAdapterData.size)
  setCurrentItem(start, false)
}

private val mockAdapterData = (0..2).map {
  Data(title = ('a' + it).toString())
}
```

`setCurrentItem`には、中心をセットしてあげます。そうすることで、初期状態でも左にスクロールすることが出来ます。

これで完成です！複数のViewTypeを扱いたいときは、もう少し工夫が必要になると思いますが、基本はこれだけで動かすことが出来ます。

## 注意点

- `Int.MAX_VALUE / 2` 回スクロールすると、先頭にたどり着くので、実は無限スクロールではない
    - とはいえ、普通そんなにスクロールしないし、無視しても問題ないと思います

## まとめ

- ViewPager2はRecyclerViewの知識を使えるので嬉しい😃
- コードは、[satoshun/ViewPager2](https://github.com/satoshun-android-example/ViewPager2/tree/master/app/src/main/java/com/github/satoshun/example/infinite)にあるので、細かいところはこちらを見て下さい。

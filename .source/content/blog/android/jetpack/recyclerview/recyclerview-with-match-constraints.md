+++
date = "Fri May 15 13:25:48 UTC 2020"
title = "Android: ConstraintLayoutの子にRecyclerViewを配置して、Match Constraintsを設定すると良くない挙動をする"
tags = ["android", "recyclerview", "jetpack"]
blogimport = true
type = "post"
draft = false
+++

備忘録です。

次のようなレイアウトは良くないぞという話です。

```xml
<androidx.constraintlayout.widget.ConstraintLayout
  android:layout_width="match_parent"
  android:layout_height="wrap_content">

  <androidx.recyclerview.widget.RecyclerView
    android:id="@+id/recycler"
    android:layout_width="0dp"
    android:layout_height="wrap_content"
    app:layout_constraintEnd_toEndOf="parent"
    app:layout_constraintStart_toStartOf="parent"
    app:layout_constraintTop_toTopOf="parent" />
</androidx.constraintlayout.widget.ConstraintLayout>
```

今回の例では、LinearLayoutManagerをLayoutManagerとして使っています。他のLayoutManagerの場合、どういう挙動をするか分かりません。
また、ConstraintLayout 2.0.0-beta06、RecyclerView 1.1.0で試しています。


## どんな挙動をするか?

このレイアウトはファーストビューのタイミングで、**すべてのアイテム**をバインドしようとします。

例えば500個のRecyclerViewアイテムがあったときに、画面に収まるかどうかに関わらず500個のバインドが走ります。

```kotlin
with(recycler) {
  // 横方向のLinearLayoutManager
  layoutManager = LinearLayoutManager(
    this@ConstraintMatchConstraintsActivity,
    RecyclerView.HORIZONTAL,
    false
  )

  // 500個のアイテムを生成
  adapter = SampleAdapter().apply {
    submitList((0..500).map { "$index $it" })
  }
}

private class SampleAdapter : ListAdapter<String, RecyclerView.ViewHolder>(...) {
  ...
  override fun onBindViewHolder(holder: RecyclerView.ViewHolder, position: Int) {
    println(getItem(position)) // ここが、0 ~ 500まで表示される
  }
}
```

500個全部に対して、最初にバインドが走るのは非効率なので良くないです😂

## なんでか?

LinearLayoutManagerでは、内部で`mInfinite` っていうフィールドを持っていて、これは次の関数によって生成されます。

```java
boolean resolveIsInfinite() {
  return mOrientationHelper.getMode() == View.MeasureSpec.UNSPECIFIED
      && mOrientationHelper.getEnd() == 0;
  }
```

細かいところまで追えてないのですが、今回のレイアウトの場合、上記の関数がtrueを返していました。

この値がtrueだと、どういうことが起こるかっていうと、RecyclerViewにアイテムを詰めるタイミングで一気に全部アイテムを詰めようと試みます。
具体的には、次のコードになります。

```java
int fill(RecyclerView.Recycler recycler, LayoutState layoutState,
    RecyclerView.State state, boolean stopOnFocusable) {
  ...
  // mInfintieがtrueなので、このwhileが最後まで行くことが可能になる
  while ((layoutState.mInfinite || remainingSpace > 0) && layoutState.hasMore(state)) {
    ...
  }
}
```

前述の`mInfinite`がtrueだと、whileが`remainingSpace`の値に関係なく、最後まで行くことが可能になります。
結果、今回の場合では全アイテムをファーストビューのタイミングでバインドします。

## どう直すか?

直し方としては

1. ConstraintLayoutをRecyclerViewの親にしない
2. wrap_contentを使う

```xml
<androidx.constraintlayout.widget.ConstraintLayout
  android:layout_width="match_parent"
  android:layout_height="wrap_content">

  <androidx.recyclerview.widget.RecyclerView
    android:id="@+id/recycler"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    app:layout_constraintStart_toStartOf="parent"
    app:layout_constraintTop_toTopOf="parent" />
</androidx.constraintlayout.widget.ConstraintLayout>
```

3. match_parentを使う

```xml
<androidx.constraintlayout.widget.ConstraintLayout
  android:layout_width="match_parent"
  android:layout_height="wrap_content">

  <androidx.recyclerview.widget.RecyclerView
    android:id="@+id/recycler"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    app:layout_constraintTop_toTopOf="parent" />
</androidx.constraintlayout.widget.ConstraintLayout>
```

で良いかなと思います😃

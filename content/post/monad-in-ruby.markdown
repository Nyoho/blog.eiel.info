---
title: "Ruby で モナドを書いてみた。"
date: 2013-04-04T00:41:00+09:00
comments: true
tags: ["ruby","monad"]
---

ちょっと気分転換したかっただけで、反省している。(2015年4月11加筆修正)

記事を書く目的があったわけでも、何か確認したかったわけでもないけど、自分的に得るものがあったので、それを書いておきます。

Rubyでモナドをつくってみました。

* [ソースコード](https://gist.github.com/eiel/5302011)

モナドってモノイドに名前が似ていることからわかるようにモノイド的な特性があるらしいです。

* [Wikipedia:モノイド](http://ja.wikipedia.org/wiki/%E3%83%A2%E3%83%8E%E3%82%A4%E3%83%89)

今回の話は、モナドだと簡単にモノイドが作れるという話のような気がする。

### 結合律

モノイドであれば**結合律**が成立します。

結合律をプログラミングに当てはめてみると

```
func1();
func2();
func3();
```

という命令列があった場合

```
func1(); func2();
func3();
```

と

```
func1();
func2(); func3();
```

は等価と言えるという話にできます。

違いがよくわからないので、別名をつけてまとめてみます。

```
funcX();   // funcX () { func1(); func2(); }
func3();
```

```
func1();
funcY();   // funcY () { func2(); func3(); }
```

どちらも同じように動きますよね。
セミコロンを演算子とみたててみます

```
func1 ; func2 ; func3
```

最後のセミコロンはみにくいので削除しました。

```
(func1 ; func2) ; func3 == func1 ; (func2; func3)
```

### 単位元

* 単位元の存在  - 演算してもコンテキストが変化しない値が存在する

プログラムでいえばセミコロンだけでかこまれていればそんな感じになりそうです。

```
func1;
;         // あってもなくても変わらない
func2;
```

### かけ算で考える

結合律と単位元を整数のかけ算に確認しておきます。

```ruby
3 * 4 * 5 == (3 * 4) * 5 == 3 * (4 * 5) == 60 // どっちを先に計算してもOK

3 * 1 == 3    // 単位元になにをかけ算してもそのまま。
1 * 3 == 3
```

簡単ですね。

### 具体例

さて、作成した Maybeクラスですが `*` を用意しています。遊んでみましょう。
Maybeのインスタンス同士でしか演算はできません。

```ruby
Maybe.new(3)                # => Just 3
Maybe.new(2)                # => Just 2
Maybe.new(3) * Maybe.new(2) # => Just 2

Maybe.zero    # => Nothing
Maybe.new(3) * Maybe.zero   # => Nothing
Maybe.zero  * Maybe.new(2)  # => Nothing
Maybe.zero  * Maybe.zero  # => Nothing
Maybe.new(3) * Mabye.zero * Mabye.new(2) # => Nothing
```

2 とか 3 とかは気にせずに `Just` と `Nothing`だけに注目しましょう。
あえて無視するようにつくっています。

* Just    * Just    = Just
* Just    * Nothing = Nothing
* Nothing * Just    = Nothing
* Nothing * Nothing = Nothing

というルールが成立しています。

整数で考えたいのであれば Just を 1、 Nothing を 0 と考えても良いかもしれません。
Bool であれば Just は true, Nothing は false で、 * は `and` という置き換えができそうですね。

この`*`ですか実装はとてもシンプルです。

```ruby
def *(m)
  bind { m }
end
```

この実装はHaskellの`>>`と同じです。
bindを使えば似たようなものをたくさんつくれます。

なので、モナドであれば簡単にモノイドをつくれるということが言えそうです。
単位元はreturnすれば作れるし、演算子はbindをつかって簡単に作れます。

以下補足。

Maybe.zero は Nothing を生成するためのファクトリです。

Maybe は `Just` と `Nothing` しかないので `*`という演算に対する単位元と zero しか値がないのであんまり楽しくないですね。
あとで、Haskell で リストモナドをみてみましょう。

### >>

`>>` はいままで保持している値を捨ててしまうので、後のほうの値である 2 しか残っていません。
この `>>` を別のものに代えてしまえばモナドが保持している値も持ち運びができます。

`Maybe.bind` ですが、
`a -> M b` な型の関数を用意すれば単項演算子が作れる感じです。
(>>= return) の型を見てみるとわかりやいです。

```
:t (>>= return)
(>>= return) :: Monad m => m b -> m b
```

Maybe.new は Haskell では `Just` になります。
抽象化してどんなモナドでも使えるのが `return` になります。

Maybe であれば `+` も定義できます。MonadPlus です。

```ruby
Maybe.new(3) + Maybe.new(2) # => Just 2
Maybe.new(3) * Maybe.zero + Maybe.new(2) # => Just 2
Maybe.zero + Maybe.new(3) * Maybe.new(2) # => Just 2
Maybe.zero + Maybe.zeror # => Nothing
```

* Just    + Just    = Just
* Nothing + Just    = Just
* Just    + Nothing = Just
* Nothing + Nothing = Nothing

+ においては 単位元は Nothing になります。


## List モナド

これらのことを踏まえて Haskell でリストモナドで遊んでみます。

```haskell
[1] >> [2] -- => [2]
[1] >> []  -- => []
[] >> [1]  -- => []
[] >> []   -- => []
```

`>>` を `*` だと考えましょう。

要素が1つの リストが単位元です。

もうちょっと複雑な例にいきましょう。


```haskell
[1,2] >> [3]   -- => [3,3]
[1,2] >> [3,4] -- => [3,4,3,4]
[1,2] >> []    -- => []
[]    >> [3,4] -- => []
```

ちょっとわかりにくい。
以下の作業をしてみましょう。

* `>>` を `*` に
* 演算される値を要素数に置き換えてみましょう。

```
2 * 1 => 2  --  [1,2] >> [3]   # => [3,3]
2 * 2 => 4  --  [1,2] >> [3,4] # => [3,4,3,4]
2 * 0 => 0  --  [1,2] >> []    # => []
0 * 2 => 0  --  []    >> [3,4] # => []
```

整数の積になりました。

値が捨てられるのが気に食わないですか？
活かせるようにもできます。

```haskell
let product = \x y -> x >>= \x' -> (y >>= \y' -> return $ x' * y') :: [Int]
[1,2] `product` [3] -- => [3,6]
-- 2 * 1 = 2
[1,2] `product` [3,4] -- => [3,4,6,8]
-- 2 * 2 = 4
```

ちょっと難しいかもしれません、product は do 記法つかえばシンプルに書けます。

```haskell
product x y = do
  x' <- x
  y' <- y
  return $ x' * y'
```

## IO モナド

なんとなく思ったこと。

結合律を利用して順番を保証しているのだと思いました。

## Rubyの話もすこし

型推論ができないため、 bind がブロックに渡す引数を ふたつにして、第2引数に型を渡してみました。
これでbind に渡すブロック内の処理が Maybe に依存せずに済んでいます。

## まとめ。

Rubyでモナドを実装してみたら、
「モナドを勉強していて気になっていたことが文章にできそうになった」ので、文章にしてみました。

途中に説明不足な部分がありますが、今回は割愛しておきます。

圏論楽しいですね。やばいです。

誰かのモナドの理解への助けになるといいな。


それより斧で血だらけになりそうです。

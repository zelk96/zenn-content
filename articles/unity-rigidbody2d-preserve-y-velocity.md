---
title: "Unityでジャンプ後の落下が遅い原因は、毎フレームY速度を0にしていたことだった"
emoji: "🧱"
type: "tech"
topics: ["unity", "csharp", "ゲーム制作", "unity2d"]
published: true
published_at: 2026-08-10 12:00
---

Unityで初めて2Dアクションの左右移動とジャンプを実装したとき、プレイヤーがジャンプ後になかなか落下しない問題が発生しました。

Gravity Scaleを変更しても期待した動きにならず、最初は重力の設定が原因だと思っていました。しかし、原因は自分で書いた移動処理でした。

## 当時の移動処理

左右入力を受け取り、`Rigidbody2D.linearVelocity`へ速度を設定していました。

問題があったコードを単純化すると、次の形です。

```csharp
rb.linearVelocity = new Vector2(
    horizontalInput * moveSpeed,
    0
);
```

X方向には左右の移動速度を入れ、Y方向には`0`を指定しています。

一見すると「横へ移動させているだけ」に見えます。しかし、この処理を物理更新のたびに実行すると、ジャンプや重力によって変化したY方向の速度まで毎回`0`へ戻してしまいます。

## Vector2はXとYをまとめて上書きする

`linearVelocity`は、2D空間におけるX方向とY方向の速度を持っています。

```text
linearVelocity
├─ x：左右方向の速度
└─ y：上下方向の速度
```

新しい`Vector2`を代入するときは、変更したいXだけでなく、Yも含めた全体を上書きします。

そのため、

```csharp
new Vector2(x, 0)
```

と書くと、「Xだけ変更し、Yはそのまま」ではなく、「Xを変更し、Yは0にする」という意味になります。

## 修正：現在のY速度を引き継ぐ

左右移動で変更したいのはX方向だけです。そこで、Y方向には現在の値をそのまま入れるよう修正しました。

```csharp
rb.linearVelocity = new Vector2(
    horizontalInput * moveSpeed,
    rb.linearVelocity.y
);
```

これなら、

- X方向：入力に応じた移動速度へ更新
- Y方向：ジャンプと重力で変化した現在の速度を維持

という動きになります。

修正後は、ジャンプで上昇したあとに重力で自然に落下するようになりました。

## UpdateとFixedUpdateも分ける

このときは、入力の取得と物理移動も分けました。

```csharp
private void Update()
{
    horizontalInput = Input.GetAxisRaw("Horizontal");
}

private void FixedUpdate()
{
    rb.linearVelocity = new Vector2(
        horizontalInput * moveSpeed,
        rb.linearVelocity.y
    );
}
```

この試作では、

- `Update()`：プレイヤー入力を受け取る
- `FixedUpdate()`：`Rigidbody2D`の速度を変更する

という役割にしました。

## 動作確認

![左右移動とジャンプの試作](/images/unity-rigidbody2d-preserve-y-velocity/day01.gif)

見た目は白い四角と床だけですが、最初の試作だからこそ、速度がどのように変化しているかを確認しやすい状態でした。

## コード以外が原因だったトラブルもあった

同じ日に、「コードを書いたのにプレイヤーがまったく動かない」という問題も起きました。

こちらの原因は、`PlayerController.cs`をPlayerオブジェクトへアタッチしていなかったことでした。

Unityでは、コードだけでなく次の場所も確認する必要があります。

- Hierarchyに対象オブジェクトが存在するか
- InspectorにScriptがアタッチされているか
- 必要なComponentが付いているか
- Consoleにエラーが出ていないか

「コードは合っているはず」と思ったときほど、Unity Editor側の設定も確認するのが大切だと学びました。

## まとめ

今回の原因は、左右移動のために速度全体を上書きし、Y方向の速度まで消していたことでした。

```csharp
// Y速度を消してしまう
rb.linearVelocity = new Vector2(x, 0);

// 現在のY速度を維持する
rb.linearVelocity = new Vector2(x, rb.linearVelocity.y);
```

「変更したい軸だけを更新し、それ以外は現在値を引き継ぐ」という考え方は、その後のプレイヤー操作でも何度も使う基礎になりました。

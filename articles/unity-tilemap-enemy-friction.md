---
title: "UnityのTilemap上で敵が止まる原因はPhysics Material 2Dの摩擦だった"
emoji: "⚙️"
type: "tech"
topics: ["unity", "csharp", "ゲーム制作", "unity2d"]
published: true
---

Unityで2DアクションゲームのステージをTilemapへ移行したとき、巡回する敵が床の途中で止まる問題が発生しました。

壁にぶつかったわけでも、崖へ到達したわけでもありません。平らに見える床の上で、敵だけが急に動かなくなります。

最終的な原因は、Tilemapの形状やRaycastではなく、Collider同士の摩擦でした。

## 当時の構成

ステージ制作では、複数のTile Colliderをまとめるために次の構成を使っていました。

- Tilemap Collider 2D
- Composite Collider 2D
- StaticのRigidbody2D

敵側には`BoxCollider2D`があり、左右へ巡回しながら、壁または崖を検知すると反転する仕組みです。

Tilemap導入後のステージ全体は次のような形でした。

![Stage01のグレーボックス](/images/unity-tilemap-enemy-friction/stage01-graybox.png)

## 最初に疑ったもの

敵が止まったとき、最初は次のような原因を疑いました。

- Composite Collider 2Dの形状に隙間や段差がある
- 崖判定用のRaycastが誤反応している
- 壁判定の距離が長すぎる
- TilemapのCollider生成に問題がある

しかし、判定処理を確認しても、敵が反転条件へ入った形跡はありませんでした。

つまり、「停止処理が実行された」のではなく、「移動しようとしているが物理的に進めなくなっている」可能性が高くなりました。

## 原因はColliderの摩擦

調査の結果、敵の`BoxCollider2D`と床のColliderの間に働く摩擦が原因だと分かりました。

そこで、摩擦をなくした`Physics Material 2D`を作成しました。

```text
Name: NoFriction
Friction: 0
```

このMaterialを敵の`BoxCollider2D`へ設定したところ、Tilemap上でも滑らかに巡回できるようになり、壁・崖での反転も正常に動作しました。

## なぜTilemapの不具合に見えたのか

Tilemapへ移行した直後に発生したため、原因もTilemap固有の設定だと思い込んでいました。

しかし実際には、ステージ側のCollider構成が変わったことで、以前は目立たなかった摩擦の影響が表面化していました。

今回の問題は、次のように切り分けると考えやすくなります。

```text
敵が途中で止まる
├─ 反転している → 壁・崖判定を確認
├─ 速度が0になる → Scriptの代入を確認
└─ 速度を与えても進まない → Collider・摩擦を確認
```

## Frictionを0にすれば常に正解ではない

今回は、敵を一定速度で巡回させることが目的だったため、摩擦をなくす設定が適していました。

一方で、坂道で踏ん張るキャラクターや、床材によって滑り方を変えたいゲームでは、摩擦もゲーム性の一部になります。すべてのColliderへ無条件に`Friction: 0`を設定するのではなく、求める挙動に応じて使い分ける必要があります。

自分のプロジェクトでは、移動するキャラクターがTilemap上で不自然に止まったとき、Physics Material 2Dも確認項目へ加えることにしました。

## 今後の確認項目

同じような現象が起きた場合は、次の順番で確認します。

1. Consoleにエラーがないか
2. 移動用の速度が実際に設定されているか
3. 壁・崖判定が誤反応していないか
4. Colliderの形状に段差や隙間がないか
5. Physics Material 2DのFrictionを確認する

原因候補を一度に変更すると、どの修正が効いたのか分からなくなります。1項目ずつ確認し、変更後に同じ場所で再テストすることも大切でした。

## まとめ

Tilemap上で敵が止まった原因は、Composite Collider 2DやRaycastではなく、Collider同士の摩擦でした。

見た目では分かりにくい不具合でしたが、「Scriptが止めたのか」「物理的に進めないのか」を分けて考えることで原因へたどり着けました。

Unityの2D物理で移動オブジェクトが引っかかるときは、Colliderの形状だけでなく、Physics Material 2Dも確認する価値があります。

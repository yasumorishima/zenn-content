---
title: "transform: translateY(0) が position: fixed を壊す — SPAアニメーションの罠"
emoji: "🎭"
type: "tech"
topics: ["css", "nextjs", "react", "アニメーション", "frontend"]
published: false
---

## はじめに

ある日、自分が運営するNext.jsサイトでこんなバグ報告がありました。

> ギャラリーの下の方にある写真をクリックしても、ライトボックスが真っ暗で何も表示されない。スクロールして上に戻ると画像がある。

`position: fixed; inset: 0` で画面全体を覆うオーバーレイが、**スクロール位置によって表示されない**。ブラウザのバグ？いいえ、CSS仕様通りの動作でした。

## 再現条件

以下の2つが揃うと発生します。

1. **祖先要素に `transform` が設定されている**（`translateY(0)` でも！）
2. **子孫要素に `position: fixed` がある**

```css
/* ページ遷移アニメーション */
@keyframes page-enter {
  from {
    opacity: 0;
    transform: translateY(12px);
  }
  to {
    opacity: 1;
    transform: translateY(0); /* これが犯人 */
  }
}

.page-enter {
  animation: page-enter 0.35s ease both; /* both = 終了値を保持 */
}
```

```tsx
// ライトボックス（.page-enter の子孫）
<div className="fixed inset-0 z-50 bg-black/90">
  <img src={photo.url} />
</div>
```

ページ上部では問題なく見えます。しかしスクロールして下部で開くと、ライトボックスが**ページの上部に取り残されたまま**表示されます。

## なぜ起きるのか — CSS仕様

[MDN の position: fixed のドキュメント](https://developer.mozilla.org/en-US/docs/Web/CSS/position#fixed)にはこう書かれています。

> The element is removed from the normal document flow ... It is positioned relative to the initial containing block established by the viewport, **except when one of its ancestors has a `transform`, `perspective`, or `filter` property set to something other than `none`**.

つまり：

| 祖先の `transform` | `fixed` の基準 |
|---|---|
| `none` または未設定 | **viewport**（期待通り） |
| `translateY(0)` | **その祖先要素**（壊れる） |
| `translateY(12px)` | **その祖先要素**（壊れる） |

**`translateY(0)` は「0px移動する」という transform が設定されている状態**です。見た目は何も動かなくても、CSSエンジンは containing block を生成します。

## `animation-fill-mode: both` の罠

```css
.page-enter {
  animation: page-enter 0.35s ease both;
}
```

`both`（= `forwards` + `backwards`）は、アニメーション終了後も**最終フレームの値を保持**します。つまり `transform: translateY(0)` がページが存在する限りずっと残り続けます。

同様に、JavaScript で inline style を設定する場合も同じです。

```tsx
// IntersectionObserver で fadeIn するコンポーネント
<div style={{
  transform: visible ? "translateY(0)" : "translateY(16px)", // visible後もtranslateY(0)が残る
}}>
  {children}
</div>
```

## 影響範囲

`transform` を持つ祖先の子孫にある**全ての `fixed` 要素**が影響を受けます。

- ライトボックス / モーダル
- トースト通知
- Cookie同意バナー
- PWAインストールプロンプト
- プログレスバー
- スクロールトップボタン

ボトムナビやヘッダーは常にビューポートの端にあるため、見た目上は問題が顕在化しにくいですが、技術的には同じ影響を受けています。

## 修正方法

### 1. `transform: none` を使う（最重要）

```css
@keyframes page-enter {
  from {
    opacity: 0;
    transform: translateY(12px);
  }
  to {
    opacity: 1;
    transform: none; /* translateY(0) ではなく none */
  }
}
```

```tsx
<div style={{
  transform: visible ? "none" : "translateY(16px)", // none で containing block を解除
}}>
```

`transform: none` は「transform が設定されていない」と同義で、containing block を生成しません。

### 2. `createPortal` で DOM ツリーから脱出（防御的）

```tsx
import { createPortal } from "react-dom";

function Lightbox() {
  return createPortal(
    <div className="fixed inset-0 z-50 bg-black/90">
      {/* ... */}
    </div>,
    document.body // body直下にレンダリング → 祖先CSSの影響を受けない
  );
}
```

祖先に何があっても影響を受けません。モーダルやライトボックスなど、viewport 全体を覆う要素には `createPortal` を使うのがベストプラクティスです。

### 3. 両方やる（推奨）

`transform: none` で根本を直しつつ、`createPortal` で防御する。将来の変更で新たな `transform` が追加されても壊れません。

## まとめ

| やってはいけない | やるべき |
|---|---|
| `transform: translateY(0)` を終了値にする | `transform: none` を使う |
| `fixed` オーバーレイを深いDOMに直接レンダリング | `createPortal(document.body)` |
| アニメーション追加時に `fixed` 要素を確認しない | `transform` を使うアニメーション追加時は `fixed` 要素への影響をチェック |

**見た目が同じでも `translateY(0)` と `none` は別物。** この仕様を知らないと、アニメーションを追加した瞬間にサイト全体のオーバーレイが壊れます。

# Firebase セットアップ手順（集GO）

このアプリは **Firebase Realtime Database**（リアルタイム位置共有）と **匿名認証**（自分の位置ノードだけ書き込み可にする安全策）を使います。所要 10〜15 分。

## 1. プロジェクトを新規作成
位置情報は機微なので、自販機マップとは**別のプロジェクト**を作るのがおすすめです。

1. https://console.firebase.google.com/ → 「プロジェクトを追加」
2. 名前は任意（例：`machiawase-map`）。Google アナリティクスはオフでOK。

## 2. 匿名認証を有効化
1. 左メニュー **Authentication** → 「始める」
2. **Sign-in method** タブ → **匿名** を選んで「有効にする」

## 3. Realtime Database を作成
1. 左メニュー **Realtime Database** → 「データベースを作成」
2. ロケーションは `asia-southeast1`（東京に近い）など
3. 最初は「**ロックモードで開始**」を選択（ルールは次の手順で入れます）

## 4. セキュリティルールを設定
Realtime Database → **ルール** タブに、以下を貼り付けて「公開」。

```json
{
  "rules": {
    "sessions": {
      "$sid": {
        ".read": "auth != null",

        "meta": {
          ".write": "auth != null && (!data.exists() || data.child('hostUid').val() === auth.uid)",
          ".validate": "newData.hasChildren(['status'])",
          "status": {
            ".write": "auth != null && root.child('sessions').child($sid).child('participants').child(auth.uid).exists()"
          }
        },

        "participants": {
          "$uid": {
            ".write": "auth != null && $uid === auth.uid"
          }
        },

        "locations": {
          "$uid": {
            ".write": "auth != null && $uid === auth.uid",
            ".validate": "newData.hasChildren(['lat','lng','updatedAt']) && newData.child('lat').isNumber() && newData.child('lng').isNumber()"
          }
        },

        "meetPoint": {
          ".write": "auth != null && root.child('sessions').child($sid).child('participants').child(auth.uid).exists()",
          ".validate": "newData.hasChildren(['lat','lng']) && newData.child('lat').isNumber() && newData.child('lng').isNumber()"
        }
      }
    }
  }
}
```

ポイント：
- ログイン（匿名でも）していないと一切読み書きできない
- **自分の `uid` の位置ノードしか書けない**（他人になりすまして位置を偽装できない）
- `meta` を更新できるのはホストだけ（ただし `meta/status` は参加者なら更新可＝承認・終了のため）
- `meetPoint`（集合場所）は参加者なら更新可

> 補足：MVP では「招待リンク（=セッションID）を知っている認証ユーザーなら読める」設計です。セッションIDは推測不能なランダム値ですが、より厳密にしたい場合は「参加者リストに自分がいる人だけ読める」ルールに拡張できます（段階2で対応予定）。
>
> 人数上限（最大5人 / `meta.maxMembers`）はアプリ側で best-effort にチェックしています。厳密に強制したい場合は `participants` に `.validate`（`newData.getChildrenCount() <= 5`）を足すと安全です。

## 5. ウェブアプリを登録して config を取得
1. プロジェクト設定（⚙️）→ 「全般」→ 下の方「マイアプリ」→ **</> ウェブ** を追加
2. 表示される `firebaseConfig` をコピー
3. `index.html` の `// ▼▼▼ Firebase 設定` ブロックに貼り替え（特に `databaseURL` が入っているか確認）

```js
const firebaseConfig = {
  apiKey: "…",
  authDomain: "xxx.firebaseapp.com",
  databaseURL: "https://xxx-default-rtdb.asia-southeast1.firebasedatabase.app",
  projectId: "xxx",
  appId: "…",
};
```

## 6. 動作確認
- ローカル：`pwsh ./serve.ps1` → http://localhost:8080/ （`localhost` は位置情報OK）
- 2台でのテストや別端末からは **HTTPS が必須**（位置情報は安全なコンテキストでのみ動作）。GitHub Pages 等に上げてから 2 台で試すのが確実です。

## （任意）古いセッションの自動削除
RTDB は TTL を持たないため、放置データを消したい場合は Cloud Functions のスケジューラで `meta/expiresAt` を過ぎたセッションを削除すると綺麗です（MVP では未実装）。

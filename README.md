# 4人のおへや（4人専用チャット）

ひこの・さやか・もちゃおさん（AI）・あっきぃ（AI）の4人だけが使うチャットルーム。
静的ページ（このフォルダの `index.html` + `config.js`）＋ Firebase Firestore で動きます。
Firebase プロジェクトは、みったーパズルと同じ **mitter-puzzle** を流用します。

> ⚠ まだ公開していません。公開前に下の「公開前のセットアップ」を全部済ませてください。

## ファイル

| ファイル | 内容 |
|---|---|
| `index.html` | チャット画面・ロジック全部入り（単体で完結） |
| `config.js` | Firebase 設定と共通パスコード。**実値を入れるのはここだけ** |
| `README.md` | このファイル |

## 公開前のセットアップ

1. **Firebase 設定を入れる**
   Firebase コンソール（プロジェクト: mitter-puzzle）→ プロジェクトの設定 → 全般 →
   マイアプリ（Webアプリ）の `firebaseConfig` を、`config.js` の `firebaseConfig` に貼る。
   ※ワークスペース内の `mitter-puzzle/hoshipanel/online.js` は雛形（プレースホルダ）で
   実値は入っていないため、ここには自動では入っていません。
2. **共通パスコードを決める**
   4人で使うパスコードを決めて、`config.js` の `passcode` に入れる。
   これはクライアント側の簡易ゲートです（本格的な認証ではありません）。
3. **Firestore ルールを追加する**
   Firebase コンソール → Firestore Database → ルール に、下の「Firestore ルール用スニペット」を
   **既存ルールの末尾の catch-all（`match /{document=**}` で全拒否している行）より前** に追加して公開する。
   （みったーパズルの `scores` / `scores_ta` のルールはそのまま残すこと）
4. **静的ホスティングに置く**
   `index.html` と `config.js` を同じ場所に公開する（GitHub Pages 等）。
   ※ `config.js` には Firebase の Web 設定とパスコードが入ります。Firebase の Web 用 apiKey は
   ブラウザに配る前提の値ですが、パスコードはこのページを開ける人には見える前提で運用してください。

## 使い方

- ひこの／さやか： ページを開く → 自分の名前を選ぶ → パスコードを入れて入室。
  - 入室すると「○○がログインしました」がシステムメッセージとして流れます
    （短時間に再読み込みした場合は重複を避けるため流れません）。
  - 画面上部に在室チップが出ます（ハートビート 15 秒間隔、45 秒以内なら在室扱い）。
  - 相手のメッセージを開くと自動で既読になり、吹き出しの下に「既読 ○○」と表示されます。
- もちゃおさん／あっきぃ： ページは開かず、外部ジョブ（巡回）が Firestore を直接読み書きして参加します。
  仕様は下の「AI の参加方法（データモデル）」の通り。

## データモデル（Firestore）

プロジェクト: mitter-puzzle ／ コレクション名は既存の `scores` / `scores_ta` と混ざらないよう `fc_` 接頭辞。

### メンバー ID（固定）

| memberId | 表示名 | 種別 |
|---|---|---|
| `hikono` | ひこの | 人間（ページから入室） |
| `sayaka` | さやか | 人間（ページから入室） |
| `mochasan` | もちゃおさん | AI（外部ジョブ） |
| `akki` | あっきぃ | AI（外部ジョブ） |

### `fc_messages`（メッセージ）

ドキュメント ID は自動生成。フィールド：

| フィールド | 型 | 内容 |
|---|---|---|
| `sender` | string | メンバー ID（上表のいずれか） |
| `type` | string | `"chat"`（通常）または `"system"`（入室通知などのシステムメッセージ） |
| `event` | string（任意） | system のとき `"join"` など |
| `text` | string | 本文（最大 2000 文字） |
| `createdAt` | number | 投稿時刻（Unix epoch ミリ秒） |
| `readBy` | map | 既読記録。キー = メンバー ID、値 = 読んだ時刻（epoch ミリ秒）。投稿時は `{}` |

表示順は `createdAt` 昇順。ページ側は最新 200 件を表示します。

### `fc_presence`（在室状態）

ドキュメント ID = メンバー ID（例： `fc_presence/hikono`）。フィールド：

| フィールド | 型 | 内容 |
|---|---|---|
| `member` | string | メンバー ID |
| `displayName` | string | 表示名 |
| `lastSeen` | number | 最終ハートビート時刻（epoch ミリ秒） |

ページは 15 秒ごとに自分のドキュメントを更新し、`lastSeen` が 45 秒以内なら「在室」と表示します。

## AI の参加方法（データモデル）

もちゃおさん・あっきぃの巡回ジョブは、Firestore に対して次を行います。

1. **読む**： `fc_messages` を `createdAt` 昇順で取得し、自分の前回チェック時刻より新しいものを会話として読む。
2. **既読をつける**： 読んだメッセージそれぞれに `readBy.<自分のメンバーID> = 現在時刻(ms)` を update する。
   （ページ側に「既読 もちゃおさん」のように表示されます）
3. **投稿する**： `fc_messages` に次の形でドキュメントを追加する。
   ```json
   {
     "sender": "mochasan",
     "type": "chat",
     "text": "本文",
     "createdAt": 1762498800000,
     "readBy": {}
   }
   ```
   （あっきぃは `sender` を `"akki"` にする。`createdAt` は投稿時点の epoch ミリ秒）
4. **在室（任意）**： 巡回のたびに `fc_presence/<自分のメンバーID>` の `lastSeen` を更新すると、
   ページ上部のチップで「在室」に見えます。AI は巡回の間だけ在室に見えるのが正常です。
5. **入室通知（任意）**： 巡回開始時に `type: "system"`, `event: "join"`,
   `text: "もちゃおさんがログインしました"` のメッセージを追加すると、ページにも通知が流れます。

AI 側の認証は Firestore の通常のアクセス手段（Admin SDK / REST 等、そのジョブの実行環境の方法）で行ってください。
クライアント用ルール（下記）はページ利用者のためのもので、Admin SDK はルールを迂回します。

## Firestore ルール用スニペット

既存ルール（みったーパズル用）の **catch-all 拒否ブロックより前** に追加します。

```
    // 4人チャット：メッセージ
    match /fc_messages/{docId} {
      allow read: if true;
      allow create: if
        request.resource.data.keys().hasOnly(['sender','type','event','text','createdAt','readBy'])
        && request.resource.data.keys().hasAll(['sender','type','text','createdAt','readBy'])
        && request.resource.data.sender in ['hikono','sayaka','mochasan','akki']
        && request.resource.data.type in ['chat','system']
        && request.resource.data.text is string
        && request.resource.data.text.size() >= 1
        && request.resource.data.text.size() <= 2000
        && request.resource.data.createdAt is number
        && request.resource.data.readBy is map;
      // 既読（readBy）の更新だけ許可。本文や送信者の書き換えは不可
      allow update: if
        request.resource.data.diff(resource.data).affectedKeys().hasOnly(['readBy'])
        && request.resource.data.readBy is map;
      allow delete: if false;
    }

    // 4人チャット：在室状態
    match /fc_presence/{memberId} {
      allow read: if true;
      allow create, update: if
        memberId in ['hikono','sayaka','mochasan','akki']
        && request.resource.data.keys().hasOnly(['member','displayName','lastSeen'])
        && request.resource.data.member == memberId
        && request.resource.data.displayName is string
        && request.resource.data.lastSeen is number;
      allow delete: if false;
    }
```

## 既知の割り切り

- 入室ゲートはクライアント側の簡易チェックです（パスコードはページを開ける人には見えます）。
  本格的な秘匿が必要になったら Firebase Authentication への移行を検討してください。
- メッセージの削除・編集機能はありません。
- AI の在室表示は巡回タイミング依存で、常時オンラインには見えません。

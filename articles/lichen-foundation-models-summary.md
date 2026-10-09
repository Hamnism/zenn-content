---
title: "4,096トークンで掲示板をどう要約するか｜Foundation Modelsを専ブラに組み込んだ設計"
emoji: "🪨"
type: "tech"
topics: ["swift", "ios", "foundationmodels", "appleintelligence", "swiftui"]
published: true
---

こんにちは、Hamnismです。iOSで、掲示板専用ブラウザ「Lichen（ライケン）」を個人で開発しています。iPhoneのための、広告のない5ch専ブラです。

Lichenには、Apple Intelligenceを使ってスレのやり取りを3行に要約する機能があります。要約はすべて端末の中で作り、スレの内容を外部に送ることはありません。

この記事では、iOS 26で使えるようになったFoundation Modelsフレームワークを使って、この要約機能をどう設計・実装したかを紹介します。

:::message
この記事は、Lichenに搭載している機能の紹介を前提に、構成とAppleのAPIの使い方を中心に解説します。しきい値などの調整値や、プロンプトの全文は載せていません。
:::

## どんな機能か

Lichenでは、次の3か所で要約が使えます。

- **IDのポップアップ**：そのIDの人が、このスレで何を言っているか
- **返信ツリーのポップアップ**：そのツリーで、どんな議論になっているか
- **Homeのシート**：スレ全体の流れ（「何が起きているか」「流れの転換点」「今の論点」の3文）

使い方は、ブログで紹介しています。

- [Apple Intelligenceで長いスレを要約｜IDと返信ツリーを3行に](https://hamnism.com/lichen/blog/guide/ai-summary/)
- [せり上がるプレビュー｜スレごとに生まれるウィンドウとSUMMARY](https://hamnism.com/lichen/blog/guide/home-preview/)

## 前提：端末内モデルのコンテキストは4,096トークン

設計を決めるうえで、いちばん大きな前提はコンテキストの大きさでした。

Appleの技術ノート[TN3193](https://developer.apple.com/documentation/technotes/tn3193-managing-the-on-device-foundation-model-s-context-window)によると、端末内のモデルが扱えるのは、1つの`LanguageModelSession`あたり4,096トークンです。しかも、ここにはinstructions、プロンプト、モデルの応答のすべてが含まれます。

日本語は、およそ1文字が1トークンです。つまり、指示と応答のぶんを差し引くと、モデルに渡せる本文は数千文字ほどしかありません。

一方、5chのスレは1,000レスまで伸び、数万文字になることも珍しくありません。スレ全体をそのまま渡すことは、最初からできませんでした。

## 設計判断：スレ全体ではなく「範囲」を要約する

そこで、要約する範囲を絞ることにしました。

スレ全体を分割して要約し、それをさらに要約する、という方法もTN3193では紹介されています。ただ、5chのスレはひとつの話題で進むわけではなく、いくつもの話題が同時に流れています。全体を何段も要約していくと、時間もかかるうえに、ぼんやりした要約になりがちです。

Lichenで解決したかったのは、「途中からスレに来た人が、さっと流れをつかんで会話に混ざれること」でした。それなら、スレ全体よりも、

- この人は、このスレで何を主張しているのか
- このレスバトルや長いツリーは、何の話なのか

が分かるほうが役に立ちます。IDやツリーに絞れば、渡す量も自然に小さくなり、4,096トークンの枠にも収まりやすくなります。

## 全体の構成

構成は次のようになっています。

![Lichenの要約機能の構成。3つのUIが観点を指定してAISummarizerを呼び、AISummarizerがキャッシュの確認、入力の予算づけ、LanguageModelSessionの作成、respondでの生成、3行への整形を行う。モデルは端末内のSystemLanguageModel](/images/lichen-foundation-models-summary/architecture.png)

3つのUIは、どれも同じ`AISummarizer`を呼び出します。違うのは、渡す本文と「観点」だけです。ここからは、それぞれの部分をAppleのAPIに沿って見ていきます。

## 1. 使える端末かを確かめる

Foundation Modelsは、Apple Intelligenceに対応し、有効になっている端末でしか使えません。Lichenでは、`SystemLanguageModel.default.availability`を見て、使えないときは要約の欄そのものを出さないようにしています。

```swift
var isAvailable: Bool {
    if case .available = SystemLanguageModel.default.availability {
        return true
    }
    return false
}
```

「使えません」と表示するより、欄ごと出さないほうが、非対応の端末の人にとって自然だと考えました。要約以外の部分は、どの端末でもそのまま使えます。

## 2. 観点ごとにinstructionsを差し替える

3か所の要約は、同じ「3行の要約」でも、書き方を変えています。

- **ID**：その人を主語にして「〜と主張している」
- **ツリー**：第三者の目線で、論点・対立・結論
- **スレ**：3文それぞれに役割を持たせる

これを、観点を表す`enum`で切り替えています。

```swift
enum Perspective {
    case id(String)  // この人は、このスレで何を言っているか
    case tree        // このツリーで、どんな議論になっているか
    case thread      // スレ全体の流れ（3文に役割を持たせる）

    var instruction: String { /* 観点ごとの指示文 */ }
}
```

instructionsには、共通の指示（日本語で3行、簡潔な文体で、煽りや荒らしは除く）と、観点ごとの指示を組み合わせて渡しています。

TN3193にも「instructionsは短く、命令形で」とあるとおり、指示は欲張らずに短くしています。長い指示はトークンを使うだけでなく、応答の質を下げることもあるからです。

## 3. 入力を予算内に詰める

モデルに渡す本文は、新しいレスから順に、決めておいた量に収まるところまで詰めています。改行は1レス1行にまとめ、空のレスは除きます。

新しいレスを優先しているのは、「今どうなっているか」を知りたい場面が多いからです。古いレスから詰めると、スレの序盤の話ばかりの要約になってしまいます。

## 4. 1回の要約ごとにセッションを作る

`LanguageModelSession`は、要約を1回作るたびに新しく作っています。

```swift
let session = LanguageModelSession {
    // 共通の指示 + 観点ごとの指示
}
let response = try await session.respond(to: prompt, options: options)
```

セッションを使い回すと、前の要約のやり取りがコンテキストに積み重なり、4,096トークンの枠を圧迫していきます。要約はそれぞれが独立した仕事なので、毎回まっさらなセッションで実行するのが合っていました。

`GenerationOptions`では、温度を低めにして、毎回の出力がぶれすぎないようにしています。`maximumResponseTokens`も指定していますが、これは応答が暴走したときの保険です。TN3193では、厳しすぎる上限は文の途中で切れた応答の原因になるとされているので、余裕を持たせています。

## 5. 出力を3行に整える

「3行で」と指示しても、モデルが見出しや箇条書きの記号を付けてくることがあります。そこで、応答を受け取ったあとに、

- Markdownの強調や、行頭の番号・記号を取り除く
- 指示した3文の役割の名前を、そのまま見出しとして出してきた行を捨てる
- 空行を除いて、先頭から3行を使う

という後処理をしています。モデルの出力をそのまま信じず、表示する形に整える層を1枚はさむと、UIが安定します。

## 6. エラーを種類ごとの文言にする

生成に失敗したときは、`LanguageModelSession.GenerationError`の種類ごとに、ユーザー向けの文言に変えています。

```swift
catch let error as LanguageModelSession.GenerationError {
    switch error {
    case .exceededContextWindowSize: // 内容が長すぎた
    case .guardrailViolation:        // 要約の対象にできない内容だった
    case .assetsUnavailable:         // モデルの準備中
    case .rateLimited, .concurrentRequests: // 混み合っている
    default: break
    }
}
```

「要約に失敗しました」の一言で済ませず、理由が分かるようにしておくと、ユーザーも「少し待てばいいのか」「この内容は無理なのか」を判断できます。ポップアップでは、理由と一緒に「もう一度」のボタンを出しています。

## 7. キャッシュと、実行のタイミング

一度作った要約は、「スレ・要約の対象・その時点のレス数」をキーにして、メモリに保存しています。同じ対象を同じレス数のまま開き直したときは、推論せずにすぐ表示します。レスが増えればキーが変わるので、新しく作り直します。

実行のタイミングは、場所によって変えています。

- **ポップアップ**：ユーザーがタップしたときだけ
- **Homeのシート**：シートを開いたら自動で

端末内での推論には時間がかかり、端末への負荷もかかります。ポップアップは読む途中で何度も開くものなので、開くたびに推論が走らないよう、タップしたときだけにしました。シートは「スレを開く前にのぞく」ための画面なので、開いたら要約が出てくるほうが体験として自然です。

生成中は、要約の欄のふちを細い光が回る演出を入れて、待ち時間を「考えている時間」に見せています。iOSの「視差効果を減らす」がオンのときは、演出なしですぐ表示します。

## 開発で気づいたこと

### シミュレータでは動かない

Foundation Modelsの要約は、Lichenの開発環境ではシミュレータで動かず、実機での確認が必須でした。シミュレータで実行したときは、そのことが分かるエラー文言を出すようにしておくと、開発中に迷いません。

### 不適切な内容は、iOS側で止めてくれる

掲示板には荒れた内容も流れてきますが、不適切な内容の要約は、ある程度iOSの側で「要約できない」として止めてくれます。アプリ側では`guardrailViolation`を受け取って、その旨を表示するだけで済みました。

### スレのスラングを取り込んでまとめてくれる

逆に面白かったのは、スレごとのスラングや言い回しを取り込んだ形でまとめてくれることです。そのスレならではの空気感が残った要約になるので、読んでいて楽しい機能になりました。

## まとめ

端末内のLLMは、何でも渡して要約してもらう道具ではなく、「範囲を絞って使う部品」として扱うと、うまく機能すると感じました。

- コンテキストの枠（4,096トークン）から逆算して、要約する範囲を決める
- 観点ごとにinstructionsを差し替え、短く保つ
- 1回の要約ごとにセッションを作る
- 出力は後処理で整え、エラーは理由が分かる文言にする

Lichenは、App Storeで公開しています。掲示板をiPhoneで読む方は、ぜひ試してみてください。

- [Lichen（App Store）](https://apps.apple.com/jp/app/lichen/id6771805838)
- [Lichenのブログ](https://hamnism.com/lichen/blog/)

### 参考資料

- [TN3193: Managing the on-device foundation model's context window](https://developer.apple.com/documentation/technotes/tn3193-managing-the-on-device-foundation-model-s-context-window)
- [Foundation Models（Apple Developer Documentation）](https://developer.apple.com/documentation/foundationmodels)
- [Meet the Foundation Models framework（WWDC25）](https://developer.apple.com/videos/play/wwdc2025/286/)

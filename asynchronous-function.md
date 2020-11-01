original: <https://github.com/DougGregor/swift-evolution/blob/async-await/proposals/nnnn-async-await.md>
forum: <https://forums.swift.org/t/concurrency-asynchronous-functions/41619>


---

# Async/await

## 概要

最近のSwift開発では、クロージャや完了ハンドラを使った非同期プログラミングが多く使われていますが、これらのAPIは使いづらいものです。これは、多くの非同期操作が使われたり、エラー処理が必要になったり、非同期呼び出し間の制御フローが複雑になったりした場合に特に問題となります。この提案では、これをより自然でエラーの発生しにくいものにするための言語拡張を記述しています。

この設計では、Swiftにコルーチンモデルを導入します。関数は非同期であることを選択することができ、プログラマは通常のコントロールフローのメカニズムを使って非同期操作を含む複雑なロジックを構成することができます。コンパイラは、非同期関数を適切なクロージャとステートマシンのセットに変換する責任があります。

本提案では、非同期関数のセマンティクスを定義します。これは構造化された同時実行機能を導入するための提案で、非同期関数と同時実行タスクを関連付け、タスクの作成、問い合わせ、キャンセルのためのAPIを提供しています。

この提案は、Chris LattnerとJoe Groffによって書かれた以前の提案からインスピレーションを得ています。この提案自体は Oleg Andreev によって書かれた提案から派生しています。（再び）大幅に書き直され、多くの詳細が変更されていますが、非同期関数の核となる考え方は変わっていません。

## モチベーション：完了ハンドラは最適化でない

明示的なコールバック（完了ハンドラ（completion handlers）とも呼ばれる）を用いた非同期プログラミングには多くの問題があります。私たちは、非同期関数を言語に導入することで、これらの問題に対処することを提案します。非同期関数は、非同期コードを直線的なコードとして書くことを可能にします。また、コードの実行パターンを直接推論することができるので、コールバックをはるかに効率的に実行することができます。

### 問題1：破滅のピラミッド

単純な非同期演算のシーケンスは、しばしば深い入れ子になったクロージャを必要とします。これを示す例を作ってみました。

```swift
func processImageData1(completionBlock: (result: Image) -> Void) {
    loadWebResource("dataprofile.txt") { dataResource in
        loadWebResource("imagedata.dat") { imageResource in
            decodeImage(dataResource, imageResource) { imageTmp in
                dewarpAndCleanupImage(imageTmp) { imageResult in
                    completionBlock(imageResult)
                }
            }
        }
    }
}

processImageData1 { image in
    display(image)
}
```

この「破滅のピラミッド」は、コードがどこで実行されているかを読み取ったり、追跡したりすることを困難にします。さらに、クロージャのスタックを使用しなければならないことは、次に説明する多くの二次効果をもたらします。

### 問題2：エラーハンドリング

コールバックはエラー処理を難しくし、非常に冗長にします。Swift 2では同期コード用のエラー処理モデルが導入されましたが、コールバックベースのインターフェースはその恩恵を受けていません。

```swift 
func processImageData2(completionBlock: (result: Image?, error: Error?) -> Void) {
    loadWebResource("dataprofile.txt") { dataResource, error in
        guard let dataResource = dataResource else {
            completionBlock(nil, error)
            return
        }
        loadWebResource("imagedata.dat") { imageResource, error in
            guard let imageResource = imageResource else {
                completionBlock(nil, error)
                return
            }
            decodeImage(dataResource, imageResource) { imageTmp, error in
                guard let imageTmp = imageTmp else {
                    completionBlock(nil, error)
                    return
                }
                dewarpAndCleanupImage(imageTmp) { imageResult in
                    guard let imageResult = imageResult else {
                        completionBlock(nil, error)
                        return
                    }
                    completionBlock(imageResult)
                }
            }
        }
    }
}

processImageData2 { image, error in
    guard let image = image else {
        error("No image today")
        return
    }
    display(image)
}
```

標準ライブラリに`Result`が追加されたことで、Swift APIのエラー処理が改善されました。非同期APIは`Result`の主な動機の一つでした。

```swift
func processImageData2(completionBlock: (Result<Image, Error>) -> Void) {
    loadWebResource("dataprofile.txt") { dataResourceResult in
        dataResourceResult.map { dataResource in
            loadWebResource("imagedata.dat") { imageResourceResult in
                imageResultResult.map { imageResource in
                    decodeImage(dataResource, imageResource) { imageTmpResult in
                        imageTmpResult.map { imageTmp in 
                            dewarpAndCleanupImage(imageTmp) { imageResult in
                                completionBlock(imageResult)
                            }
                        }
                    }
                }
            }
        }
    }
}

processImageData2 { result in
    switch result {
    case .success(let image):
        display(image)
    case .failure(let error):
        error("No image today")
    }
}
```

`Result`を使用すると、エラーを適切にスレッドスルーすることが容易になり、コードを短くすることができます。しかし、クロージャの入れ子の問題が残っています。

### 問題点3：条件付きの実行が難しく、エラーが出やすい

非同期関数を条件付きで実行するのは非常に面倒です。例えば、ある画像を取得した後に「Swizzle」する必要があるとします。しかし、swizzleする前に画像をデコードするために非同期呼び出しをしなければならないことがあります。おそらく、この関数を構造化するための最良のアプローチは、以下のように、条件付きで補完ハンドラに取り込まれるヘルパーの「継続（continuation）」クロージャにSwizzlingコードを記述することです。

```swift
func processImageData3(recipient: Person, completionBlock: (result: Image) -> Void) {
    let swizzle: (contents: image) -> Void = {
      // ... continuation closure that calls completionBlock eventually
    }
    if recipient.hasProfilePicture {
        swizzle(recipient.profilePicture)
    } else {
        decodeImage { image in
            swizzle(image)
        }
    }
}
```

このパターンは、関数の自然なトップダウン構成を逆にしています。関数の後半で実行されるコードは、前半で実行される部分の前に現れなければなりません。関数全体を再構築することに加えて、クロージャは補完ハンドラで使用されるため、継続クロージャ内のキャプチャについても慎重に考えなければなりません。この問題は、条件付きで実行される非同期関数の数が増えるにつれて悪化し、本質的には逆さの「破滅のピラミッド」となってしまいます。

### 問題4：たくさんのミスが起こりやすくなる

正しい完了ハンドラブロックを呼び出さずに非同期処理を早期リターンすることが非常に簡単にできてしまいます。忘れてしまうと、この問題をデバッグするのが非常に困難になります。

```swift 
func processImageData4(completionBlock: (result: Image?, error: Error?) -> Void) {
    loadWebResource("dataprofile.txt") { dataResource, error in
        guard let dataResource = dataResource else {
            return // <- forgot to call the block
        }
        loadWebResource("imagedata.dat") { imageResource, error in
            guard let imageResource = imageResource else {
                return // <- forgot to call the block
            }
            ...
        }
    }
}
```

ブロックを呼んだとしても、その後にreturnするのを忘れてしまうことがあります。

```swift
func processImageData5(recipient:Person, completionBlock: (result: Image?, error: Error?) -> Void) {
    if recipient.hasProfilePicture {
        if let image = recipient.profilePicture {
            completionBlock(image) // <- forgot to return after calling the block
        }
    }
    ...
}
```

ありがたいことに、ガード構文は戻り忘れをある程度防いでくれますが、必ずしも関連性があるとは限りません。

### 問題5：完了ハンドラが厄介なため、同期的に定義されるAPIが多すぎる

これを定量化するのは難しいですが、著者らは、非同期APIの定義と使用（完了ハンドラを使用）の厄介さが、ブロックできるAPIであっても見かけ上は同期的な振る舞いで定義されているものが多い事であると考えています。これはUIアプリケーションのパフォーマンスや応答性に問題があり、例えばカーソルが回転するなどの問題を引き起こす可能性があります。また、サーバーなどのスケールを達成するために非同期が重要な場合には、使用できないAPIの定義につながる可能性もあります。

## 提案する解決策：async/await

非同期関数（async/awaitとして知られていることが多い）は、非同期コードをあたかも直線的な同期コードであるかのように書くことを可能にします。これにより、プログラマは同期コードで利用可能な言語構造をフルに利用することができ、上述した多くの問題を即座に解決することができます。async/await の使用は、コードのセマンティック構造を自然に保持し、少なくとも 3 つの横断的な言語改善に必要な情報を提供します。(1) 非同期コードのパフォーマンスの向上、(2) デバッグ、プロファイリング、コード探索の際に、より一貫した体験を提供するためのツールの改善、(3) タスクの優先度やキャンセルなどの将来の同時実行機能のための基盤です。前のセクションの例では、async/awaitがいかに非同期コードを劇的に単純化しているかを示しました。

```swift
func loadWebResource(_ path: String) async throws -> Resource
func decodeImage(_ r1: Resource, _ r2: Resource) async throws -> Image
func dewarpAndCleanupImage(_ i : Image) async throws -> Image

func processImageData2() async throws -> Image {
  let dataResource  = await try loadWebResource("dataprofile.txt")
  let imageResource = await try loadWebResource("imagedata.dat")
  let imageTmp      = await try decodeImage(dataResource, imageResource)
  let imageResult   = await try dewarpAndCleanupImage(imageTmp)
  return imageResult
}
```

async/await の多くの記述では、関数を複数のコンポーネントに分割するコンパイラパスという共通の実装メカニズムを使って説明しています。これは、マシンがどのように動作しているかを理解するための低レベルの抽象度では重要ですが、高レベルでは無視することをお勧めします。その代わりに、非同期関数はスレッドを放棄する特別な力を持つ普通の関数だと考えてください。非同期関数は通常この力を直接使うことはありませんが、その代わりに呼び出しを行い、その呼び出しによってスレッドを放棄して何かが起こるのを待つ必要があることもあります。それが完了すると、関数は再び実行を再開します。

同期関数との類似性は非常に強いです。同期関数は呼び出しを行うことができ、呼び出しが行われると、関数は直ちに呼び出しが完了するのを待ちます。呼び出しが完了すると、制御は関数に戻り、中断したところから再開します。非同期関数でも同じことが言えます。非同期関数は通常通りに呼び出しを行うことができます。呼び出しが完了すると、制御は関数に戻り、元の位置に戻ります。唯一の違いは、同期関数はスレッドとそのスタック（の一部）をフルに活用できることですが、非同期関数はそのスタックを完全に放棄して、独自の別のストレージを使うことができます。非同期関数に与えられたこのような追加のパワーは実装コストがかかりますが、全体的に設計することでそのコストを大幅に削減することができます。

非同期関数はスレッドを放棄することができなければならず、同期関数はスレッドを放棄する方法を知らないので、同期関数は通常非同期関数を呼び出すことができません。非同期関数は自分が占有していたスレッドの一部を放棄することしかできず、もし放棄しようとした場合、同期関数の呼び出し元はそれをリターンのように扱い、リターン値なしで元の場所に戻ろうとするでしょう。一般的にこれを実現する唯一の方法は、非同期関数が再開されて完了するまでスレッド全体をブロックすることですが、これでは非同期関数の目的が完全に失われてしまいます。

対照的に、非同期関数は同期関数と非同期関数のどちらを呼び出すこともできます。もちろん、同期関数を呼び出している間はスレッドを放棄することはできません。実際、非同期関数は自然にスレッドを放棄することはありません。これらの関数がスレッドを放棄するのは、「サスペンドポイント（suspension points）」と呼ばれる、`await`でマークされたポイントに到達したときだけです。サスペンドポイントは関数内で直接発生することもあれば、関数が呼び出す別の非同期関数内で発生することもありますが、どちらの場合も関数とその非同期関数の呼び出し元は同時にスレッドを放棄します。(実際には、非同期関数は非同期呼び出しの間はスレッドに依存しないようにコンパイルされているので、一番内側の関数だけが余分な作業をする必要があります)

制御が非同期関数に戻ってきたときには、元の場所に戻ってきます。しかし、これは必ずしも以前と全く同じスレッドで実行されているとは限りません。この設計では、スレッドはほとんどが実装メカニズムであり、意図された同時実行のインターフェースの一部ではありません。しかし、多くの非同期関数はただの非同期ではありません。それらは特定のアクター(これは別の提案の主題です)に関連付けられており、常にそのアクターの一部として実行されることになっています。Swiftはそのような関数が実際にアクタに戻って実行を終えることを保証しています。したがって、状態分離のために直接スレッドを使うライブラリ、例えば、独自のスレッドを作成し、その上にタスクを順次スケジューリングすることで、これらの基本的な言語保証が適切に機能するようにするために、一般的にはそれらのスレッドをSwiftのアクタとしてモデル化するべきです。

### サスペンドポイント

サスペンドポイントとは、非同期関数の実行中にスレッドを放棄しなければならないポイントのことです。サスペンドポイントは常に、関数内の何らかの決定論的で、構文的に明示的なイベントに関連付けられています。関数の観点から見れば、決して隠されていたり、非同期であったりすることはありません。詳細な言語設計では、いくつかの異なる操作をサスペンドポイントとして記述しますが、最も重要なのは、異なる実行コンテキストに関連付けられた非同期関数の呼び出しです。

サスペンドポイントは明示的な操作にのみ関連付けられることが重要です。実際、この提案ではサスペンドする可能性のある呼び出しを`await`式で囲むことを要求しているほど重要です。これは、エラーを投げる可能性のある関数への呼び出しをカバーするために`try`式を要求するというSwiftの前例を踏襲しています。サスペンドポイントをマークすることは、サスペンドによってアトミック性が中断されるので、特に重要です。例えば、非同期関数がシリアルキューによって保護されたコンテキスト内で実行されている場合、サスペンドポイントに到達することは、他のコードが同じシリアルキュー上でインタリーブされることを意味します。このアトミック性が重要になる古典的な例として、銀行のモデリングがあります。ある口座に預金が入金されたが、一致した引き出しを処理する前に操作が中断された場合、その資金が二重に使われる可能性のあるウィンドウが作成されます。多くのSwiftプログラマーにとってより本質的な例はUIスレッドです。サスペンドポイントはUIをユーザーに表示できるポイントです。(サスペンド ポイントは、明示的なコールバックを使用してコード内で明示的に呼び出されることにも注意してください。サスペンドは、外部関数がリターンしてからコールバックの実行が開始されるまでの間に発生します)。すべてのサスペンドポイントがマークされていることを必須とすることで、プログラマはサスペンドポイントのない場所はアトミックな振る舞いをすると安全に仮定することができ、問題のあるアトミックでないパターンをより簡単に認識することができます。

サスペンドポイントは非同期関数の中で明示的にマークされた場所にしか現れないので、長い計算でもスレッドがブロックされる可能性があります。これは、同期関数を呼び出して多くの作業を行う場合や、非同期関数内で直接書かれた激しい計算ループに遭遇した場合などに起こります。どちらの場合でも、これらの計算を実行している間はスレッドはコードをインターリーブすることができません。これは通常は正しい選択ですが、スケーラビリティの問題になることもあります。激しい計算を行う必要がある非同期プログラムは、一般的には別のコンテキストで実行する必要があります。それが不可能な場合は、人為的にサスペンドして他の演算をインターリーブできるようにするライブラリ機能があるでしょう。

非同期関数は、実際にスレッドをブロックするような関数の呼び出しは避けるべきです。特に、現在実行中であることが保証されていない作業を待っている間にブロックすることができる場合は、特に注意が必要です。特に、現在実行中であることが保証されていない作業を待つスレッドをブロックすることができる場合は、そのような関数を呼び出すことは避けるべきです。例えば、ミューテックスを取得しても、現在実行中のスレッドがミューテックスを放棄するまでしかブロックできません。これは許容できる場合もありますが、デッドロックや人為的なスケーラビリティの問題が発生しないように注意して使用しなければなりません。これとは対照的に、条件変数での待機は、その変数にシグナルを送る任意の他の作業がスケジュールされるまでブロックすることができます。このパターンは推奨されていることに強く反しています。プログラムがこれらの落とし穴を回避できるような抽象化を提供するための継続的なライブラリ作業が必要になります。

このデザインは現在のところ、非同期関数が別のコンテキストで操作を待っている間に、現在のコンテキストがコードをインターリーブするのを防ぐ方法を提供していません。この省略は意図的なものです。インターリーブの防止を可能にすることは、本質的にデッドロックになりやすいのです。

## 非同期呼び出し

非同期関数の呼び出しは、ほとんどの場合、同期関数（または通常の関数）を呼び出すのと同じように見え、動作します。非同期関数への呼び出しの見かけ上のセマンティクスは以下の通りです。

1. 引数は、`inout`パラメータへのアクセスの開始を含む、通常のルールを使用して評価されます。
1. 呼び出先のエクセキューターが決定されます。この提案では、呼び出先のエクセキューターを決定するためのルールは記述していません。アクターのプロポーザルを読んでください。
1. 呼び出先のエクセキューターが呼び出し側のエクセキューターと異なる場合、一時停止が発生し、呼び出先の実行を再開するための部分タスクが呼び出先の実行者にエンキューされる。
1. 呼び出先は、そのエクセキューター上で与えられた引数と共に実行されます。
1. returnする間、呼び出先のエクセキューターが呼び出し側のエクセキューターと異なる場合、一時停止が発生し、呼び出し側での実行を再開するための部分タスクが呼び出し側のエクセキューターにエンキューに入れられる。
1. 最後に、呼び出し元はそのエクセキューター上で実行を再開する。呼び出し元が正常に返された場合、呼び出し式の結果は関数によって返された値であり、そうでない場合、式は呼び出し元から投げられたエラーを投げます。

呼び出し元の視点から見ると、非同期呼び出しは同期呼び出しと似たような動作をしますが、異なるエクセキューター上で実行される可能性があり、タスクを一時的に中断する必要があります。また、`inout`アクセスの持続時間は、呼び出しが中断されるため、潜在的にはるかに長くなる可能性があることに注意してください。そのため、十分に分離されていない共有された変異可能な状態への`inout`参照は、動的排他性違反を生成する可能性が高くなります。

## 詳細なデザイン

関数タイプは、関数が非同期であることを示す非同期を明示的にマークすることができます。

```swift
func collect(function: () async -> Int) { ... }
```

関数やイニシャライザの宣言は、明示的に非同期で宣言することもできます。

```swift
class Teacher {
  init(hiringFrom: College) async throws {
    ...
  }
  
  private func raiseHand() async -> Bool {
    ...
  }
}
```

非同期宣言された関数またはイニシャライザへの参照の型は、非同期関数型です。参照がインスタンスメソッドへの "カリー化された"静的参照である場合、そのような参照に対する通常のルールと一致して、非同期となるのは "内部 "の関数型です。

`deinit`やストレージアクセサのような特殊な関数は非同期にすることはできません。

> __理由__：ゲッターのみを持つプロパティは、`async`である可能性があります。しかし、`async`セッターを持つプロパティは、そのプロパティを `inout`として渡して、そのプロパティ自体のプロパティをドリルダウンする能力を意味しますが、これはセッターが実質的に「瞬間的な」（同期的な、投げない）操作であることに依存します。`async`プロパティを禁止することは、取得のみの`async`プロパティを許可するよりも簡単なルールです。

関数が`async`と`throws`の両方である場合、型宣言では`async`キーワードが`throws`の前になければなりません。この規則は、`async`と`rethrows`の両方の場合にも適用されます。

> __理由__：この順序制限は恣意的ではあるが、弊害はなく、文体論的な議論の可能性を排除している。

### 非同期関数型

非同期関数型は同期関数型とは異なります。同期関数型の値から対応する非同期関数型への暗黙の変換はありません。しかし、非`throw`非同期関数型の値から対応するthrowする同期関数型への暗黙の変換は許可されています。例えば、以下のようになります。

```swift
struct FunctionTypes {
  var syncNonThrowing: () -> Void
  var syncThrowing: () throws -> Void
  var asyncNonThrowing: () async -> Void
  var asyncThrowing: () async throws -> Void
  
  mutable func demonstrateConversions() {
    // Okay to convert to throwing form
    syncThrowing = syncNonThrowing
    asyncThrowing = asyncNonThrowing
    
    // Error to convert between asynchronous and synchronous
    asyncNonThrowing = syncNonThrowing // error
    syncNonThrowing = asyncNonThrowing // error
    asyncThrowing = syncThrowing       // error
    syncThrowing = asyncThrowing       // error
  }
}
```

同期関数を呼び出す`async`クロージャを手動で作成できるので、暗黙の変換がないことはモデルの表現力に影響しません。`async`クロージャを定義する構文については、「クロージャ」のセクションを参照してください。

> __理由__：同期関数から非同期関数への暗黙の変換は型チェックを複雑にするので提案していません。特に、同じ関数の同期オーバーロードと非同期オーバーロードが存在する場合には注意が必要です。詳細は「オーバーロードとオーバーロードの解決」を参照してください。

### Await式

非同期関数型の値への呼び出し（非同期関数への直接呼び出しを含む）は、サスペンドポイントを導入します。サスペンドポイントは、非同期コンテキスト（例えば、非同期関数）内で発生しなければなりません。さらに、`await`式のオペランド内で発生しなければなりません。

次の例を考えてみましょう。

```swift
// func redirectURL(for url: URL) async -> URL { ... }
// func dataTask(with: URL) async throws -> URLSessionDataTask { ... }

let newURL = await server.redirectURL(for: url)
let (data, response) = await try session.dataTask(with: newURL)
```

このコード例では、`redirectURL(for:)`と`dataTask(with:)`は非同期関数であるため、タスクの一時停止が発生する可能性があります。それぞれの呼び出しにはサスペンドポイントが含まれているため、両方の呼び出し式は何らかの`await`式の中に含まれていなければなりません。`await`式のオペランドには少なくとも1つのサスペンドが含まれていなければなりませんが、1つの`await`のオペランド内には複数のサスペンドが許されています。例えば、次のように書き換えることで、1つの`await`を使用して、この例の両方のサスペンドポイントをカバーすることができます。

```swift
let (data, response) = await try session.dataTask(with: server.redirectURL(for: url))
```

`await`には追加のセマンティクスはありません。`try`のように、単に非同期呼び出しが行われていることを示すだけです。`await`式の型はオペランドの型であり、結果はオペランドの結果です。

> __理由__：非同期呼び出しはサスペンドポイントを導入してしまうため、関数内で明確に識別できるようにすることが重要です。サスペンドポイントは呼び出しに固有のものかもしれませんし(非同期呼び出しは別の実行者で実行しなければならないため)、単に呼び出し元の実装の一部かもしれませんが、どちらの場合も意味的に重要であり、プログラマはそれを認識する必要があります。また、`await`式は非同期コードの指標でもあり、クロージャの推論と相互作用します。詳細は「クロージャ」のセクションを参照してください。

サスペンドポイントは、非同期関数型ではないオートクロージャー内で発生してはいけません。

サスペンドポイントは、`defer`ブロック内で発生してはいけません。

### クロージャ

クロージャは非同期関数型を持つことができます。このようなクロージャは、以下のように明示的に非同期であることを示すことができます。

```swift
{ () async -> Int in
  print("here")
  return await getInt()
}
```

匿名クロージャは、それが`await`式を含む場合、非同期関数型であると推定されます。

```swift
let closure = { await getInt() } // implicitly async

let closure2 = { () -> Int in     // implicitly async
  print("here")
  return await getInt()
}
```

クロージャに対する非同期の推論は、そのクロージャを囲む関数や入れ子になっている関数、クロージャには伝わらないことに注意してください。例えば、この状況ではクロージャ6だけが非同期であると推論されます。

```swift
// func getInt() async -> Int { ... }

let closure5 = { () -> Int in       // not 'async'
  let closure6 = { () -> Int in     // implicitly async
    if randomBool() {
      print("there")
      return await getInt()
    } else {
      let closure7 = { () -> Int in 7 }  // not 'async'
      return 0
    }
  }
  
  print("here")
  return 5
}
```

### オーバーロードとオーバーロードの解決

操作のための同期と非同期の両方のエントリポイントを含む既存のSwiftプログラムは、各操作のために2つの類似した名前のメソッドを使用して設計されている可能性が高いです。

```swift
func doSomething() -> String { ... }
func doSomething(completionHandler: (String) -> Void) { ... }
```

呼び出しサイトでは、完了ハンドラの存在の有無によって、どのメソッドが呼び出されているかが明らかになります。しかし、2 番目のメソッドの API を非同期のものに直接マッピングしたことで、シグネチャは非常に似てきました。

```swift
func doSomething() -> String { ... }
func doSomething() async -> String { ... }

doSomething() // synchronous or asynchronous?
```

もし、`async`を`throws`に置き換えると、上記の 2 つのメソッドを宣言すると "invalid redeclaration" というコンパイラエラーが発生します。しかし、非同期関数をオーバーロードするために非同期関数を許可することを提案しているので、上記のコードは正しいです。これにより、既存のSwiftプログラムは、見せかけのリネームなしに、既存の同期関数の非同期バージョンを進化させることができます。

非同期関数と同期関数をオーバーロードする機能は、呼び出しのコンテキストに基づいて適切な関数を選択するためのオーバーロード解決ルールとペアになっています。呼び出しがあった場合、オーバーロード解決は、同期関数内で非同期関数を呼ぶ事はできないので、同期コンテキスト内では同期関数を優先します。さらに、オーバーロード解決は非同期コンテキスト内では非同期関数を優先します。なぜなら、非同期関数から大体があるときにAPIをブロックする同期関数を呼ぶ事は避けられるべきだからです。オーバーロード解決が非同期関数を選択した場合、その呼び出しは`await`式の中で発生しなければならないというルールに従うことになります。

### Autoclosures

関数は、関数自体が非同期でない限り、非同期関数型のオートクロージャーパラメータを取ることはできません。例えば、以下の宣言は不定形です。

```swift
// error: async autoclosure in a function that is not itself 'async'
func computeArgumentLater<T>(_ fn: @escaping @autoclosure () async -> T) { } 
```

この制限はいくつかの理由で存在します。次の例を考えてみましょう。

```swift
// func getIntSlowly() async -> Int { ... }

let closure = {
  computeArgumentLater(await getIntSlowly())
  print("hello")
}
```

一見すると、この await 式は、プログラマが computeArgumentLater(_:) を呼び出す前にサスペンドポイントがあることを暗示しているように見えますが、実際はそうではありません。サスペンドポイントは、`computeArgumentLater(_:)`のボディ内で渡され、使用される(auto)クロージャの中にあります。これはいくつかの問題を引き起こします。第一に、 `await`が呼び出しの前にあるように見えるという事実は、クロージャが非同期関数型を持っていると推測されることを意味しています。第二に、`await`のオペランドは、その中のどこかにサスペンドポイントを含む必要があるだけなので、 呼び出しの等価な書き換えは、次のようにすべきです。

```swift
await computeArgumentLater(getIntSlowly())
```

しかし、引数がオートクロージャであるため、この書き換えはもはや意味論的に保存されません。このように、`async`オートクロージャパラメータの制限は、`async`オートクロージャパラメータが非同期コンテキストでのみ使用できることを保証することで、これらの問題を回避します。

## ソース互換性

既存のコードは新機能を使用しておらず(例: `async` 関数やクロージャを作成しない)、影響を受けることはありません。しかし、この提案では `async` と `await` という2つの新しいコンテキストキーワードが導入されています。

文法の中での `async` の新しい用途の位置（関数宣言、関数型、let の接頭辞として）によって、ソースの互換性を壊すことなく `async` を文脈キーワードとして扱うことができます。ユーザー定義の async は、整形されたコードでは、これらの文法的な位置には出現しません。

コンテキストキーワードの `await` は、式の中で発生するため、より問題があります。例えば、今日のSwiftで関数 `await` を定義することができます。

```swift
func await(_ x: Int, _ y: Int) -> Int { x + y }

let result = await(1, 2)
```

これは今日のコードで、`await`関数の呼び出しである。この提案では、このコードは `(1, 2)` という副式を持つ `await` 式になります。`await` は非同期コンテキスト内でのみ使用することができ、既存の Swift プログラムにはそのようなコンテキストが存在しないので、コンパイルエラーとなります。このような関数は一般的ではないようなので、これは `async`/`await` の導入の一環として許容できるソースブレークだと考えています。

## ABI安定性への影響

非同期関数や関数型はABIに加算されるため、既存の（同期）関数や関数型は変化しないため、ABIの安定性に影響はありません。

## APIの復元力への影響

非同期関数のABIは同期関数のABIとは全く異なります（例えば、呼び出し規約が互換性がないなど）ので、関数や型から非同期を追加したり削除したりすることは回復力のある変更ではありません。

## 既存のプロポーザル

この提案に加えて、Swift Concurrency モデルのさまざまな側面をカバーするいくつかの関連する提案があります。

- Objective-C との相互運用性：Objective-C との相互運用性、特に補完ハンドラを受け入れる非同期 Objective-C メソッドと @objc 非同期 Swift メソッドとの関係について説明します。
- 構造化された並行性：非同期呼び出しで使用されるタスク構造、子タスクと離脱タスクの両方の作成、キャンセル、優先順位付け、およびその他のタスク管理APIについて説明します。
- アクター：並行プログラムの状態分離を提供するアクターモデルについて説明します。
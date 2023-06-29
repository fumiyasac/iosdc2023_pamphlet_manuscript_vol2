## UIKit＆SwiftUIとCombineを組み合わせた処理で上手にUnitTestを整えていくアイデア解説

<p align="right">
<strong>酒井文也 (Fumiya Sakai) Twitter &amp; github: @fumiyasac</strong>
</p>

<hr>

### はじめに

アプリ開発の中で、画面要素と内部ロジックとを結合する処理等でCombineを利用する場合もあると思います。iOS13以降で登場したApple純正のCombineの登場によって、これまで取り扱いが難しかった、APIリクエストを伴う非同期処理の取り扱いを、ネストされたクロージャーを利用した処理や、RxSwift・ReactiveSwift等の様な外部OSSを利用せずとも処理を記述する事が可能になりました。

それだけに留まらず、SwiftUIを利用した場合では __「配置しているSwiftUIで作成された画面要素とのデータの双方向のBinding処理を実現する」__ 際はもちろん、UIKitを利用した画面の場合においても、 __「Presentation層(ViewModel層)で何らかの処理を要求した場合に対応する画面要素側での処理と連動する様な関係を実現する」__ 際においても、とても有効かつ強力なパートナーになり得るのではないかと思います。

この様な処理を実現したい場合はもとより、Combine+SwiftUI（場合によってはUIKit）の組み合わせをより強力かつ上手に活用していきたい場合においては、Combineそのものに関する理解を深める事に加えて、仕様把握や機能担保の観点において __「Combineを利用した場合におけるUnitTestを活用方法」__ に関して理解を深めていく事は、今後大きな意義を持つと考えております。

本稿では、主に __「Presentation層(ViewModel層)と画面ないしはView要素間における連結処理部分」__ を題材として __「Combineを活用していく際における処理のポイントとなり得る部分」__ のご紹介や __「Combineを利用する際のUnitTestを記述する際のアイデアと事例」__ を、コードを通じて簡単に解説できればと考えております。

対象の読者としましては __「Combineを利用した処理を実際にどの様な形で活用しているかを知りたい方」__  や __「Combineを利用した処理には触れた経験はあるが実際のUnitTestの記述例を知りたい方」__ 等を想定しております。決して特別な事はありませんが、ほんの少しでも皆様のご参考になる事ができれば嬉しく思います。

### 1. MVVMアーキテクチャでCombineを活用する際の基本イメージ

本稿では、MVVMアーキテクチャ（Model-ViewModel-View構成）を取る様な画面処理部分に対して、まずは想定している処理イメージをまずは簡単に整理していくことにします。私自身もこれまでの実務経験の中でRxSwiftを活用したコードに触れる機会が数多くありましたので、MVVMアーキテクチャをRxSwiftを利用する場合と、Combineを利用した場合とで簡単に比較してみる事にします。

こちらで例示している例は、比較的簡単なものではありますが、実装の方針次第では、ベースとなる部分においては類似した考え方ができる余地は大いにあるかと思います。

（ここに図解1が入ります）

### 2. `@Published`・`PassthroughSubject`・`AnyPublisher`を利用した処理で画面要素と内部ロジック間を結合する処理のポイント

ここからは、ViewModelクラス側の構造や想定している処理機構に注目しながらポイントとなり得そうな部分についてコードを交えながら解説ができればと思います。

#### ⭐️2-1. UIKitを利用時でのCombineを利用する処理イメージ

UIKitを利用した画面におけるViewModelクラス処理においてCombineをベースとした実装を考えていく場合の一例で、ここでは __PassthroughSubject・AnyPublisher__ を活用する事でViewModel側でInput(入力用)・Output(出力用)の処理を定義し、ViewModel側での画面表示要素データを取得等の処理とViewController側での処理結果に対応した画面表示要素の更新処理を結合する様な処理例を考えてみます。

下記の様な形でViewModelクラス内部で展開する、API非同期通信処理やデータ永続化処理を組み合わせる事で実現する様にすると良さそうに思います。

- __Input:__ 画面からViewModel内で定義した処理を実行する
  👉 __PassthroughSubject<ViewModelに送る値の型, Never>__
- __InputとOutputの仲介:__ ViewModel内に定義した変数の値変化を受け取る
  👉 __privateで定義した@Publishedの変数を利用する__
- __Output:__ ViewModel内に定義した変数の値変化を受け取る
  👉 __AnyPublisher<期待する値の型, Never>__

後述するコード事例については、Kickstarterというクラウドファンディング事業を展開しているサービスがGithub上でOSSとして公開しているiOSアプリ内で利用されているViewModelクラスでの実装を参考にしています。

- __Kickstarterが公開しているリポジトリ__
  - GitHub: https://github.com/kickstarter/ios-oss
- __Kickstarter-iOSのViewModelの作り方がウマかった__
  - 記事URL: https://qiita.com/muukii/items/045b12405f7acff1a9fd
- __Introducing ViewModel Inputs/Outputs: a modern approach to MVVM architecture__
  - 記事URL: https://engineering.mercari.com/en/blog/entry/2019-06-12-120000/

__【🌷記事一覧を取得する「ArticleViewModel.swift」の構築例】__

```swift
// ----------
// 📝 ArticlesViewModel.swift
// 👉 記事データ一覧を取得して画面に表示させる想定のもの
// ----------

// MARK: - Protocol

protocol ArticlesViewModelInputs {
    var fetchArticlesTrigger: PassthroughSubject<Void, Never> { get }
}

protocol ArticlesViewModelOutputs {
    var articles: AnyPublisher<[Article], Never> { get }
}

protocol ArticlesViewModelType {
    var inputs: ArticlesViewModelInputs { get }
    var outputs: ArticlesViewModelOutputs { get }
}

final class ArticlesViewModel: ArticlesViewModelType, ArticlesViewModelInputs, ArticlesViewModelOutputs {

    // MARK: - ArticlesViewModelType

    var inputs: ArticlesViewModelInputs { return self }
    var outputs: ArticlesViewModelOutputs { return self }

    // MARK: - ArticlesViewModelInputs

    let fetchArticlesTrigger = PassthroughSubject<Void, Never>()

    // MARK: - ArticlesViewModelOutputs

    var articles: AnyPublisher<[Article], Never> {
        return $_articles.eraseToAnyPublisher()
    }

    private let api: APIRequestManagerProtocol

    private var cancellables: [AnyCancellable] = []

    // MARK: - @Published

    // 👉 InputとOutputを仲介するための変数
    // fetchArticlesTrigger実行 → この値が更新 → var articles: AnyPublisher<[Article], Never>へ値が流れる
    @Published private var _articles: [Article] = []

    // MARK: - Initializer

    init(api: APIRequestManagerProtocol) {

        // MEMO: 適用するAPIリクエスト用の処理
        self.api = api

        // 👉 画面からInputが実行されたらAPIリクエスト処理を実行する
        fetchArticlesTrigger
            .sink(
                receiveValue: { [weak self] in
                    self?.fetchArticles()
                }
            )
            .store(in: &cancellables)
    }

    // MARK: - deinit

    deinit {
        cancellables.forEach { $0.cancel() }
    }

    // MARK: - Privete Function

    // MEMO: APIリクエストを実行して記事データ一覧の取得をするメソッド
    private func fetchArticles() {
        api.getArticles()
            .receive(on: RunLoop.Articles)
            .sink(
                receiveCompletion: { completion in
                    switch completion {
                    case .finished:
                        // MEMO: APIから値取得処理が「成功」した際に実行される部分
                    case .failure(let error):
                        // MEMO: APIから値取得処理が「失敗」した際に実行される部分
                    }
                },
                receiveValue: { [weak self] hashableObjects in
                    // 👉 記事データ一覧が取得できた場合は仲介役となる変数へ反映する
                    self?._articles = hashableObjects
                }
            )
            .store(in: &cancellables)
    }
}
```

__【🌷記事一覧を取得する「ArticleViewModel.swift」を利用した画面での処理抜粋】__

```swift
// ----------
// 📝 ArticlesViewController.swift（処理抜粋）
// 👉 記事データの一覧を取得して画面に表示させる想定のもの
// ----------

// ① ViewModel内に定義した処理を実行する
override func viewWillAppear(_ animated: Bool) {
    super.viewWillAppear(animated)

    // 👉 ArticleViewModel内に定義した記事データの一覧取得処理を実行する 
    viewModel.inputs.fetchArticlesTrigger.send()
}

// ② ViewModel内の処理結果として受け取った値を反映する
override func viewDidLoad() {
    super.viewDidLoad()

    // 👉 ArticleViewModel内に定義した`AnyPublisher<[Article], Never>`の値が変化したらその値が流される 
    viewModel.outputs.articles
        .subscribe(on: RunLoop.main)
        .sink(
            receiveValue: { [weak self] articles in
                // TODO: 受け取った記事データの一覧を画面表示へ反映する 
                // ※ UITableViewやUICollectionViewを利用した一覧表示処理等を実行する 
            }
        )
        .store(in: &cancellables)
}
```

__【🌷補足（RxSwiftを利用した場合との比較）】__

RxSwiftを利用した処理に馴染みのある方であれば、Combineを利用した処理を記載している部分を下記の様な形に置き換えて考えるとイメージがつきやすいかと思います。私自身もCombineに初めて触れた際は、なかなかPropertyWrapperやOperatorを扱うイメージがなかなか掴めずにいましたが、下記の様なイメージを持って「置き換えながら考えていく」事で徐々にイメージを掴む事ができる様になりました。

```swift
// 1. Input: 

// (a) RxSwift利用時:
var fetchArticlesTrigger: PublishSubject<Void> { get }
// (b) Combine利用時:
var fetchArticlesTrigger: PassthroughSubject<Void, Never> { get }

// 2. InputとOutputの仲介

// (a) RxSwift利用時: 
private let _articles: BehaviorRelay<[Article]> = BehaviorRelay<[Article]>(value: [])
// (b) Combine利用時:
@Published private var _articles: [Article] = []

// 3. Output

// (a) RxSwift利用時:
var articles: Observable<[Article]> {
    return _articles.asObservable()
}
// (b) Combine利用時: 
var articles: AnyPublisher<[Article], Never> {
    return $_articles.eraseToAnyPublisher()
}
```

#### ⭐️2-2. 

```swift
// ----------
// 📝 NewsViewModel.swift
// 👉 お知らせデータの一覧を取得して画面に表示させる想定のもの
// ----------
```

### 3. 値変化を基準としたCombineベースの処理におけるUnitTestへの工夫

- __CombineExpectations__
  - GitHub: https://github.com/groue/CombineExpectations

#### ⭐️3-1. 

#### ⭐️3-2. 

#### ⭐️3-3. 

```swift

```

#### ⭐️3-4. 

### 余談. その他UnitTestやCombineに関する補足事項

この部分では、本稿における本題とは少し離れてしまいますが、小さな余談としてCombineを利用した処理で活用する機会もあるOperatorに関連する豆知識や、UnitTestのコードを記述していく中で適切なタイミングで有効活用ができそうなTipsをご紹介できればと思います。

本稿でご紹介している「Combine + Quick + Nimble + CombineExpectations」の組み合わせを利用したUnitTestに関連する事例では触れておりませんが、Combineで実装されたコードを読み解いていく際やUnitTestを記述する際の参考になれば幸いです。

#### ⭐️その1. Publisher.zipに関する小ネタ

以前にSingle.zipを利用した処理を作成した時に、RxSwiftにおいて、1つのSingle.zip(Obsevable.zip)で何個のObservableを接続できるかが気になった事があり、改めて内部実装を調べると1つのSingle.zip(Obsevable.zip)で処理を並列に取り扱う事ができる最大値は8つでした。では、Combineで似た様な振る舞いをするPublisher.zipではどうなるか？を調べてみると、最大値は4つでした。1つの画面で複数のAPI非同期通信処理結果を表示する様な場合で用いる際に利用する局面はあると思いますが、この様にRxSwiftと見比べをしてみると、同じ様な振る舞いをするOperatorであったとしても、ちょっとした様で大きな違いがあるなと、私自身がCombineに触れた際に感じた次第です。

- Publishers.Zip4に関するApple公式ドキュメント
  👉 https://developer.apple.com/documentation/combine/publishers/zip4

__【Publisher.zipとRxSwiftのコード抜粋例】__

```swift
// ----------
// MEMO: CombineのPublisher.zipは最大4つまでPublisherを同時接続可能
// ※ CombineLatestは最大4つ / Mergeは最大8つまで可能
// ----------
struct Zip4<A, B, C, D> where A : Publisher, B : Publisher, C : Publisher, D : Publisher, 
A.Failure == B.Failure, B.Failure == C.Failure, C.Failure == D.Failure

// ----------
// MEMO: (比較) RxSwiftのSingle.zipはだ最大8つまでObservableをを同時接続可能
// https://github.com/ReactiveX/RxSwift/blob/Articles/RxSwift/Observables/Zip%2Barity.swift
// ----------
public static func zip<E1, E2, E3, E4, E5, E6, E7, E8>(
    _ source1: PrimitiveSequence<Trait, E1>, 
    _ source2: PrimitiveSequence<Trait, E2>, 
    _ source3: PrimitiveSequence<Trait, E3>, 
    _ source4: PrimitiveSequence<Trait, E4>, 
    _ source5: PrimitiveSequence<Trait, E5>, 
    _ source6: PrimitiveSequence<Trait, E6>,
    _ source7: PrimitiveSequence<Trait, E7>, 
    _ source8: PrimitiveSequence<Trait, E8>, 
    resultSelector: @escaping (E1, E2, E3, E4, E5, E6, E7, E8) throws -> Element) -> PrimitiveSequence<Trait, Element> {
    return PrimitiveSequence(
        raw: Observable.zip(
            source1.asObservable(), 
            source2.asObservable(), 
            source3.asObservable(), 
            source4.asObservable(), 
            source5.asObservable(), 
            source6.asObservable(), 
            source7.asObservable(), 
            source8.asObservable(), 
            resultSelector: resultSelector
        )
    )
}
```

また、これまでRxSwiftを利用した開発経験をお持ちの方がCombineの実装へ触れていく際には、Combineの基本事項に加えて下記リンクで紹介している様なチートシート等を活用しながら理解を進めていくのも、有効な手段ではないかと思います。必ずしも"RxSwiftのノリで読み進めていく事ができる訳ではありません"が、類似点や相違点の概要について効率よく知る際の一助となる場合も多いのではないかと思います。

- RxSwiftとCombineを比較した際にできること＆できないことをまとめたチートシート
  👉 https://github.com/CombineCommunity/rxswift-to-combine-cheatsheet

#### ⭐️その2. Quickを利用したUnitTestでit部分を共通化するTips

（※文章の構成が必要）

### まとめ

（※文章の構成が必要）

今回紹介した様な形のUnitTestの事例では、とりわけサーバーサイド側での処理が主役となる様なiOSアプリにおいては特に考えやすい形ではないかと考えております。

また、本稿ではCombineを利用したUnitTestを説明する際に「CombineExpectations」を活用したコードの事例を中心に解説を進めて参りましたが、

- combine-schedulers
  👉 https://github.com/pointfreeco/combine-schedulers

### 御礼

本稿の執筆に当たりましては、現在モバイルアプリ開発に携わらせて頂いている職場の方々には、この場をお借りして感謝の意を述べさせて頂きます。これまではRxSwift+UIKitを利用した開発の中で、仕様把握や機能担保のためにUnitTestを活用したり、実際に経験はありましたが、実際にCombineを有効活用したアーキテクチャに触れる機会やCombineを利用したUnitTestを事例にも数多く触れる事ができる機会を得られた事は、私にとって本当に貴重な経験でした。

そして、私自身SwiftUIを業務内で利用した経験がそれ程ありませんでしたが、チームの皆様から業務キャッチアップの為のソースコードリーディングを通じた仕様理解の機会をはじめとして、とても丁寧かつ根気よくご指導をして頂きましたお陰もあり、今日まで来る事ができたと深く感じる次第であります。

最後までお読み頂きまして本当にありがとうございました。

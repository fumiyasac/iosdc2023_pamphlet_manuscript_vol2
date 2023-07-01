## UIKit＆SwiftUIとCombineを組み合わせた処理で上手にUnitTestを整えていくアイデア解説

<p align="right">
<strong>酒井文也 (Fumiya Sakai) Twitter &amp; github: @fumiyasac</strong>
</p>

<hr>

### はじめに

アプリ開発の中で、画面要素と内部ロジックとを結合する処理等でCombineを利用する場合もあると思います。iOS13以降で登場したApple純正のCombineの登場によって、これまで取り扱いが難しかった、APIリクエストを伴う非同期処理の取り扱いを、ネストされたクロージャーを利用した処理や、RxSwift・ReactiveSwift等の様な外部OSSを利用せずとも処理を記述する事が可能になりました。

それだけに留まらず、SwiftUIを利用した場合では __配置しているSwiftUIで作成された画面要素とのデータの双方向のBinding処理を実現する__ 際はもちろん、UIKitを利用した画面の場合においても、 __Presentation層(ViewModel層)で何らかの処理を要求した場合に対応する画面要素側での処理と連動する様な関係を実現する__ 際においても、とても有効かつ強力なパートナーになり得るのではないかと思います。

この様な処理を実現したい場合はもとより、Combine+SwiftUI（場合によってはUIKit）の組み合わせをより強力かつ上手に活用していきたい場合においては、Combineそのものに関する理解を深める事に加えて、仕様把握や機能担保の観点において __「Combineを利用した場合におけるUnitTestを活用方法」__ に関して理解を深めていく事は、今後大きな意義を持つと考えております。

本稿では、主に __Presentation層(ViewModel層)と画面ないしはView要素間における連結処理部分__ を題材として __Combineを活用していく際における処理のポイントとなり得る部分__ のご紹介や __Combineを利用する際のUnitTestを記述する際のアイデアと事例__ を、コードを通じて簡単に解説できればと考えております。

対象の読者としましては __Combineを利用した処理を実際にどの様な形で活用しているかを知りたい方__  や __Combineを利用した処理には触れた経験はあるが実際のUnitTestの記述例を知りたい方__ 等を想定しております。決して特別な事はありませんが、ほんの少しでも皆様のご参考になる事ができれば嬉しく思います。

### 1. MVVMアーキテクチャでCombineを活用する際の基本イメージ

本稿では、MVVMアーキテクチャ（Model-ViewModel-View構成）を取る様な画面処理部分に対して、まずは想定している処理イメージをまずは簡単に整理していくことにします。私自身もこれまでの実務経験の中でRxSwiftを活用したコードに触れる機会が数多くありましたので、MVVMアーキテクチャをRxSwiftを利用する場合と、Combineを利用した場合とで簡単に比較してみる事にします。

こちらで例示している例は、比較的簡単なものではありますが、実装の方針次第では、ベースとなる部分においては類似した考え方ができる余地は大いにあるかと思います。

（ここに図解が入ります）

### 2. `@Published`・`PassthroughSubject`・`AnyPublisher`を利用した処理で画面要素と内部ロジック間を結合する処理のポイント

ここからは、ViewModelクラス側の構造や想定している処理機構に注目しながらポイントとなり得そうな部分についてコードを交えながら解説ができればと思います。

#### ⭐️2-1. UIKit利用時でのCombineを利用する処理イメージ

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

__【🌷UIKitでの画面構築を想定した記事一覧を取得する「ArticleViewModel.swift」の構築例】__

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
        .subscribe(on: DispatchQueue.main)
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

#### ⭐️2-2. SwiftUI利用時でのCombineを利用する処理イメージ

SwiftUIを利用した画面におけるViewModelクラス処理においてCombineをベースとした実装を考えていく場合の一例で、ここでは __@Pubishedで定義した変数をSwiftUIのView要素で直接利用できる様な形__ を考えてみます。

前提としてViewModelクラスについては、ObservableObjectを継承したクラスとし、Outputについては __@Pubilshedとして定義した変数 or この値を利用したComputed Propertyで定義した変数__ を定義する形で、 __表示対象のView要素がこの変数の値変化に伴って表示状態が変化する形を実現する実装__ とする点がポイントとなると個人的に考えております。

下記の様な形でViewModelクラス内部で展開する、API非同期通信処理やデータ永続化処理を組み合わせる事で実現する様にすると良さそうに思います。

- __ViewModelクラスでのポイント:__ ObservableObjectクラスを継承する
- __Input:__ 画面からViewModel内で定義した処理を実行する
  👉 __ViewModel内で定義したメソッドを実行する__
- __Output:__ ViewModel内に定義した変数の値変化を受け取る
  👉 __@Pubilshedとして定義した変数 or この値を利用したComputed Propertyで定義した変数__

UIKitで利用時に比べると、SwiftUIで実装された画面要素とViewModel内に定義したOutputとしての変数をいかに上手に接続するかという部分に難しさを感じる部分はあるものの、View要素側のコードに関してはSwiftUIの恩恵を受けられることもあるので、実現する画面次第ではとてもシンプルな形にまとめられる余地は大いにあると感じた次第です。

__【🌷SwiftUIでの画面構築を想定した記事一覧を取得する「NewsViewModel.swift」の構築例】__

```swift
// ----------
// 📝 NewsViewModel.swift
// 👉 お知らせデータの一覧を取得して画面に表示させる想定のもの
// ----------
final class NewsViewModel: ObservableObject {

    private let api: APIRequestManagerProtocol

    private var cancellables: [AnyCancellable] = []

    // MARK: - NewsViewModelOutputs

    // 👉 `@Published`で定義した変数をそのままOutputとして利用する
    // (1) SwiftUI製のView要素に直接Bindして利用する場合もあります。
    // (2) アクセス修飾子はBindするSwiftUI要素に合わせて変更する場合もあります。 
    // ※要素によってはこの値を利用した「Computed Property」を別途定義して利用する場合もあります。
    @Published private(set) var newsItem: [NewsItem] = []

    // MARK: - Initializer

    init(api: APIRequestManagerProtocol) {

        // MEMO: 適用するAPIリクエスト用の処理
        self.api = api
    }

    // MARK: - NewsViewModelInputs

    // MEMO: APIリクエストを実行してお知らせデータ一覧の取得をするメソッド
    // 👉 お知らせデータ一覧が取得できた場合はSwiftUI側でも利用する`@Published`で定義した変数へ反映する
    func fetchNewsItems() {
        api.getNewsItems()
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
                    // 👉 お知らせデータ一覧が取得できた場合はSwiftUI側でも利用する`@Published`で定義した変数へ反映する
                    self?.newsItem = hashableObjects
                }
            )
            .store(in: &cancellables)
    }    
}
```

__【🌷お知らせ一覧を取得する「NewsViewModel.swift」を利用した画面での処理抜粋】__


```swift
// ----------
// 📝 NewsListView.swift
// 👉 お知らせデータの一覧を取得して画面に表示させる想定のもの
// ----------

// 🌾 画面要素全体を構成するView要素におけるbody内の処理を抜粋したものになります。
var body: some View {
    // 👉 お知らせデータの一覧をSwiftUIのList要素を利用して表示する想定です。
    // ※ViewModelの「Output」として定義した変数の状態に合わせて表示内容が変化する
    List {
        // (1) お知らせデータの一覧が0件の場合:
        if viewModel.newsItems.isEmpty {
            // (例) 表示前の「読み込み中...」の様な表示をするためのView 
            LoadingView()
        // (2) お知らせデータの一覧が少なくとも1件以上の場合:
        } else {
           // 👉 お知らせデータの一覧をSwiftUIのList要素を利用して
            ForEach(Array(viewModel.newsItems.enumerated())), id: \.offset) { index, newsItem in
                // (例) 1件分のCell要素として内容を表示するためのView 
                NewsListRowCell(newsItem: newsItem)
            }
        }
    } 
    .onAppear {
        // 👉 NewsViewModel内に定義した記事データの一覧取得処理を実行する 
        // ※ViewModelの「Input」に相当する部分と考えてOK
        viewModel.fetchNewsItems()
    }
    .listStyle(PlainListStyle())
    .edgesIgnoringSafeArea(.bottom)
}
```

### 3. 値変化を基準としたCombineベースの処理におけるUnitTestへの工夫

ここからは、Combineを活用したPresentation層(ViewModel層)のクラス内での処理において、UnitTestを記述していく際のアイデアをご紹介します。私の場合は業務や個人開発の中でも「Quick + Nimble」の組み合わせを利用する場合が多かったのですが、本稿ではそこに加えてCombineでのUnitTestを便利にしてくれる「CombineExpectations」というOSSを更に組み合わせた形考えていくことにします。

- __CombineExpectations__
  - GitHub: https://github.com/groue/CombineExpectations

CombineExpectationsを利用したUnitTestを考えていく際における重要なポイントとしては、 __記録対象とした変数の変化が全て記録されている（すなわち、変数のBefore → Afterを追いかける事が可能である）__ にあると思います。この性質を有効活用することによって、

- 特定の画面表示に関連するViewModel内部に定義した変数の状態変化を把握する
- 内部実装の工夫次第ではユーザーからの値入力に伴って変化する値等の変化の検知にも応用可能である

といった、細かな部分に対する仕様把握や理解にも結果的には繋がること多かった様に感じる経験も少なからずありました。

#### ⭐️3-1. CombineExpectationsを簡単な使い方とUnitTest例

ここでは、CombineExpectationsを利用した際の簡単な記述と使い方のコード事例を2点程ご紹介しています。

__【🌷例1: PassthroughSubjectに送信した値をテストする簡単な事例】__

こちらは、CombineExpectationsで提供されている機能を利用した、簡単な値変化をテストするためのコード例となります。記録対象の変数に対する、値変化のBefore → Afterの記録を取り出した後に値変化の流れが正しい事をUnitTestを用いて確認する様な場面が多いため、初期状態から一連の処理後までの値変化を比較する様なイメージとなります。

（初期値〜任意の処理実行後までの一連の値変化の結果については __availableElements__ から取得する事ができます。）

```swift
final class SimplePassthroughSubjectTests: XCTestCase {

    func test_PassthroughSubjectValueChange() throws {

        // 準備: 空のPassthroughSubjectを準備し変遷を記録可能にする
        let dessertNamePublisher = PassthroughSubject<String, Never>()
        let recorder = dessertNamePublisher.record()
        // その1: 1回目の内容を送る → `.next().get()`でPassthroughSubjectの最新を取得する
        dessertNamePublisher.send("🍪:チョコチップクッキー")
        try XCTAssertEqual(recorder.next().get(), "🍪:チョコチップクッキー")
        // その2: 2回目の内容を送る → `.next().get()`でPassthroughSubjectの最新を取得する
        dessertNamePublisher.send("🍦:ソフトクリーム")
        try XCTAssertEqual(recorder.next().get(), "🍦:ソフトクリーム")
        // その3: `availableElements`を下記の様な形で現在時点での「dessertNamePublisher」の変化全てを取得する
        let result = try! self.wait(for: recorder.availableElements, timeout: 0.16)
        XCTAssertEqual(result, ["🍪:チョコチップクッキー", "🍦:ソフトクリーム"])
    }
}
```

__【🌷例2: @Publishedで定義された値変化をテストする簡単な事例】__

こちらは、ObservableObjectを継承したクラス内に定義した@Publishedとして定義された変数の値変化をテストする事例になります。特に、前述したSwiftUIを利用した画面とViewModelクラスに定義された@Publishedで提供されている値変化と連動する様な方針を取る際においても、こちらのコード例を応用したものと捉えて頂くと、よりUnitTestを考えていく際のイメージが明確になるのではないかと思います。

```swift
final class SimpleObservableObjectTests: XCTestCase {

    func test_ObservableObjectValueChange() throws {

        // 準備: EatenMealStatus内に定義した`@Published`で定義した変数の変遷を記録可能にする
        let eatenMealStatus = EatenMealStatus()
        let recorder = eatenMealStatus.$currentValue.record()
        // 処理実行: EatenMealStatus内に定義した`@Published`で定義した変数の変遷を記録可能にする
        eatenMealStatus.update(inputValue: "🥪:BLTサンド")
        eatenMealStatus.update(inputValue: "🍜:味噌ラーメン")
        // `availableElements`を下記の様な形で現在時点での「eatenMealStatus.$currentValue」の変化全てを取得する
        let recorderResult = try! self.wait(for: recorder.availableElements, timeout: 0.16)
        XCTAssertEqual(recorderResult, ["", "🥪:BLTサンド", "🍜:味噌ラーメン"])
    }

    // MEMO: テスト対象のObservableObject継承クラス
    final class EatenMealStatus: ObservableObject {

        // Output: @Publishedで定義した変数（初期値は空文字）
        @Published private(set) var currentValue: String = ""

        // Input: クラス内のプロパティを変化させるためのメソッド
        func update(inputValue: String) {
            currentValue = inputValue
        }
    }
}
```

#### ⭐️3-2. 

ここからは具体的な事例として、前述した __「⭐️2-2. SwiftUI利用時でのCombineを利用する処理イメージ」__ で紹介したViewModelクラスのテストコード記述例を考えてみます。後述するコードについてはAPI非同期通信処理における成功時のみの抜粋となりますが、実務でのコードでは失敗した場合等の処理についても考慮をした方が良いかと思います。他にも、コメント等の入力処理等をはじめとした、ユーザーからの入力に基づいてViewModel内部の値が変化する様な処理（例. バリデーション処理）等を考えていくUnitTest等でも応用または活用が可能です。

```swift
// ----------
// 📝 APIRequestManagerMock.swift
// 👉 UnitTest利用するお知らせデータの一覧を取得する処理をMock化したもの
// ----------
protocol APIRequestManagerProtocol {
    func getNewsItems() -> AnyPublisher<[NewsItem], Never>
}

final class APIRequestManagerMock: APIRequestManagerProtocol {

    // 👉 API非同期通信処理成功時を想定した返り値を設定する
    func getNewsItems() -> AnyPublisher<[NewsItem], Never> {
        let newsItems = [
            NewsItem(id: 1, title: "夏祭り2023開催のお知らせ", category: "イベント"),
            NewsItem(id: 2, title: "美味しい餃子の販売開始", category: "新商品"),
            NewsItem(id: 3, title: "美味しいプリンの販売開始", category: "新商品")
        ]
        return Just(newsItems)
            .eraseToAnyPublisher()
    }
}

// ----------
// 📝 NewsViewModelTests.swift
// 👉 お知らせデータの一覧を取得して画面に表示させる処理をするViewModel処理のUnitTest
// ----------
final class NewsViewModelTest: QuickSpec {

    // MARK: - Override

    override func spec() {

        // MEMO: Quick + NimbleをベースにしたUnitTestを実行する
        describe("#fetchNewsItems") {

            // 期待する初期値 & 実行後に期待する値を定義する
            let emptyItems: [NewsItem] = []
            let expectedNextItems: [NewsItem] = [
                NewsItem(id: 1, title: "夏祭り2023開催のお知らせ", category: "イベント"),
                NewsItem(id: 2, title: "美味しい餃子の販売開始", category: "新商品"),
                NewsItem(id: 3, title: "美味しいプリンの販売開始", category: "新商品")
            ]

            // 👉 NewsViewModelをインスタンス化する際に、想定するAPIリクエストのMockを適用する
            // ※補足: func getNewsItems() -> AnyPublisher<[NewsItem], Never> を想定
            let sut = NewsViewModel(api: APIRequestManagerMock())

            // CombineExpectationを利用してViewModel内の変数:newsItemの変化を記録するようにしたい
            // 👉 変数:newsItemで`@Published`を利用しているのでこの値変化を記録対象とする
            var newsItemsRecorder: Recorder<[NewsItem], Never>!
            context("API通信処理が成功した場合") {
                // 👉 UnitTest実行前後で実行する処理
                beforeEach {
                    newsItemsRecorder = sut.$newsItem.record()
                }
                afterEach {
                    newsItemsRecorder = nil
                }
                // 値変化を記録対象とする変数が、期待した変化をすること確認する
                it("初期値:emptyItemsの内容 → 実行後:expectedNextItems となること") {
                    // 👉 ViewModel内処理を実行
                    sut.fetchNewsItems()
                    // 0.16秒間の変化を見て、期待した値変化となることを確認する
                    let newsItemsRecorderResult = try! self.wait(for: newsItemsRecorder.availableElements, timeout: 0.16)
                    expect(newsItemsRecorderResult).to(equal([emptyItems, expectedNextItems]))
                }
            }
        }
    }
}
```

### まとめ

（※文章の構成が必要）

__※余談その1:__

参考資料の1つとして、RxSwiftを活用した場合のUnitTestに関する解説については、昨年に開催されたiOSDC2022のパンフレット原稿でも簡単ではありますが触れていますので、ご興味がある方は、RxSwiftとCombineを見比べた際における類似点や相違点を見比べながら一読頂けますと幸いに思います。

- __不具合や仕様もれを減らすための勘所とユニットテストで学ぶ簡単事例集__
  👉 https://github.com/fumiyasac/iosdc2022_pamphlet_manuscript/blob/main/manuscript.md

__※余談その2:__

本稿ではCombineを利用したUnitTestを説明する際に「CombineExpectations」を活用したコードの事例を中心に解説を進めて参りましたが、他のCombineを利用した実装でのUnitTestを考えていく際には、The Composable Architectureを開発しているPoint-FreeがOSSとして公開している下記の様なOSSを活用する手段もあります。

- __combine-schedulers__
  👉 https://github.com/pointfreeco/combine-schedulers

### 御礼

本稿の執筆に当たりましては、現在モバイルアプリ開発に携わらせて頂いている職場の方々には、この場をお借りして感謝の意を述べさせて頂きます。これまではRxSwift+UIKitを利用した開発の中で、仕様把握や機能担保のためにUnitTestを活用したり、実際に経験はありましたが、実際にCombineを有効活用したアーキテクチャに触れる機会やCombineを利用したUnitTestを事例にも数多く触れる事ができる機会を得られた事は、私にとって本当に貴重な経験でした。

そして、私自身SwiftUIを業務内で利用した経験がそれ程ありませんでしたが、チームの皆様から業務キャッチアップの為のソースコードリーディングを通じた仕様理解の機会をはじめとして、とても丁寧かつ根気よくご指導をして頂きましたお陰もあり、今日まで来る事ができたと深く感じる次第であります。

最後までお読み頂きまして本当にありがとうございました。
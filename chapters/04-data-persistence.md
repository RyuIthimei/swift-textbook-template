# 第4章：データの永続化

> 執筆者：劉 一鳴
> 最終更新：2026-06-10

## この章で学ぶこと

（この章で扱うトピックの概要を2〜3行で書く。自分の言葉で。）

例：この章では、AppStorageとSwiftDataを使ってアプリのデータを端末に永続的に保存する方法を学ぶ。具体的にはSwiftDataを使ったメモアプリを題材として、@Modelでデータモデルを定義し、modelContextを使ったデータ操作、@Queryによる動的なデータ取得、そして@AppStorageによるユーザー設定の保存を実装する。

## 模範コードの全体像

（教員から配布された模範コードをここに貼り付ける）

```swift
// ============================================
// 第4章：データの永続化（AppStorage + SwiftData）
// ============================================
// シンプルなメモアプリで、2つの永続化方法を学びます。
// - AppStorage：アプリ設定の保存
// - SwiftData：構造化データの保存
// ============================================

import SwiftUI
import SwiftData

// MARK: - SwiftDataモデル

@Model
class Memo {
    var title: String
    var content: String
    var createdAt: Date
    var isFavorite: Bool

    init(title: String, content: String, createdAt: Date = .now, isFavorite: Bool = false) {
        self.title = title
        self.content = content
        self.createdAt = createdAt
        self.isFavorite = isFavorite
    }
}

// MARK: - アプリのエントリポイント
// ※ @main のあるAppファイルに以下を記述してください：
//
// @main
// struct MemoApp: App {
//     var body: some Scene {
//         WindowGroup {
//             ContentView()
//         }
//         .modelContainer(for: Memo.self)
//     }
// }

// MARK: - メインビュー

struct ContentView: View {
    @Environment(\.modelContext) private var modelContext
    @Query(sort: \Memo.createdAt, order: .reverse) private var memos: [Memo]
    @AppStorage("sortByFavorite") private var sortByFavorite: Bool = false
    @AppStorage("userName") private var userName: String = ""
    @State private var isShowingAddSheet = false
    @State private var isShowingSettings = false

    var displayedMemos: [Memo] {
        if sortByFavorite {
            return memos.sorted { $0.isFavorite && !$1.isFavorite }
        }
        return memos
    }

    var body: some View {
        NavigationStack {
            Group {
                if memos.isEmpty {
                    ContentUnavailableView(
                        "メモがありません",
                        systemImage: "note.text",
                        description: Text("右上の＋ボタンからメモを追加してください")
                    )
                } else {
                    List {
                        ForEach(displayedMemos) { memo in
                            NavigationLink(destination: MemoEditView(memo: memo)) {
                                MemoRow(memo: memo)
                            }
                        }
                        .onDelete(perform: deleteMemos)
                    }
                }
            }
            .navigationTitle(userName.isEmpty ? "メモ帳" : "\(userName)のメモ帳")
            .toolbar {
                ToolbarItem(placement: .topBarLeading) {
                    Button {
                        isShowingSettings = true
                    } label: {
                        Image(systemName: "gear")
                    }
                }
                ToolbarItem(placement: .topBarTrailing) {
                    Button {
                        isShowingAddSheet = true
                    } label: {
                        Image(systemName: "plus")
                    }
                }
            }
            .sheet(isPresented: $isShowingAddSheet) {
                MemoAddView()
            }
            .sheet(isPresented: $isShowingSettings) {
                SettingsView(userName: $userName, sortByFavorite: $sortByFavorite)
            }
        }
    }

    func deleteMemos(at offsets: IndexSet) {
        for index in offsets {
            let memo = displayedMemos[index]
            modelContext.delete(memo)
        }
    }
}

// MARK: - メモの行

struct MemoRow: View {
    let memo: Memo

    var body: some View {
        HStack {
            VStack(alignment: .leading, spacing: 4) {
                Text(memo.title)
                    .font(.headline)

                Text(memo.content)
                    .font(.caption)
                    .foregroundStyle(.secondary)
                    .lineLimit(2)

                Text(memo.createdAt, style: .date)
                    .font(.caption2)
                    .foregroundStyle(.tertiary)
            }

            Spacer()

            if memo.isFavorite {
                Image(systemName: "star.fill")
                    .foregroundStyle(.yellow)
            }
        }
        .padding(.vertical, 2)
    }
}

// MARK: - メモ追加画面

struct MemoAddView: View {
    @Environment(\.modelContext) private var modelContext
    @Environment(\.dismiss) private var dismiss
    @State private var title = ""
    @State private var content = ""

    var body: some View {
        NavigationStack {
            Form {
                Section("タイトル") {
                    TextField("メモのタイトル", text: $title)
                }
                Section("内容") {
                    TextEditor(text: $content)
                        .frame(minHeight: 200)
                }
            }
            .navigationTitle("新しいメモ")
            .navigationBarTitleDisplayMode(.inline)
            .toolbar {
                ToolbarItem(placement: .cancellationAction) {
                    Button("キャンセル") { dismiss() }
                }
                ToolbarItem(placement: .confirmationAction) {
                    Button("保存") {
                        let memo = Memo(title: title, content: content)
                        modelContext.insert(memo)
                        dismiss()
                    }
                    .disabled(title.isEmpty)
                }
            }
        }
    }
}

// MARK: - メモ編集画面

struct MemoEditView: View {
    @Bindable var memo: Memo

    var body: some View {
        Form {
            Section("タイトル") {
                TextField("タイトル", text: $memo.title)
            }
            Section("内容") {
                TextEditor(text: $memo.content)
                    .frame(minHeight: 200)
            }
            Section {
                Toggle("お気に入り", isOn: $memo.isFavorite)
            }
        }
        .navigationTitle("メモを編集")
        .navigationBarTitleDisplayMode(.inline)
    }
}

// MARK: - 設定画面（AppStorageの活用）

struct SettingsView: View {
    @Binding var userName: String
    @Binding var sortByFavorite: Bool
    @Environment(\.dismiss) private var dismiss

    var body: some View {
        NavigationStack {
            Form {
                Section("ユーザー設定") {
                    TextField("あなたの名前", text: $userName)
                }
                Section("表示設定") {
                    Toggle("お気に入りを上に表示", isOn: $sortByFavorite)
                }
                Section {
                    Text("設定はアプリを閉じても保存されます")
                        .font(.caption)
                        .foregroundStyle(.secondary)
                }
            }
            .navigationTitle("設定")
            .navigationBarTitleDisplayMode(.inline)
            .toolbar {
                ToolbarItem(placement: .confirmationAction) {
                    Button("完了") { dismiss() }
                }
            }
        }
    }
}

#Preview {
    ContentView()
        .modelContainer(for: Memo.self, inMemory: true)
}

```

**このアプリは何をするものか：**

（アプリの動作を自分の言葉で説明する。スクリーンショットを貼ってもよい。）

このアプリは、メモを作成・編集・削除できるシンプルなメモアプリである。

メモはアプリを閉じても保存され、再起動しても残る。

また、ユーザー名や表示設定も保存される。

## コードの詳細解説

### SwiftDataモデル（@Model）

```swift
@Model
class Memo {
    var title: String
    var content: String
    var createdAt: Date
    var isFavorite: Bool

    init(title: String, content: String, createdAt: Date = .now, isFavorite: Bool = false) {
        self.title = title
        self.content = content
        self.createdAt = createdAt
        self.isFavorite = isFavorite
    }
}
```

**何をしているか：**
（この部分が果たしている役割を説明する）

メモのデータの形を定義している部分である。
タイトル・内容・作成日時・お気に入りかどうかを1つのデータとしてまとめている。

**なぜこう書くのか：**
（別の書き方ではなく、この書き方が選ばれている理由を説明する）

SwiftData でデータを保存するためには、このように @Model を付けて「管理できるデータ」にする必要がある。

これにより、保存や取得、更新が自動でできるようになる。

**もしこう書かなかったら：**
（この部分を省略したり変えたりすると何が起きるか。実際に試した結果があればここに書く）

@Model を付けないと、SwiftData で保存できないため、アプリを閉じるとデータが消えてしまう。

また、データの変更も自動で追跡されないので、一覧画面の更新も反映されない。

---

### データの追加・削除（modelContext）

```swift
 ToolbarItem(placement: .confirmationAction) {
                    Button("保存") {
                        let memo = Memo(title: title, content: content)
                        modelContext.insert(memo)
                        dismiss()
                    }
                    .disabled(title.isEmpty)
                }


func deleteMemos(at offsets: IndexSet) {
        for index in offsets {
            let memo = displayedMemos[index]
            modelContext.delete(memo)
        }
    }
```

**何をしているか：**

メモの追加と削除を行っている部分である。「保存」ボタンを押すと新しいメモを作成して追加し、スワイプなどで選んだメモを削除している。

**なぜこう書くのか：**

SwiftData では modelContext を使うことで、データの追加や削除をまとめて管理できる。

保存処理や更新処理を自分で書かなくても、変更が自動で反映されるようになっている。

**もしこう書かなかったら：**

modelContext を使わないと、メモの追加や削除ができず、画面の内容も更新されない。

また、アプリを再起動すると変更した内容が保存されず、すべて元に戻ってしまう。

---

### @Queryによるデータ取得

```swift
 @Query(sort: \Memo.createdAt, order: .reverse) private var memos: [Memo]
```

**何をしているか：**

保存されているメモを自動で取得して一覧として表示している部分である。

また、作成日時の新しい順に並べて表示するようになっている。

**なぜこう書くのか：**

@Query を使うことで、データの変更を自動で監視し、画面が自動的に更新されるようになる。

自分で配列を更新しなくても、SwiftData が最新のデータを返してくれる。

**もしこう書かなかったら：**

@Query を使わずに自分で配列を管理すると、データが変わったときに画面が更新されない。

また、削除や追加のたびに自分でリロード処理を書く必要がある。

---

### @AppStorageによる設定保存

```swift
    @AppStorage("sortByFavorite") private var sortByFavorite: Bool = false
    @AppStorage("userName") private var userName: String = ""
```

**何をしているか：**

ユーザーの設定（名前や表示方法）を保存している部分である。
アプリを閉じても値が残るようになっている。

**なぜこう書くのか：**

@AppStorage を使うことで、簡単に少量の設定データを保存できる。

SwiftData のような複雑な仕組みを使わなくても、自動で端末に保存される。

**もしこう書かなかったら：**

@AppStorage を使わない場合、自分で UserDefaults などを管理する必要がある。

その場合はコードが長くなり、設定の保存や読み込みが面倒になる。

---

（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| 例：`@Model` | SwiftDataでオブジェクトを永続化するためのマクロ | `@Model final class Memo { ... }` |
| 例：`@Query` | データベースからデータを取得し、変更を自動で反映するプロパティラッパー | `@Query var memos: [Memo]` |
| `@AppStorage` | 小さな設定データを端末に自動保存する | @AppStorage("userName") var userName: String |
| `@Bindable` | SwiftDataのデータを画面上で直接編集できるようにする | @Bindable var memo: Memo |
| .modelContainer(for:) | SwiftDataをアプリ全体で使えるようにする設定 | .modelContainer(for: Memo.self) |
| modelContext | データの追加・削除などの操作を行うための環境変数 | modelContext.insert(memo) |

## 自分の実験メモ

（模範コードを改変して試したことを書く）

**実験1：**
- やったこと：メモをいくつか追加したあと、アプリを完全に終了して再起動した。
- 結果：追加したメモが消えずにそのまま残っていた。
- わかったこと：SwiftData はデータを端末に保存するため、アプリを閉じても内容が保持される。

**実験2：**
- やったこと：ユーザー名を変更してアプリを再起動した。
- 結果：再起動してもユーザー名がそのまま残っていた。
- わかったこと：@AppStorage は小さな設定データを自動で保存する仕組みである。

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：**
   **得られた理解：**

2. **質問：**
   **得られた理解：**

3. **質問：**
   **得られた理解：**

## この章のまとめ

（この章で学んだ最も重要なことを、未来の自分が読み返したときに役立つように書く）

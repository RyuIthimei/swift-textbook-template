# 第1章：WebAPIの基本

> 執筆者：劉 一鳴
> 
> 最終更新：YYYY-MM-DD

## この章で学ぶこと

この章では、iTunes Search APIを使ってインターネットから音楽データを取得し、それをアプリ内で表示する方法を学んだ。特にJSON形式のデータをSwiftのCodableを使って構造体に変換し、Listを使って画面に表示する流れを理解した。
## 模範コードの全体像

```swift
// ============================================
// 第1章（基本）：iTunes Search APIで音楽を検索するアプリ
// ============================================
// このアプリは、iTunes Search APIを使って
// 音楽（曲）を検索し、結果をリスト表示します。
// APIキーは不要で、すぐに動かすことができます。
// ============================================

import SwiftUI

// MARK: - データモデル

struct SearchResponse: Codable {
    let results: [Song]
}

struct Song: Codable, Identifiable {
    let trackId: Int
    
    let trackName: String
    let artistName: String
    let artworkUrl100: String
    let previewUrl: String?

    var id: Int { trackId }
}

// MARK: - メインビュー

struct ContentView: View {
    @State private var songs: [Song] = []
    @State private var searchText: String = ""
    @State private var isLoading: Bool = false

    var body: some View {
        NavigationStack {
            VStack {
                // 検索バー
                HStack {
                    TextField("アーティスト名を入力", text: $searchText)
                        .textFieldStyle(.roundedBorder)

                    Button("検索") {
                        Task {
                            await searchMusic()
                        }
                    }
                    .buttonStyle(.borderedProminent)
                    .disabled(searchText.isEmpty)
                }
                .padding(.horizontal)

                // 検索結果リスト
                if isLoading {
                    ProgressView("検索中...")
                        .padding()
                    Spacer()
                } else if songs.isEmpty {
                    ContentUnavailableView(
                        "曲を検索してみよう",
                        systemImage: "music.note",
                        description: Text("アーティスト名を入力して検索ボタンを押してください")
                    )
                } else {
                    List(songs) { song in
                        SongRow(song: song)
                    }
                }
            }
            .navigationTitle("Music Search")
        }
    }

    // MARK: - API通信

    func searchMusic() async {
        guard let encodedText = searchText.addingPercentEncoding(
            withAllowedCharacters: .urlQueryAllowed
        ) else { return }

        let urlString = "https://itunes.apple.com/search?term=\(encodedText)&media=music&country=jp&limit=25"

        guard let url = URL(string: urlString) else { return }

        isLoading = true

        do {
            let (data, _) = try await URLSession.shared.data(from: url)
            let response = try JSONDecoder().decode(SearchResponse.self, from: data)
            songs = response.results
        } catch {
            print("エラー: \(error.localizedDescription)")
            songs = []
        }

        isLoading = false
    }
}

// MARK: - 曲の行ビュー

struct SongRow: View {
    let song: Song

    var body: some View {
        HStack(spacing: 12) {
            AsyncImage(url: URL(string: song.artworkUrl100)) { image in
                image
                    .resizable()
                    .aspectRatio(contentMode: .fill)
            } placeholder: {
                Color.gray.opacity(0.3)
            }
            .frame(width: 60, height: 60)
            .clipShape(RoundedRectangle(cornerRadius: 8))

            VStack(alignment: .leading, spacing: 4) {
                Text(song.trackName)
                    .font(.headline)
                    .lineLimit(1)

                Text(song.artistName)
                    .font(.subheadline)
                    .foregroundStyle(.secondary)
            }
        }
        .padding(.vertical, 4)
    }
}

#Preview {
    ContentView()
}

```

**このアプリは何をするものか：**

このアプリは、ユーザーがアーティスト名を入力すると、そのキーワードをもとに iTunes Search API にHTTPリクエストを送信し、音楽データを取得して一覧表示するアプリである。

具体的には、検索ボタンを押すと入力された文字列がAPIに渡され、サーバーからJSON形式のデータ（曲情報のリスト）が返ってくる。そのJSONをSwiftのCodableを使ってデータモデルに変換し、画面上にListとして表示する。

また、各曲データはIdentifiableに準拠しており、SwiftUIのListで各項目を正しく識別できるようになっている。各リスト項目には、曲のアートワーク画像、曲名、アーティスト名が表示される。

## コードの詳細解説

### データモデル（Codable構造体）

```swift
struct SearchResponse: Codable {
    let results: [Song]
}

struct Song: Codable, Identifiable {
    let trackId: Int
    
    let trackName: String
    let artistName: String
    let artworkUrl100: String
    let previewUrl: String?

    var id: Int { trackId }
}

```

**何をしているか：**
（この部分が果たしている役割を説明する）

この部分では、iTunes Search APIから返ってくるJSONデータの構造をSwiftで扱える形として定義している。

SearchResponseは検索結果全体を表し、その中にSongの配列が含まれている。

Songは1曲分のデータを表しており、曲名・アーティスト名・アートワークなどの情報を持っている。

また、SwiftUIのListで使用できるようにするためにIdentifiableに準拠し、trackIdをidとして使用している。

Codableを使うことで、JSONデータをSwiftの構造体へ自動的に変換できるようになっている。

**なぜこう書くのか：**

（別の書き方ではなく、この書き方が選ばれている理由を説明する）

このように書く理由は、APIから取得したJSONデータをSwiftで扱いやすくし、SwiftUIのListで正しく表示するためである。

まずCodableを使うことで、JSONデータを手動で解析せずにSwiftの構造体へ自動的に変換でき、コードを簡潔にできる。

またIdentifiableを使うことで、Listが各データを正しく識別できるようになる。

そのため、APIから取得したtrackIdをそのままidとして利用し、各曲を一意に区別できるようにしている。

**もしこう書かなかったら：**

（この部分を省略したり変えたりすると何が起きるか。実際に試した結果があればここに書く）

Codableを使わない場合、JSONデータを手動で解析する必要があり、コードが複雑になりやすい。

Identifiableを使わない場合、Listが各要素を正しく識別できず、画面更新が正しく行われない可能性がある。

またidを定義しない場合、SwiftUIのListでエラーが発生したり、各曲を区別できなくなることがある。

---

### API通信の処理

```swift
    func searchMusic() async {
        guard let encodedText = searchText.addingPercentEncoding(
            withAllowedCharacters: .urlQueryAllowed
        ) else { return }

        let urlString = "https://itunes.apple.com/search?term=\(encodedText)&media=music&country=jp&limit=25"

        guard let url = URL(string: urlString) else { return }

        isLoading = true

        do {
            let (data, _) = try await URLSession.shared.data(from: url)
            let response = try JSONDecoder().decode(SearchResponse.self, from: data)
            songs = response.results
        } catch {
            print("エラー: \(error.localizedDescription)")
            songs = []
        }

        isLoading = false
    }
}
```

**何をしているか：**

**なぜこう書くのか：**

**もしこう書かなかったら：**

---

### ビューの構成

```swift
// 該当部分のコードを抜粋して貼る
```

**何をしているか：**

**なぜこう書くのか：**

**もしこう書かなかったら：**

---

（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| 例：`Codable` | JSONデータとSwiftの構造体を相互変換するプロトコル | `struct Song: Codable { ... }` |
| 例：`async/await` | 非同期処理を同期的に書ける構文 | `let data = try await URLSession.shared.data(from: url)` |
| | | |
| | | |
| | | |

## 自分の実験メモ

（模範コードを改変して試したことを書く）

**実験1：**
- やったこと：
- 結果：
- わかったこと：

**実験2：**
- やったこと：
- 結果：
- わかったこと：

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：**
   **得られた理解：**

2. **質問：**
   **得られた理解：**

3. **質問：**
   **得られた理解：**

## この章のまとめ

（この章で学んだ最も重要なことを、未来の自分が読み返したときに役立つように書く）

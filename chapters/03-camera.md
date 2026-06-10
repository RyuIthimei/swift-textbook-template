# 第3章：カメラの利用

> 執筆者：劉 一鳴
> 最終更新：2026-05-20

## この章で学ぶこと

（この章で扱うトピックの概要を2〜3行で書く。自分の言葉で。）

例：この章では、PhotosPickerでフォトライブラリから写真を選択し、UIImagePickerControllerでカメラ撮影した画像を扱う方法を学ぶ。具体的には非同期で画像データを読み込み、UIViewControllerRepresentableを使ってUIKitをSwiftUIに統合し、Coordinatorパターンを使ってカメラ機能と連携するアプリを題材にする。

この章では、PhotosPickerを使ってフォトライブラリから画像を選択する方法と、UIImagePickerControllerを使ってカメラで撮影した画像を扱う方法を学ぶ。
また、画像データを読み込む方法や、カメラ機能をSwiftUIから利用する方法についても学ぶ。

## 模範コードの全体像

（教員から配布された模範コードをここに貼り付ける）

```swift
// ============================================
// 第3章（基本）：写真を選択・撮影して表示するアプリ
// ============================================
// PhotosPickerを使ってフォトライブラリから写真を選択し、
// 画面に表示します。シミュレータでも動作します。
// ============================================

import SwiftUI
import PhotosUI

// MARK: - メインビュー

struct ContentView: View {
    @State private var selectedItem: PhotosPickerItem?
    @State private var selectedImage: Image?
    @State private var isShowingCamera = false
    @State private var capturedUIImage: UIImage?

    var body: some View {
        NavigationStack {
            VStack(spacing: 20) {
                // 画像表示エリア
                imageDisplayArea

                // ボタンエリア
                HStack(spacing: 20) {
                    // フォトライブラリから選択
                    PhotosPicker(selection: $selectedItem, matching: .images) {
                        Label("ライブラリ", systemImage: "photo.on.rectangle")
                    }
                    .buttonStyle(.bordered)

                    // カメラで撮影
                    Button {
                        isShowingCamera = true
                    } label: {
                        Label("カメラ", systemImage: "camera")
                    }
                    .buttonStyle(.borderedProminent)
                }
                .padding()
            }
            .navigationTitle("写真アプリ")
            .onChange(of: selectedItem) { _, newItem in
                Task {
                    await loadImage(from: newItem)
                }
            }
            .fullScreenCover(isPresented: $isShowingCamera) {
                CameraView(capturedImage: $capturedUIImage)
            }
            .onChange(of: capturedUIImage) { _, newImage in
                if let uiImage = newImage {
                    selectedImage = Image(uiImage: uiImage)
                }
            }
        }
    }

    // MARK: - 画像表示エリア

    @ViewBuilder
    private var imageDisplayArea: some View {
        if let image = selectedImage {
            image
                .resizable()
                .aspectRatio(contentMode: .fit)
                .frame(maxHeight: 400)
                .clipShape(RoundedRectangle(cornerRadius: 16))
                .shadow(radius: 4)
                .padding()
        } else {
            RoundedRectangle(cornerRadius: 16)
                .fill(.gray.opacity(0.1))
                .frame(height: 300)
                .overlay {
                    VStack(spacing: 8) {
                        Image(systemName: "photo")
                            .font(.system(size: 48))
                            .foregroundStyle(.gray)
                        Text("写真を選択または撮影してください")
                            .font(.caption)
                            .foregroundStyle(.secondary)
                    }
                }
                .padding()
        }
    }

    // MARK: - 画像の読み込み

    func loadImage(from item: PhotosPickerItem?) async {
        guard let item = item else { return }

        do {
            if let data = try await item.loadTransferable(type: Data.self),
               let uiImage = UIImage(data: data) {
                selectedImage = Image(uiImage: uiImage)
            }
        } catch {
            print("画像の読み込みに失敗: \(error.localizedDescription)")
        }
    }
}

// MARK: - カメラビュー（UIKit連携）

struct CameraView: UIViewControllerRepresentable {
    @Binding var capturedImage: UIImage?
    @Environment(\.dismiss) private var dismiss

    func makeUIViewController(context: Context) -> UIImagePickerController {
        let picker = UIImagePickerController()
        picker.sourceType = .camera
        picker.delegate = context.coordinator
        return picker
    }

    func updateUIViewController(_ uiViewController: UIImagePickerController, context: Context) {}

    func makeCoordinator() -> Coordinator {
        Coordinator(self)
    }

    class Coordinator: NSObject, UIImagePickerControllerDelegate, UINavigationControllerDelegate {
        let parent: CameraView

        init(_ parent: CameraView) {
            self.parent = parent
        }

        func imagePickerController(
            _ picker: UIImagePickerController,
            didFinishPickingMediaWithInfo info: [UIImagePickerController.InfoKey: Any]
        ) {
            if let image = info[.originalImage] as? UIImage {
                parent.capturedImage = image
            }
            parent.dismiss()
        }

        func imagePickerControllerDidCancel(_ picker: UIImagePickerController) {
            parent.dismiss()
        }
    }
}

#Preview {
    ContentView()
}

```

**このアプリは何をするものか：**

（アプリの動作を自分の言葉で説明する。スクリーンショットを貼ってもよい。）

このアプリは、ユーザーがフォトライブラリから写真を選択するか、カメラで写真を撮影すると、その画像を画面に表示するアプリである。選択した画像と撮影した画像は同じ表示エリアに反映される。

## コードの詳細解説

### PhotosPickerによる写真選択

```swift
                    PhotosPicker(selection: $selectedItem, matching: .images) {
                        Label("ライブラリ", systemImage: "photo.on.rectangle")
                    }
                    .buttonStyle(.bordered)
```

**何をしているか：**
（この部分が果たしている役割を説明する）

PhotosPicker を使って、フォトライブラリから写真を選択できるようにしている。

選択した写真は selectedItem に保存される。

また、matching: .images を指定することで、画像だけを選択対象にしている。

**なぜこう書くのか：**
（別の書き方ではなく、この書き方が選ばれている理由を説明する）

PhotosPicker を使うと、SwiftUI だけで簡単に写真選択機能を実装できる。

$selectedItem を使うことで、写真を選択した時に値が自動で更新される。

**もしこう書かなかったら：**
（この部分を省略したり変えたりすると何が起きるか。実際に試した結果があればここに書く）

selection: $selectedItem がないと、選択した写真を取得できない。

matching: .images を外すと、画像だけを選択する制限がなくなる。。

---

### 画像の非同期読み込み

```swift
.onChange(of: selectedItem) { _, newItem in
                Task {
                    await loadImage(from: newItem)
                }
            }


func loadImage(from item: PhotosPickerItem?) async {
        guard let item = item else { return }

        do {
            if let data = try await item.loadTransferable(type: Data.self),
               let uiImage = UIImage(data: data) {
                selectedImage = Image(uiImage: uiImage)
            }
        } catch {
            print("画像の読み込みに失敗: \(error.localizedDescription)")
        }
    }
}

```

**何をしているか：**

写真が選択された時に、画像データを読み込んで画面に表示している。

loadTransferable() を使って画像データを取得し、UIImage に変換している。

**なぜこう書くのか：**

loadTransferable() は await が必要な処理なので、

処理が終わるのを待ってから画像を表示している。

また、そのままでは呼び出せなかったため、Task の中で実行している。

**もしこう書かなかったら：**

await を付けないとエラーになり、画像を読み込めない。

また、Task を使わずに loadImage() を呼ぼうとしても async 関数なので実行できなかった。

---

### UIViewControllerRepresentableによるカメラ連携

```swift
struct CameraView: UIViewControllerRepresentable {
    @Binding var capturedImage: UIImage?
    @Environment(\.dismiss) private var dismiss

    func makeUIViewController(context: Context) -> UIImagePickerController {
        let picker = UIImagePickerController()
        picker.sourceType = .camera
        picker.delegate = context.coordinator
        return picker
    }
```

**何をしているか：**

カメラを起動するための設定を行っている。

UIImagePickerController を使ってカメラ画面を表示し、撮影した画像をアプリで利用できるようにしている。

**なぜこう書くのか：**

SwiftUI だけではカメラを直接起動できなかったため、この方法を使っている。

サンプルコードや参考資料を調べると、カメラ機能を使う場合はこの書き方がよく使われていた。

**もしこう書かなかったら：**

この部分がないとカメラを起動できない。

また、撮影した写真をアプリで受け取ることもできない。

---

### Coordinatorパターン

```swift
class Coordinator: NSObject, UIImagePickerControllerDelegate, UINavigationControllerDelegate {
        let parent: CameraView

        init(_ parent: CameraView) {
            self.parent = parent
        }

        func imagePickerController(
            _ picker: UIImagePickerController,
            didFinishPickingMediaWithInfo info: [UIImagePickerController.InfoKey: Any]
        ) {
            if let image = info[.originalImage] as? UIImage {
                parent.capturedImage = image
            }
            parent.dismiss()
        }

        func imagePickerControllerDidCancel(_ picker: UIImagePickerController) {
            parent.dismiss()
        }
    }
```

**何をしているか：**

撮影が終わった時やキャンセルした時の処理を行っている。

写真が撮影された場合は、その画像を capturedImage に保存している。

**なぜこう書くのか：**

撮影後の画像を取得するために必要だからである。

また、撮影が終わったあとに自動でカメラ画面を閉じるためにも使われている。

**もしこう書かなかったら：**

撮影した画像を取得できない。

また、撮影後やキャンセル後にカメラ画面を閉じることができなくなる。

---

（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| 例：`PhotosPicker` | フォトライブラリから画像を選択するコンポーネント | `PhotosPicker(selection: $selectedItem, matching: .images)` |
| 例：`UIImagePickerController` | カメラまたはフォトライブラリにアクセスするUIKitコンポーネント | `picker.sourceType = .camera` |
| `UIViewControllerRepresentable` | UIKit の画面を SwiftUI で利用するために使う | `struct CameraView: UIViewControllerRepresentable` |
| `await` | 非同期処理の完了を待つために使う | `try await item.loadTransferable(type: Data.self)` |
| `@Binding` | 親Viewと値を共有するために使う | `@Binding var capturedImage: UIImage?` |

## 自分の実験メモ

（模範コードを改変して試したことを書く）

**実験1：**
- やったこと：Task を削除して await loadImage() を直接呼び出した。
- 結果：「Cannot pass function of type '(PhotosPickerItem?, PhotosPickerItem?) async -> ()'」というエラーが表示された。
- わかったこと：onChange のクロージャは同期処理なので、async 関数を直接呼び出すことはできない。async 関数を実行するには Task が必要だとわかった。

**実験2：**
- やったこと：matching: .images を削除して実行した。
- 結果：今回のシミュレータでは大きな違いは確認できなかった。
- わかったこと：シミュレータ内に動画がなかったため変化は見られなかったが、matching: .images は画像のみを選択対象にするための設定であることが分かった。

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：**
   **得られた理解：**

2. **質問：**
   **得られた理解：**

3. **質問：**
   **得られた理解：**

## この章のまとめ

（この章で学んだ最も重要なことを、未来の自分が読み返したときに役立つように書く）

# Metal学習
フレームワーク	プレフィックス
Metal	MTL
MetalKit	MTK
Metal Performance Shaders	MPS
MetalShading Language(MSL)：GPUで実行させる言語

# 処理の大まかな流れ
## クラスとプロトコル
### MTLDevice
GPUのプロトコル
### MTLCommandBuffer
コマンドを格納するコンテナ。GPUにはコマンドバッファ単位で渡される
コマンドを追加(エンコード)、コマンドバッファをコミットすることでGPUに送られる
コミット処理はこのクラスで行う
### MTLCommandQueue
コマンドバッファの実行順を管理するキュー
再利用可能なのでプロパティに保持できる
MTLDeviceから生成される
アプリに一つだけ作成
このクラスでMTLCommandBufferを生成する
生成したコマンドバッファをコミットされると生成元のコマンドキューにコマンドバッファが追加される
### MTLCommandEncoder
コマンドを作成しコマンドバッファに追加(エンコード)する
いくつか種類があるが、役割は変わらない

エンコーダーがコマンドバッファにコマンドを追加
コマンドバッファがコマンドキューにエンキュー(キューに追加)する
コマンドキューが順番にGPUに送られ実行される
※図を見るとコマンドバッファは複数あるっぽい
￼
### MTLBuffer, MTLTexture
MTLTextureはテクスチャデータを、MTLBufferは頂点データ等、レンダリングに
必要なパラメータを保持
どちらもCPUとGPUの共有メモリに確保される

## MetalKit
Metalを簡単に実装するためのフレームワーク
UIKitで実装する予定はないので省略
### MTKTextureLoader
共有メモリからテクスチャデータをアップロードし、MTLTextureオブジェクトを作成するためのクラス

### 画像を描画する例
書籍ではUIKit&MetalKitを使って今までの処理を実装している、SwiftUIでの実装は別途調べるので、ここでは描画コマンドを生成する方法だけ書く

#### 流れ
1. ドローアブルを生成（Metalで描画するためのクラス）
2. コマンドバッファ生成
3. テクスチャをコピーするエンコードを生成
4. 共有メモリにある(そしてプロパティとして保持してる)MTLTextureをエンコードを使ってコピーしてドローアブルのテクスチャにセット
5. エンコード完了処理
6. ドローアブルをコマンドバッファに登録
7. コマンドバッファをコミット(エンキュー)
以下はUIKit&MetalKitで描画する場合の処理
```swift
func draw(in view: MTKView) {
    // ドローアブルを取得
    guard let drawable = view.currentDrawable else { return }
    
    // コマンドバッファを作成
    let commandBuffer = commandQueue.makeCommandBuffer()!
    
    // コピーするサイズを計算
    let w = min(texture.width, drawable.texture.width)
    let h = min(texture.height, drawable.texture.height)
    
    // MTLBlitCommandEncoder を作成
    let blitEncoder = commandBuffer.makeBlitCommandEncoder()!
    
    // コピーコマンドをエンコード
    blitEncoder.copy(from: texture, // コピー元テクスチャ
                     sourceSlice: 0,
                     sourceLevel: 0,
                     sourceOrigin: MTLOrigin(x: 0, y: 0, z: 0),
                     sourceSize: MTLSizeMake(w, h, texture.depth),
                     to: drawable.texture, // コピー先テクスチャ
                     destinationSlice: 0,
                     destinationLevel: 0,
                     destinationOrigin: MTLOrigin(x: 0, y: 0, z: 0))
    
    // エンコード完了
    blitEncoder.endEncoding()
    
    // 表示するドローアブルを登録
    commandBuffer.present(drawable)
    
    // コマンドバッファをコミット（→エンキュー）
    commandBuffer.commit()
}
```

## Metalシェーダー処理を書く
シェーダーを書くファイルの拡張子.metal
ビルド時に.metallibというライブラリファイルが生成される
### MTLLibrary
ライブラリを表すプロトコル
### MTLDeviceからオブジェクトを生成できる
```
let library = device.makeDefaultLibrary()
```
デフォルトのライブラリを呼ぶときはメインバンドルにあるMetalライブラリファイルから生成される
### MTLFunction
ライブラリに記述されてる関数を表すオブジェクト
ここから関数を取得する
```
let vertexFunction = library.makeFunction(name: "vertexShader")
let fragmentFunction = library.makeFunction(name: "fragmentShader")
```

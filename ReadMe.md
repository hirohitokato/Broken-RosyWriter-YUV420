# Broken RosyWriter(YUV420)

AppleのサンプルコードRosyWriter（とGLCameraRipple）を参考に、ビデオのデータ形式を`kCVPixelFormatType_32BGRA`から`kCVPixelFormatType_420YpCbCr8BiPlanarVideoRange`に変更し、さらに毎フレーム呼び出されるデリゲートメソッド`captureOutput:didOutputSampleBuffer:fromConnection:`にて、サンプルバッファをコピーしようとしているプログラムです。

しかし、現状ではコピーしたサンプルバッファを使おうとしても、画面表示・ビデオ録画のどちらにも使えない、壊れたサンプルバッファしか作れていません。

__(2012/4/10 23:55 @whitedev氏と@norio_nomura氏のご協力により原因判明)__

## 目的
- デリゲートメソッド`captureOutput:didOutputSampleBuffer:fromConnection:`に届くYUV420のサンプルバッファを、__正しいデータとして__コピーすること。
- イメージバッファのコピーは「浅いコピー」ではなく「深いコピー」をすること

上記の処理を、`VideoProcessor.m`の中の`deepCopySampleBuffer:`メソッドで行おうとしています。

### 期待する結果
- 画面に表示した際、きちんと色が付いて見えること
- AVAssetWriterオブジェクトに`appendSampleBuffer:`メソッドでサンプルバッファを書き込むと、ムービーの１フレームとして保存してくれること

![expected result](https://github.com/katokichisoft/Broken-RosyWriter-YUV420/raw/master/expect.png)

### 実際の結果
- 画面に表示すると、緑色に覆われる
- AVAssetWriterオブジェクトに`appendSampleBuffer:`メソッドで書き込むと、不正なイメージデータだとしてエラーが返る

![actual result](https://github.com/katokichisoft/Broken-RosyWriter-YUV420/raw/master/result.png)

## なぜコピーしようとしているか
一般的な使い方だと、デリゲートメソッドに届くサンプルバッファは、描画に使ったり録画するなどして消費します。ですが、これをあえて保持しておくことで、複数フレームからデータを解析／加工することができないかと考えました。

取りあえず到着するサンプルバッファをローカルのバッファに溜めようとしたところ、13フレームぶんのサンプルバッファが到達したところで、デリゲートメソッドが呼ばれなくなってしまいました。どうやらフレームワーク側の処理で、各サンプルバッファが持つピクセルデータ領域に対し、バッファプールから取り出したデータを使い回しているようです。そのために、保持したまま消費しないとプールが枯渇してしまい、デリゲートメソッドが呼べなくなっていることが分かりました。

そこで、到着するサンプルバッファを、ピクセルバッファごとdeep copyしてしまうことで、フレームワークの資源を切り崩さないよう実装することにしました。

## なぜ浅いコピーでは駄目なのか（サンプルバッファのコピーAPIが使えない理由）
サンプルバッファのクラス`CMSampleBufferRef`にはコピー用のAPIとして、`CMSampleBufferCreateCopy()`が用意されています。
しかしこのAPIでは、サンップルバッファ内のイメージバッファ、つまりバッファプールから取り出したイメージバッファのretain countを増やすだけの、「浅いコピー」しか行いません。
この状態でコピーしたサンプルバッファを溜めても、バッファプールが枯渇することには変わりないのです。

### 調査により判明した原因
`CVPixelBufferRef`オブジェクトを作成するときに指定するPixelFormatDescriptionについて、OpenGLESコンパチの関連設定を__kCVPixelBufferIOSurfacePropertiesKey__にぶら下げる必要があった。
これまでは辞書の最上層に置かれていたために、認識できなかったのかも？

### 対策
属性設定を行うときの階層構造に注意する。詳細はオンラインマニュアルを参照してください。


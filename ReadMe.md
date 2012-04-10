# Broken RosyWriter(YUV420)

AppleのサンプルコードRosyWriter（とGLCameraRipple）を参考に、ビデオのデータ形式を`kCVPixelFormatType_32BGRA`から`kCVPixelFormatType_420YpCbCr8BiPlanarVideoRange`に変更し、さらに毎フレーム呼び出されるデリゲートメソッド`captureOutput:didOutputSampleBuffer:fromConnection:`にて、サンプルバッファをコピーしようとしているプログラムです。

しかし、現状ではコピーしたサンプルバッファを使おうとしても、画面表示・ビデオ録画のどちらにも使えない、壊れたサンプルバッファしか作れていません。

### 期待する結果

![expected result](https://github.com/katokichisoft/Broken-RosyWriter-YUV420/raw/master/expect.png)

### 現状

![actual result](https://github.com/katokichisoft/Broken-RosyWriter-YUV420/raw/master/result.png)

どなたか、原因や解決策の分かる方がいらっしゃったら教えていただきたいです・・・。

## 目的
- デリゲートメソッド`captureOutput:didOutputSampleBuffer:fromConnection:`に届くYUV420のサンプルバッファを、__正しいデータとして__コピーすること。
- イメージバッファのコピーは「浅いコピー」ではなく「深いコピー」をすること


## なぜコピーしようとしているか
一般的な使い方だと、デリゲートメソッドに届くサンプルバッファは、描画に使ったり録画するなどして消費します。ですが、これをあえて保持しておくことで、複数フレームからデータを解析／加工することができないかと考えました。

取りあえず到着するサンプルバッファをローカルのバッファに溜めようとしたところ、13フレームぶんのサンプルバッファが到達したところで、デリゲートメソッドが呼ばれなくなってしまいました。どうやらフレームワーク側の処理で、各サンプルバッファが持つピクセルデータ領域に対し、バッファプールから取り出したデータを使い回しているようです。そのために、保持したまま消費しないとプールが枯渇してしまい、デリゲートメソッドが呼べなくなっていることが分かりました。

そこで、到着するサンプルバッファを、ピクセルバッファごとdeep copyしてしまうことで、フレームワークの資源を切り崩さないよう実装することにしました。

## サンプルバッファのコピーAPIが使えない理由
サンプルバッファのクラス`CMSampleBufferRef`にはコピー用のAPIとして、`CMSampleBufferCreateCopy()`が用意されています。
しかしこのAPIでは、サンップルバッファ内のイメージバッファ、つまりバッファプールから取り出したイメージバッファのretain countを増やすだけの、「浅いコピー」しか行いません。
この状態でコピーしたサンプルバッファを溜めても、バッファプールが枯渇することには変わりないのです。
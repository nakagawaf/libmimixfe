# libmimixfe
mimi XFE module for Fairy I/O Tumbler / version 1.x

## はじめに

### 概要

T-01 用バイナリとして提供される libmimixfe.so とそのヘッダファイル、及び簡単な利用例を含むリポジトリです。 `include/` 以下にヘッダファイル、`examples` 以下に利用例のソースコードが含まれます。利用例は、説明のための実装であり、プロダクション用の実装には好適ではない場合があります。ライブラリ本体はリリースに移動しました。

### ライブラリ本体

[リリース](https://github.com/FairyDevicesRD/libmimixfe/releases) に各バージョンのライブラリが格納されています。1.0.2 からソースレポジトリには同梱しなくなりましたのでご留意ください。

### 依存ライブラリ

LED リングの制御のために libtumbler.so が必要です。
https://github.com/FairyDevicesRD/tumbler/tree/master/libtumbler

### 利用例のビルド

T-01 実機上の Makefile が `examples/` 直下に用意されています。

``````````.sh
$ cd examples
$ make
``````````

適宜リリースに格納されている lixmimife.so 及び上記 libtumbler.so とリンクして実行してください。いくつかの利用例は `sudo` を必要とします。

## libmimixfe API

### API 概要

libmimixfe は Tumbler の 18ch マイクを直接入力とし、設定に従った信号処理済音声を出力するライブラリです。最も単純には、信号処理として無処理であり、その場合は、18ch マイクから同期録音された 18ch 音声がそのまま出力されます。典型的には、信号処理として、例えば、エコーキャンセル、ビームフォーミング（多チャンネルノイズ抑制）、VAD（発話区間抽出）を行った音声が出力されます。これはつまり、Tumbler のスピーカーから再生されている音声をキャンセルし（エコーキャンセル）、特定方向のみの音声を取得することで、目的音声を強調し（ビームフォーミング）、人間の声の区間のみが抽出された（VAD）、音声が取得できるということになります。

処理済み音声が出力されるタイミングで、ユーザー定義のコールバック関数が、処理済み音声を引数に与えられた状態で呼ばれます。コールバック関数にはユーザー定義のデータを与えることも可能です。この仕組は、Linux 環境での録音・再生によく用いられる OSS である portaudio （Portable Cross-Platform Audio I/O） が提供している API と同様の仕組みであり、コールバック型 API と呼ばれることがあります。以降の記述でも、この呼称に従います。

### API ドキュメント

#### API 利用全体の流れ

libmimixfe が提供するクラスには、大きく２つの分類が有ります。第一に、音声信号処理の内容を設定するための各種設定用データクラス（構造体）であり、各信号処理機能に対応して、複数の設定用データクラスがあります。これらは、`XFETypedef.h` に定義されています。第二に、録音・信号処理を実際に実行するためのクラスで、`XFERecorder` クラスです。これは `XFERecorder.h` に定義されています。
設定用データクラスを引数に与えた `XFERecorder` クラスによって録音と信号処理を行う、というのが libmimixfe の基本的な利用方法です。具体的には、以下のような手順となります。

1. libmimixfe の各種設定用データクラスを構築し、必要に応じてメンバー変数を変更する
2. 前手順で構築したデータクラスと、ユーザー定義コールバック関数を引数に与えて `XFERecorder` クラスをコンストラクトする
3. `XFERecorder::start()` 関数の呼び出しによって、録音及び信号処理を開始する
4. メインスレッドを待機させる
5. （libmimixfe によって、音声が取得される都度、コンストラクタで指定されたユーザー定義コールバック関数が呼びされます）

#### XFERecorder クラス

録音及び信号処理を実行するクラスです。`XFERecorder.h` で定義されています。

##### コンストラクタ

``````````.cpp
XFERecorder::XFERecorder(
	const XFESourceConfig& sourceConfig,
	const XFEECConfig& ecConfig,
	const XFEVADConfig& vadConfig,
	const XFEBeamformerConfig& bfConfig,
	const XFELocalizerConfig& locConfig,
	const XFEOutputConfig& outConfig,
	recorderCallback_t recorderCallback,
	void *userdata);
``````````

コンストラクタでは、各種設定データクラスを引数に取ります。設定データクラスはコンストラクタで初期値が設定されているので、必要に応じてメンバー変数を変更します。次に、`recorderCallback_t` 型のユーザー定義コールバック関数を与えます。このコールバック関数は、libmimixfe によって、音声が取得された任意のタイミングでメインスレッドとは異なるスレッドで呼び出されます（コールバック型 API）、最後に、任意のユーザー定義型データを引数に与えることができます。このデータは、コールバック関数に与えられます。コールバック関数の実装例は、サンプルプログラムを参照してください。

このコンストラクタでは、標準例外のみ送出される可能性があります。

##### start() 

``````````.cpp
void XFERecorder::start()
``````````

この関数によって、録音及び信号処理が開始されます。この関数を呼び出してから、録音が開始されるまでに、マイクサブシステムを初期化するので、任意の待ち時間が発生する場合があります。待ち時間は典型的には１秒〜数秒です。

利用例は、サンプルプログラムを参照してください。典型的には、この関数はアプリケーションの中で、例えば発話単位ごとに、何回も呼び出すような用い方はしません。アプリケーションの初期化時に一度だけ呼び出し、libmimixfe は常時動作状態となるように利用します。

また `start()` された状態で、`stop()` せずに、連続的に繰り返し `start()` を呼ぶことはできません。この関数では、libmimixfe から `XFERecorderError` 例外が送出される可能性があります。

##### stop() 

``````````.cpp
int XFERecorder::stop()
``````````

この関数によって、録音及び信号処理が終了します。この関数を呼び出してから、実際に全内部処理が終了するまでには、バッファをフラッシュするため、任意の待ち時間が発生する場合があります。待ち時間は典型的には1秒〜数秒です。

この関数は、`XFERecorder` クラスのデストラクタで呼び出されるので、典型的には、ユーザープログラム側から明示的に呼び出す必要はありません。Tumbler での録音を完全に停止したい場合、例えば、マイクミュートがユーザーから指示された場合、などのみに利用します。マイクミュートは、単にコールバック関数に与えられた音声データを無視するという方法によっても実現することができますが、録音そのものを停止したい場合には `stop()` を呼ぶことになります。

返り値は、終了ステータスです。正常終了の場合 0 が返されます。

このクラスからは、標準例外のみ送出される可能性があります。

##### setLogLevel()

``````````.cpp
void XFERecorder::setLogLevel(int mask)
``````````

libmimixfe がシステムログに出力するログレベルを調整できます。`mask` はログマスクであり、LOG_UPTO マクロを利用することで簡単に設定することができます。デフォルトでは、INFO レベル以上のログが出力されます。開発段階では DEBUG レベル以上のログを出力しておくことが好ましいでしょう。利用例は、サンプルプログラムを参照してください。

##### isActive()

``````````.cpp
bool XFERecorder::isActive()
``````````

`XFERecorder` が `start()` された後、録音及び信号処理が継続されているかどうかを返します。例えば、コンストラクト直後は false、正常に `start()` が完了した場合には、true、`stop()` された場合には false が返されます。`start()` された後、何らかの内部エラー等によって録音及び信号処理が停止する場合があります。そのような場合には、false が返されることになります。

典型的には、ユーザープログラムのメインスレッドで `start()` された後、メインスレッドは、`isActive()` 関数を終了判定条件にした while ループによって待機状態とします。以下のような形が典型的です。

``````````.cpp
while(rec.isActive()){
	...
	sleep(1); // ビジーループを回避します。1 秒は典型的には待ち過ぎなので、適切な値を選択します。
}
``````````

#### XFERecorderError クラス

`XFERecorder` クラスのメンバ関数から送出される可能性のある libmimixfe 定義例外クラスです。本例外が発生した場合には、システムログに `errorstr()` の内容が自動的に記録されます。このクラスは `XFERecorder.h` で定義されています。

##### errorno()

``````````.cpp
const int XFERecorderError::errorno()
``````````

 エラーコードを返します。

##### errorstr()

``````````.cpp
const std::string XFERecorderError::errorstr()
``````````

可読なエラー文字列を返します。エラー文字列中には、エラーコードが含まれる場合があります。

#### XFESourceConfig クラス

`XFETypdef.h` に定義される設定データクラスで、libmimixfe で利用するサンプリングレート、マイクの利用用途など、録音自体に関係する部分（ソース）の設定を行います。

|変数名|初期値|単位／型|説明|
|---|---|---|---|
|samplingrate_|16000|Hz|libmimixfe の信号処理パイプラインに導入する音声のサンプリングレート。16000 のみの対応となります。|
|microphoneUsage_[18]|ヘッダファイル参照|MicrophoneUsage型|マイクの利用用途。どのマイクを録音用とし、どのマイクを参照用とし、どのマイクを単に無視するかの設定。初期値のみの対応となります。|

#### XFEVadConfig クラス

`XFETypdef.h` に定義される設定データクラスで、VAD（発話区間抽出処理）の設定を行います。マイクから録音される音声には、当然ながら、人の声が含まれる区間と、そうではない区間があります。例えば、発話と発話の間の無音（静音）が含まれたり、人の声以外の、環境雑音等が含まれます。VAD は、Voice Activity Detection/Detector の略であり、音声中の人の声が含まれる区間を推定し、入力音声から、人の声が含まれる区間のみを切り出す機能を示します。libmimixfe に含まれる VAD はシンプルな機械学習に基づく VAD で、高速な処理が出来る反面、その性能は一定程度に限られます。

|変数名|初期値|単位／型|説明|
|---|---|---|---|
|enable_|true|bool型|VAD を有効にするかどうか|
|timeToActive_|80|ミリ秒|VAD によって、発話開始であると判定される最短の長さ。これが短い方が、より敏速に発話開始判定を得られるが、短すぎるとゴミを拾う場合がある。|
|timeToInactive|800|ミリ秒|VAD によって、発話終了であると判定される最短の長さ。これが短い方が、より敏速に発話終了判定を得られるが、短すぎると発話途中で切られる場合がある|
|headPaddingTime_|400|ミリ秒|VAD が発話検出区間を切り出す際に、発話検出区間の前方に与えるマージン。これが長い方が誤判定に対してロバストになるが、無駄な非発話区間をサーバーに送ることになる。その事自体は、サーバー側に悪影響を与えないが、従量課金の場合コストデメリットがある|
|tailPaddingTime_|400|ミリ秒|VAD が発話検出区間を切り出す際に、発話検出区間の後方に与えるマージン。これが長い方が誤判定に対してロバストになるが、無駄な非発話区間をサーバーに送ることになる。その事自体は、サーバー側に悪影響を与えないが、従量課金の場合コストデメリットがある|
|rmsDbfs_|-96|dbFS|VAD による発話判定を行うために必要な最低音量[dbFS]|

#### XFEECConfig クラス

`XFETypdef.h` に定義される設定データクラスで、エコーキャンセルの設定を行います。Tumbler のスピーカーから音声が出力されている状況で録音を行った場合、録音される音声には、Tumbler のスピーカーから出力されている音声が混ざります。Tumbler のスピーカーとマイクの距離は、Tumbler のマイクと発話者の距離よりもずっと近いので、録音された音声を聞いてみると、発話者の声よりもずっと大きな音量で、Tumbler のスピーカーから出ている音が記録されてしまいます。この状況では、正しく音声認識を行うことはできません。このため、いくつかの音声認識システムでは、システムが発話等している際には、マイクをオフにするという対応が取られる場合があります。つまり、システム発話を遮って、ユーザーが発話することができない場合があります。

libmimixfe に含まれるエコーキャンセル（Echo Cancellation; EC）機能とは、Tumbler のスピーカーから再生されている音声を、録音データから消すための機能です。ここで「消す」と言った場合、２つの「消す」があります。ひとつは、「キャンセル」と呼ばれる処理で、これは、録音音声に余計な歪みを与えずに、消したい音声だけを消すという意味に使われます。もうひとつは「サプレッション（抑制／抑圧）」と呼ばれる処理で、これは、「キャンセル」の後に組み合わせて使われるもので、録音音声に若干の余計な歪みを与えてしまうが、「キャンセル」では消しきれなかった残留成分を、さらに小さくする、という意味で使われます。libmimixfe に含まれる EC 機能は、この２つの「消す」を備えています（EC/AES）

「キャンセル」は一般に、後段処理に対して良い影響しか持ちませんが、「サプレッション」は副作用として音声歪みを与えてしまうことから、後段処理に対して、良い影響と悪い影響の両面を持ちます。オーバーオールで良くなるか悪くなるかはケースバイケースであって、「サプレッション」によって、人間の聴感上は、よりきれいに聞こえたとしても、音声歪みの影響により、後段の機械処理には悪影響を与えるといった、体感的に矛盾する結果となることも典型的にありますので、留意が必要です。

EC 機能によって、Tumbler が音声応答を返しているときに、ユーザーの割り込み発話を受け付けることができるようになります。また、Tumbler が音楽を再生しているときに、音声入力を受け付けることもできます。ただし、大音量においては、EC 機能をもってしても、スピーカー再生音を消しきれない場合があるため留意が必要となります。

|変数名|初期値|単位／型|説明|
|---|---|---|---|
|enable_|true|bool型|EC を有効にするかどうか|
|pref_|Preference::Fast|XFEECConfig::Preference型|EC の速度／精度バランスの調整|
|aesEnable_|false|bool型|エコーサプレッション（AES）を有効にするかどうか。 AES は上述のように良い影響と悪い影響の両方を持つので留意が必要|
|aesRatio_|1.0|float型|AES の強さ。1.0 の場合、AES を最大の強さで実行し、0.0 の場合、AES を行わない場合と同等となる。AES の効果（自己音をより強く消す）と副作用（消した音が歪む）のバランスを調整するために用いることができますが、後段で音声認識器に導入する場合は、AES は結果的に不要となる場合が多くある。|

##### XFEECConfig::Preference 列挙型

EC の速度／精度バランスを調整するために利用される列挙型です。EC は libmimixfe に含まれる各種信号処理機能の中では、比較的計算量の大きい重い処理であるため、速度／精度バランスを取るための調整機能が用意されています。

|値|説明|
|---|---|
|Accurate|最高精度だが処理が遅い（リアルタイム処理ができない、すなわち RTF > 1.0 となる）|
|Balanced|速度精度バランスを取った調整（リアルタイム処理ができる、すなわち RTF < 1.0 となる）|
|Fast|最高速度だが、後段の認識器に若干の悪影響を与える可能性がある調整（リアルタイム処理ができる）|

デフォルト値は Fast です。システム発話中に、ユーザー発話が検出された場合、システム発話をそのままの音量で再生し続けるのではなく、音量を下げるか、システム発話を止めるか、何らかの反応を取ることが VUI の設計として好適です。システム発話が停止した場合は、EC は自動的に無効になります。システム発話の音量が下がった場合は、上記の調整の差による、性能の差は相対的に小さくなります。

#### XFEBeamformerConfig クラス

`XFETypdef.h` に定義される設定データクラスで、ビームフォーマーの設定を行います。Tumbler は 16 個のマイクを備えています。最大 16 個のマイクを同時に利用することで、目的とする音声を強調することができるようになります。目的信号を強調するということは、すなわち、非目的信号を抑制するということであるので、多チャンネルノイズ抑制と呼ばれる場合もありますが、同じ概念です。

libmimixfe では、多チャンネル音声信号を利用した目的音声信号強調のために、ビームフォーミング（Beamforming; BF）という手法を用いています。これは、特定方向の音声のみを集音することができる手法です。Tumbler では、8 個のマイクを搭載したマイク基板が二段重ねとなっているため、集音したい方向を三次元的に指定することができます。方向の指定は、別の設定データクラスで行います。本クラスでは、ビームフォーミング処理自体の調整を行うことができます。

ビームフォーミングによって、多チャンネル音声がまとめられ、（目的信号が強調された） 1ch 音声（モノラル音声）が出力されることに注意してください。また、別の設定データクラスの指定に依存しますが、同時に検出する音源数が複数個であると指定された場合、ひとつの音源毎にモノラル音声が出力されることから、libmimixfe 自体の出力としては、それらの多チャンネル信号出力となることに留意してください。例えば、同時検出する最大音源数が２音源と指定された場合で、２音源が同時に音声を出している場合、つまり例えば、向き合った２人の発話者が同時に発話している場合、には、それぞれの音源に対してビームフォーミングが同時に実行され、2ch の出力となります。２音源の片方だけが音を出している場合、つまり、例えば、向き合った２人のうち、片方の人だけが発話しているような場合、最大音源数は２音源であると指定されていても、出力は１音源となります。つまり、ビームフォーミング処理の入力チャネル数と出力チャネル数は異なる場合が多いということに留意してください。

ビームフォーミング処理には、ポストフィルター処理と呼ばれる、追加的なノイズ抑制の仕組みが備えられています。これは、EC 機能の AES 機能と同様で、追加的なノイズ抑制を行うことができるが、副作用として、音声歪みを与えます。AES と比較し、副作用の度合いは低いため、デフォルトでは true となっています。

|変数名|初期値|単位／型|説明|
|---|---|---|---|
|enable_|true|bool型|ビームフォーマーを有効にするかどうか|
|type_|type::MVDR_v2|XFEBeamformerConfig::type型|ビームフォーミングの内部処理の種類。内部処理詳細は一般には開示されていません。|
|sensitibity_|1.0|float型|-|
|postfilter_enable_|true|bool型|ポストフィルター処理実行の有無。追加的な音声強調ができるが、出力音声に若干の音質劣化を与える副作用がある。|

#### XFELocalizerConfig クラス

`XFETypedef.h` に定義される設定データクラスで、ビームフォーマーと合わせて用いられ、集音したい方向などの設定を行うための基底クラスです。このクラスをそのまま利用することはできず、下記の子クラスを利用する形になります。

|変数名|初期値|単位／型|説明|
|---|---|---|---|
|enable_|true|bool|音源定位モジュールを有効にするかどうか|
|maxSimultaneousSpeakers_|1|int|同時に定位する最大音源数、指定数に制限はありませんが、実用的には状況に依存し 1 〜 4 程度となります。|
|sourceDetectionSensitibity_|0.4|float|複数音源を同時定位する場合に、音源が複数個検出される感度を指定します。0 の場合に最も音源検出感度が低く、1.0 の場合に最も音源検出感度が高くなります。音源検出感度が高すぎると、実際には存在しない偽音源が検出されてしまう場合があります。T-01 の比較的近傍で発話される場合には、音源検出感度を低め、T-01 の比較的遠方で発話される場合には、高めに設定しておくと有効である場合があります。|
|area_|SearchArea::planar|SearchArea 型|音源を定位する範囲。planar の場合は平面、sphere の場合は球面。一般的な利用方法においては平面を推奨します。|
|identitalRange_|20|int|音源定位結果の誤差の吸収幅を設定します。固定方向設定の場合は、設定方向に対して許容する定位の半値幅を示します。動的定位の場合で複数音源を同時定位する場合は、設定角度以下を同一音源を見做して処理を行います。音源定位結果は 1 度単位で出力されますが、実際は誤差が含まれます。これらの誤差を吸収して、概ね同じ方向からの音声をひとつの音源とみなすための設定です。|

#### XFEStaticLocalizerConfig クラス

`XFETypedef.h` に定義される設定データクラスで、ビームフォーマーと組み合わせて用いられ、集音したい方向などの設定を行います。上述したビームフォーマーは、特定方向の音声のみを集音することができる目的音声強調手法でした。この設定クラスは、ビームフォーマーと組み合わせて用いて、目的方向を指定するために利用される設定クラスです。このクラスは `XFETypedef.h` に定義されているように、純粋仮想クラスである `XFELocalizerConfig` の派生クラスです。`XFELocalizerConfig` クラスは、目的方向の指定や同時検出音源数の指定に関係する共通的な要素を持つ親クラスであり、このクラス以外にもいくつかの派生クラスを持っていますので、目的に応じて、最適な派生クラスを利用してください。

派生クラスのうち、この設定データクラスは、ビームフォーマーの集音する方向（「目的方向」と呼びます）を、事前に固定する際に利用します。目的方向、複数方向指定することができます。目的方向以外の方向から到来した音声は無視されます。

例えば、自動受付システムなどでの応用で、発話者が Tumbler の正面にいると前提できる場合には、正面方向に目的方向を事前固定しておくことで、正面方向以外から到来した音声は無視されると同時に、正面方向から来た音声は強調されて出力されます。
他の例では、銀行窓口など有人の対面カウンターでの利用などで、Tumbler を挟む位置に向かい合った２人の発話者がいる場合は、Tumbler の真正面と真後ろに目的方向を事前固定しておくことが有効である場合があります。これにより、隣のカウンターから漏れてくる声には反応しないようにすることができます。

|変数名|初期値|単位／型|説明|
|---|---|---|---|
|targetDirections_|なし|std::vector<Direction>|Direction型変数のベクタによって１つ以上の目的方向を指定します。本クラスのコンストラクタで指定することもできます。|
	
#### XFEDynamicLocalizerConfig クラス

派生クラスのうち、この設定データクラスは、ビームフォーマーの目的方向を音源方向に向ける際に利用します。

#### XFEOutputConfig

v1.0 から追加された recorderCallback の呼び出され方を制御する設定データクラス。デフォルトのまま利用することが推奨されます。

#### StreamInfo クラス

`XFETypedef.h` に定義されるクラスで、`recorderCallback_t` 型のユーザー定義コールバック関数で libmimixfe から与えられる音声解析情報を含むデータクラスです。この解析情報は、10 ミリ秒ごとの解析結果となります。解析結果の有効活用方法については、本ドキュメントでは省略します。

|変数名|単位／型|説明|
|---|---|---|
|milliseconds_|ミリ秒|libmimixfe が録音を開始してからの経過時間。録音開始時刻を 0 としたときに、取得された音声バッファの録音時刻に相当。|
|direction_|Direction型|この 10 ミリ秒フレームで有効な音源が検出された場合、その推定方向。ビームフォーマーが無効の場合未定義|
|utteranceDirection_|Direction型|発話単位での推定方向。ビームフォーマーが無効の場合未定義|
|speechProbability_|[0,1.0]|この 10 ミリ秒フレーム全体での発話存在確率。VAD が無効の場合未定義|
|rmsDbfs_|dbFS|平均音量|
|numSoundSources_|個数|この 10 ミリ秒フレームで、実際に抽出された同時発生音源数。ユーザー定義コールバック関数が異なる音源番号で呼び出される|
|totalNumSoundSources_|個数|この 10 ミリ秒フレームでの推定同時発生音源数。|
|spatialSpectralPeak|dbFS|空間スペクトル値|
|spatialSpectrum|dbFS|平均空間スペクトル|

#### Direction クラス

三次元での方向ベクトルを表すクラスで、方位角（`azimuth_`）と迎え角（`angle_`）を指定することで、ひとつの方向ベクトルを定めます。`azimuth_` は正面方向が 270 度、`angle_` は水平面が 90 度になることに留意してください。この詳細については、[マイク位置情報](https://github.com/FairyDevicesRD/tumbler/tree/master/hardware_api/microphone/microphone_positions)を参照してください。単位は度（degree）です。

#### Sector クラス

Tumbler 設置水平面上での扇形を表すクラスで、方位角（`azimuth_`）指定された扇形の半径から反時計周りに中心角（`range_`）度の範囲を示します。

#### SpeechState 列挙型

VAD が有効の場合に、コールバック関数に与えられる VAD の発話検出状態を示す列挙型です。

|値|説明|
|---|---|
|SpeechStart|発話が開始されたことを示すステート。発話開始時点で１回のみ出力される。|
|InSpeech|発話中であることを示すステート。複数回出力される。|
|SpeechEnd|発話が終了したことを示すステート。発話終了時点で１回のみ出力される。|
|NonSpeech|発話が検出されていないことを示すステート。VAD 有効設定の場合、発話が検出されていなければコールバック関数はそもそも呼び出されないためコールバック関数中で観測されることはない。|

#### MicrophoneUsage 列挙型

Tumbler T-01 が搭載している 16 個のマイク 1ch のスピーカーフィードバック信号及び 1ch の外部入力信号、合計 18ch に対して、それぞれをどのように取り扱うかを決めるための列挙型です。

|値|説明|
|---|---|
|DISCARD|マイク入力を利用しない|
|INPUT|マイク入力を利用する|
|REFERENCE|マイク入力を参照音声信号として利用する|

#### recorderCallback_t 型

`XFERecorder` クラスのコンストラクタの引数に与える、ユーザー定義コールバック関数であり、`XFETypedef.h` で定義されています。

``````````.cpp
using recorderCallback_t = void (*)(
	short* buffer,
	size_t buflen,
	SpeechState state,
	int sourceId,
	StreamInfo* info,
	size_t infolen,
	void* userdata);
``````````

第一引数には、libmimixfe の出力として、libmimixfe が録音及び設定に従って信号処理をした処理済音声が与えられるバッファです。

第二引数は、第一引数に与えられたバッファの長さです。

第三引数は、`SpeechState` 型変数で、VAD が有効な場合に、`XFETypedef.h` で定義される VAD の判定状態が返されます。VAD が無効な場合には未定義です。

第四引数は、音源番号です。複数音源が同時検出される場合、最も平均音量の大きな音源から順に 1,2,... と通し番号が振られます。

第五引数は、`StreamInfo` 型配列で、`XFETypedef.h` で定義される、第一引数で与えられるバッファに含まれる音声を 10 ミリ秒ごとに解析した情報が返されます。設定次第で、内容は異なります。

第六引数は、第五引数に与えられる配列の長さです。

第七引数は、`XFERecorder` のコンストラクタで与えられるユーザー定義型データが素通しされます。

この関数内部でエラーが発生した場合に、それを明示的に libmimixfe に通知する方法はありません。そのような通知が必要な場合、ユーザー定義データを経由して、メインスレッド側で何らか適切な処理を行うようにしてください。



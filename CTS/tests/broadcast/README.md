BroadcastReceiverに関するテストは下記。

http://tools.oesf.biz/android-8.1.0_r1.0/xref/cts/tests/tests/content/src/android/content/cts/BroadcastReceiverTest.java

例として、onReceiveのテストを見てみいく。

http://tools.oesf.biz/android-8.1.0_r1.0/xref/cts/tests/tests/content/src/android/content/cts/BroadcastReceiverTest.java#169

```Java
    169     public void testOnReceive () throws InterruptedException {
    170         final MockActivity activity = getActivity();
    171 
    172         MockReceiverInternal internalReceiver = new MockReceiverInternal();  // レシーバーインスタンス化
    173         IntentFilter filter = new IntentFilter();
    174         filter.addAction(ACTION_BROADCAST_INTERNAL);  // ACTION_BROADCAST_INTERNAL(="android.content.cts.BroadcastReceiverTest.BROADCAST_INTERNAL")のアクションのブロードキャストを受け取るように追加
    175         activity.registerReceiver(internalReceiver, filter);  // レシーバーの登録
    176         // 送信前のテスト
    177         assertEquals(0, internalReceiver.getResultCode());  // resultコードが0であることをテスト
    178         assertEquals(null, internalReceiver.getResultData());// resultデータがnullであることをテスト
    179         assertEquals(null, internalReceiver.getResultExtras(false));
    180 
    181         activity.sendBroadcast(new Intent(ACTION_BROADCAST_INTERNAL)  // ブロードキャスト送信
    182                 .addFlags(Intent.FLAG_RECEIVER_FOREGROUND));
                // 送信後のテスト
    183         internalReceiver.waitForReceiver(SEND_BROADCAST_TIMEOUT);
    184 
    185         activity.unregisterReceiver(internalReceiver);  // 後処理
    186     }
```

上から見てみると、まずブロードキャスト送信前に、L177～L177でresultデータやresultデータが既にセットされていないことをテストしている。
そして、L181のActivity#sendBroadcast()でブロードキャストを送信している。
ブロードキャスト送信後はMockReceiverInternal#waitForReceiverをコールしている。
MockReceiverInternal#waitForReceiverの中身は下記のようになっている。

```Java
     98     private class MockReceiverInternal extends BroadcastReceiver  {
     99         protected boolean mCalledOnReceive = false;  // onReceiveコールフラグを初期化
    100         private IBinder mIBinder;
    101 
    102         @Override
    103         public synchronized void onReceive(Context context, Intent intent) {
    104             mCalledOnReceive = true;  
    105             Intent serviceIntent = new Intent(context, MockService.class);
    106             mIBinder = peekService(context, serviceIntent);
    107             notifyAll();
    108         }
    109 
    110         public boolean hasCalledOnReceive() {
    111             return mCalledOnReceive;
    112         }
    113 
    114         public void reset() {
    115             mCalledOnReceive = false;
    116         }
    117 
    118         public synchronized void waitForReceiver(long timeout)
    119                 throws InterruptedException {
    120             if (!mCalledOnReceive) {  // onReceiveコールフラグが立っていなければtimeout待つ
    121                 wait(timeout);
    122             }
    123             assertTrue(mCalledOnReceive);  // onReceiveコールフラグが立っていることを確認
    124         }
    125 
    126         public synchronized boolean waitForReceiverNoException(long timeout)
    127                 throws InterruptedException {
    128             if (!mCalledOnReceive) {
    129                 wait(timeout);
    130             }
    131             return mCalledOnReceive;
    132         }
    133 
    134         public IBinder getIBinder() {
    135             return mIBinder;
    136         }
    137     }
```

timeout後に、mCalledOnReceiveがtrueになっている、つまりonReceiveがtimiout内にコールされたかをテストしている。

次にブロードキャスト送信⇒ブロードキャスト処理⇒onReceiveが呼ばれる当たりの処理の流れを見ていく。
ここからはOS側の処理に移る。

とっつき方としては入り口(Activity#sendBroadcast)と終端(BroadcastReceiver#onReceive)両者から調べていく。

後者の方はスタックトレースを出力するログをCTS側のonReceiveに入れてどのようなシーケンスでコールされたか確認していく。
運が良ければ入り口まで繋がっているスタックトレースが出てくれるかもしれないが、例えば間にbinderやハンドラ等入ってくると途切れてしまう。

前者に関してはAndroid OSのソース検索ツールOpenGrokを使ってひたすらソースを追っていく、静的解析的なアプローチをとる。
https://sites.google.com/site/devcollaboration/codesearch

### 入り口(Activity#sendBroadcast)

まずはActivityクラスからsendBroadcastの処理を見ていく。
OpenGrokのFile PathにActivity.javaといれて検索する。
いくつか出てくるが、アプリからアクセスできる公開API(Android SDKに含まれているもの)の実装は基本的に/frameworks/base/core/java配下にある。

http://tools.oesf.biz/android-8.1.0_r1.0/xref/frameworks/base/core/java/android/app/Activity.java

sendBroadcastは見当たらず。継承元クラスのContextThemeWrapperを見ていく。

http://tools.oesf.biz/android-8.1.0_r1.0/xref/frameworks/base/core/java/android/view/ContextThemeWrapper.java

こちらにも見当たらない。さらに継承元のContextWrapperを見てみる。

http://tools.oesf.biz/android-8.1.0_r1.0/xref/frameworks/base/core/java/android/content/ContextWrapper.java

```Java
    435     @Override
    436     public void sendBroadcast(Intent intent) {
    437         mBase.sendBroadcast(intent);
    438     }
    439 
    440     @Override
    441     public void sendBroadcast(Intent intent, String receiverPermission) {
    442         mBase.sendBroadcast(intent, receiverPermission);
    443     }
    444 
    445     /** @hide */
    446     @Override
    447     public void sendBroadcastMultiplePermissions(Intent intent, String[] receiverPermissions) {
    448         mBase.sendBroadcastMultiplePermissions(intent, receiverPermissions);
    449     }
    450 
    451     /** @hide */
    452     @SystemApi
    453     @Override
    454     public void sendBroadcast(Intent intent, String receiverPermission, Bundle options) {
    455         mBase.sendBroadcast(intent, receiverPermission, options);
    456     }
    457 
    458     /** @hide */
    459     @Override
    460     public void sendBroadcast(Intent intent, String receiverPermission, int appOp) {
    461         mBase.sendBroadcast(intent, receiverPermission, appOp);
    462     }
```

mBaseのsendBroadcastをコールしている。
mBaseは

```Java
     54 public class ContextWrapper extends Context {
     55     Context mBase;
```

Contextクラスのインスタンスなので、Contextクラスを見ていく。

http://tools.oesf.biz/android-8.1.0_r1.0/xref/frameworks/base/core/java/android/content/Context.java

```Java
   1895     public abstract void sendBroadcast(@RequiresPermission Intent intent);
```

抽象メソッドになっている。overrideしている箇所を探す。

OpenGrokでFull Searchに「“extends Context”」、File Pathに「/frameworks/base」と入力して検索してみる。

http://tools.oesf.biz/android-8.1.0_r1.0/search?q="extends+Context"&defs=&refs=&path=%2Fframeworks%2Fbase&hist=

3つ出てくるが、名前からしても
http://tools.oesf.biz/android-8.1.0_r1.0/xref/frameworks/base/core/java/android/app/ContextImpl.java
が間違いなさそう。

```Java
    964     @Override
    965     public void sendBroadcast(Intent intent) {
    966         warnIfCallingFromSystemProcess();
    967         String resolvedType = intent.resolveTypeIfNeeded(getContentResolver());
    968         try {
    969             intent.prepareToLeaveProcess(this);
    970             ActivityManager.getService().broadcastIntent(
    971                     mMainThread.getApplicationThread(), intent, resolvedType, null,
    972                     Activity.RESULT_OK, null, null, null, AppOpsManager.OP_NONE, null, false, false,
    973                     getUserId());
    974         } catch (RemoteException e) {
    975             throw e.rethrowFromSystemServer();
    976         }
    977     }
```

ActivityManager#getService()から帰ってくる何らかのクラスのbroadcastIntentメソッ
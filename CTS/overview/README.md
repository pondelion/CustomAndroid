
## 概要

CTSとはCompatibility Test Suiteの略で、Googleが提供するAndroid OSのテスト群。
Android OSはAndroid Compatibility Definition Document(https://source.android.com/compatibility/cdd)に記載されている内容を遵守している必要があり、カスタムAndroid OSを販売するにあたり、CTSを通したうえでGoogleに提出しなければいけない。
改変したAndroidが基準を満たしているかをチェックするのがCTS。

CTSは通常Android端末を接続したPCからcts実行のコマンドを実行し自動的に実行されるが、手動でインストールして手動でテストを行うCTS Verifierというものも含まれる。

CTSのソースコードはOSと同様公開されており、例えば8.1_r12だと下記から見る。
https://android.googlesource.com/platform/cts/+/android-cts-8.1_r12/

CTSとは別にGTSと呼ばれるものがあり、これはAOSPに含まれていないGMSアプリ(Gmail、YouTube、マップ、フォト等)のテストで、GMSアプリのソースコードが公開されていないのと同様にGTSのソースコードも公開されておらずバイナリで提供る。

## CTSの実行

CTSテストのバイナリは下記公式サイトからDLできる。
セットアップ手順等も記載されている。

https://source.android.com/compatibility/cts/setup

実行方法は下記。

https://source.android.com/compatibility/cts/run

全体実行やモジュール実行・メソッド実行など細かい設定はできるが、実行手順等の詳細は割愛。

CTS自体をソースコードからのビルドも可。

## CTSの構成

基本的に実行されるテストは

https://android.googlesource.com/platform/cts/+/android-cts-8.1_r12/hostsidetests/
https://android.googlesource.com/platform/cts/+/android-cts-8.1_r12/tests/

で、前者がホストPC上で実行されるテストで、後者がAndroid側にインストールされて実行されるテスト。

後者を見てみると構成は大まかに下記のようになっている。

```
JobScheduler/
ProcessTest/
acceleration/
accessibility/
accessibilityservice/
admin/
app/
aslr/
autofillservice/
backup/
camera/
core/
dram/
expectations/
filesystem/
fragment/
inputmethod/
jank/
jdwp/
leanbackjank/
libcore/
netlegacy22.api/
netlegacy22.permission/
netsecpolicy/
openglperf2/
pdf/
sample/
sensor/
signature/
simplecpu/
systemAppTest/
tests/
tvprovider/
ui/
video/
vm/
vr/
```

上記のtests内にも色々ある。

```
Android.mk
accounts/
alarmclock/
animation/
app.usage/
app/
appwidget/
assist/
background/
bionic/
bluetooth/
calendarcommon/
car/
carrierapi/
colormode/
contactsproviderwipe/
content/
database/
debug/
display/
dpi/
dpi2/
dreams/
drm/
dynamic_linker/
effect/
externalservice/
gesture/
graphics/
hardware/
icu/
incident/
jni/
jni_vendor/
keystore/
libcorefileio/
libcorelegacy22/
location/
location2/
media/
mediastress/
midi/
multiuser/
nativehardware/
nativemedia/
ndef/
net/
netsecpolicy/
networksecurityconfig/
neuralnetworks/
opengl/
openglperf/
os/
packageinstaller/
permission/
permission2/
preference/
preference2/
print/
proto/
provider/
renderscript/
renderscriptlegacy/
rsblas/
rscpp/
sax/
security/
selinux/
shortcutmanager/
simpleperf/
speech/
systemintents/
systemui/
telecom/
telecom2/
telecom3/
telephony/
telephony2/
text/
theme/
toast/
toastlegacy/
transition/
tv/
uiautomation/
uidisolation/
uirendering/
util/
view/
voiceinteraction/
voicesettings/
webkit/
widget/
wrap/
```


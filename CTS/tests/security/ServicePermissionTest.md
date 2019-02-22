
セキュリティに関するテストServicePermissionTest

https://android.googlesource.com/platform/cts/+/refs/tags/android-cts-8.0_r17/tests/tests/security/src/android/security/cts/ServicePermissionsTest.java

コードは短いので、上から順に処理を見ていく。

https://search.siprop.org/android-8.1.0_r1.0/xref/cts/tests/tests/security/src/android/security/cts/ServicePermissionsTest.java#64
```java
64     public void testDumpProtected() throws Exception {
65 
66         String[] services = null;
67         try {
68             services = (String[]) Class.forName("android.os.ServiceManager")
69                     .getDeclaredMethod("listServices").invoke(null);
70         } catch (ClassCastException e) {
71         } catch (ClassNotFoundException e) {
72         } catch (NoSuchMethodException e) {
73         } catch (InvocationTargetException e) {
74         } catch (IllegalAccessException e) {
75         }
76 
77         if ((services == null) || (services.length == 0)) {
78             Log.w(TAG, "No registered services, that's odd");
79             return;
80         }
```

L68 : リフレクションを用いてandroid.os.ServiceManager#listServiceを呼び出し、返されたサービス名一覧を取得している。

L77 : nullが返された、もしくは空リストが返された場合は、異常な結果としてリターンしている。(failではない)

https://search.siprop.org/android-8.1.0_r1.0/xref/cts/tests/tests/security/src/android/security/cts/ServicePermissionsTest.java#82
```java
82         for (String service : services) {
83             mTempFile.delete();
84 
85             IBinder serviceBinder = null;
86             try {
87                 serviceBinder = (IBinder) Class.forName("android.os.ServiceManager")
88                         .getDeclaredMethod("getService", String.class).invoke(null, service);
89             } catch (ClassCastException e) {
90             } catch (ClassNotFoundException e) {
91             } catch (NoSuchMethodException e) {
92             } catch (InvocationTargetException e) {
93             } catch (IllegalAccessException e) {
94             }
95 
96             if (serviceBinder == null) {
97                 Log.w(TAG, "Missing service " + service);
98                 continue;
99             }
100 
101             Log.d(TAG, "Dumping service " + service);
102             final FileOutputStream out = new FileOutputStream(mTempFile);
103             try {
104                 serviceBinder.dump(out.getFD(), new String[0]);
105             } catch (SecurityException e) {
106                 String msg = e.getMessage();
107                 if ((msg == null) || msg.contains("android.permission.DUMP")) {
108                     // Service correctly checked for DUMP permission, yay
109                 } else {
110                     // Service is throwing about something else; they're
111                     // probably not checking for DUMP.
112                     throw e;
113                 }
114             } catch (TransactionTooLargeException | DeadObjectException e) {
115                 // SELinux likely prevented the dump - assume safe
116                 continue;
117             } finally {
118                 out.close();
119             }
120 
121             // Verify that dump produced minimal output
122             final BufferedReader reader = new BufferedReader(
123                     new InputStreamReader(new FileInputStream(mTempFile)));
124             final ArrayList<String> lines = new ArrayList<String>();
125             try {
126                 String line;
127                 while ((line = reader.readLine()) != null) {
128                     lines.add(line);
129                     Log.v(TAG, "--> " + line);
130                 }
131             } finally {
132                 reader.close();
133             }
134 
135             if (lines.size() > 1) {
136                 fail("dump() for " + service + " produced several lines of output; this "
137                         + "may be leaking sensitive data.  At most, services should emit a "
138                         + "single line when the caller doesn't have DUMP permission.");
139             }
140 
141             if (lines.size() == 1) {
142                 String message = lines.get(0);
143                 if (!message.contains("Permission Denial") &&
144                         !message.contains("android.permission.DUMP")) {
145                     fail("dump() for " + service + " produced a single line which didn't "
146                             + "reference a permission; it may be leaking sensitive data.");
147                 }
148             }
149         }
150     }
```

得られた各サービスに対してループ。

L87 : リフレクションを用いてandroid.os.ServiceManager#getServiceにserviceを引数で渡してコールし、返されたサービスのインスタンスを保存。

L96 : nullが返された場合該当のサービスはスキップ。

L102～L119 : キャッシュディレクトリ(mTempFile = new File(getContext().getCacheDir(), "CTS_DUMP"))を出力先としてFileOutputStreamを作成し、上記getServiceで得られたサービスのインスタンスのdumpメソッドにファイルディスクリプタとテスト文字列を渡しダンプを試みている。
その際に、SecurityExceptionが発生し、メッセージがnullもしくは「android.permission.DUMP」を含んでいる場合ダンプパーミッションが正常にチェックされているとし先に進むが、それ以外の場合はダンプパーミッションが正常にチェックされていないと判断し例外をスローしている(つまりfail)。
TransactionTooLargeExceptionもしくはDeadObjectExceptionが発生した場合は、SELinuxがdumpを阻止したとし、安全であると判断し該当サービスのチェックをスキップする。

L121～L133 : 上記のdumpが最小限の出力をしたかチェック。先ほどのダンプファイルを読み込み、一行ずつリスト(lines)に追加

L135 : 2行以上だった場合、sensitive dataを漏洩している可能性があり、呼び出し元がダンプパーミッションを持たない場合は多くても1行の出力出なくてはならないとしfail。

L141 : 1行であり、文字列に「Permission Denial」もしくは「android.permission.DUMP」を含んでいない場合は、sensitive dataを漏洩している可能性があるとしfail。


任意サービスのdumpの処理に関するテストなので、いくつかのシステムサービスを例としてみていく

- LocationManagerService

https://search.siprop.org/android-8.1.0_r1.0/xref/frameworks/base/services/core/java/com/android/server/LocationManagerService.java#3045
```java
3045     @Override
3046     protected void dump(FileDescriptor fd, PrintWriter pw, String[] args) {
3047         if (!DumpUtils.checkDumpPermission(mContext, TAG, pw)) return;
3048 
3049         synchronized (mLock) {
3050             if (args.length > 0 && args[0].equals("--gnssmetrics")) {
3051                 if (mGnssMetricsProvider != null) {
3052                     pw.append(mGnssMetricsProvider.getGnssMetricsAsProtoString());
3053                 }
3054                 return;
3055             }
…
```

- DiskStatsService

https://search.siprop.org/android-8.1.0_r1.0/xref/frameworks/base/services/core/java/com/android/server/DiskStatsService.java#65
```java
64     @Override
65     protected void dump(FileDescriptor fd, PrintWriter pw, String[] args) {
66         if (!DumpUtils.checkDumpAndUsageStatsPermission(mContext, TAG, pw)) return;
67 
68         // Run a quick-and-dirty performance test: write 512 bytes
69         byte[] junk = new byte[512];
70         for (int i = 0; i < junk.length; i++) junk[i] = (byte) i;  // Write nonzero bytes
71 
72         File tmp = new File(Environment.getDataDirectory(), "system/perftest.tmp");
73         FileOutputStream fos = null;
74         IOException error = null;
…
```

- UserManagerService

https://search.siprop.org/android-8.1.0_r1.0/xref/frameworks/base/services/core/java/com/android/server/pm/UserManagerService.java#3375
```java
3375     @Override
3376     protected void dump(FileDescriptor fd, PrintWriter pw, String[] args) {
3377         if (!DumpUtils.checkDumpPermission(mContext, LOG_TAG, pw)) return;
3378 
3379         long now = System.currentTimeMillis();
3380         StringBuilder sb = new StringBuilder();
3381         synchronized (mPackagesLock) {
3382             synchronized (mUsersLock) {
3383                 pw.println("Users:");
3384                 for (int i = 0; i < mUsers.size(); i++) {
3385                     UserData userData = mUsers.valueAt(i);
3386                     if (userData == null) {
3387                         continue;
3388                     }
…
```

※いくつか見るとわかるが、すべてのサービスのdump内で渡されたファイルディスクリプタを用いてファイルに書き込み処理をしているわけではない。

いずれもDumpUtils#checkDumpPermissionを最初にコールしており、これが正常に動作しているかのテストと考えられる。
falseが返された場合、つまりパーミッションチェックがNGだった場合は即returnし何もファイルにdumpしない。(⇒passルート)

DumpUtils#checkDumpPermissionの処理を見ていく。

https://search.siprop.org/android-8.1.0_r1.0/xref/frameworks/base/core/java/com/android/internal/util/DumpUtils.java#78
```java
78     public static boolean checkDumpPermission(Context context, String tag, PrintWriter pw) {
79         if (context.checkCallingOrSelfPermission(android.Manifest.permission.DUMP)
80                 != PackageManager.PERMISSION_GRANTED) {
81             logMessage(pw, "Permission Denial: can't dump " + tag + " from from pid="
82                     + Binder.getCallingPid() + ", uid=" + Binder.getCallingUid()
83                     + " due to missing android.permission.DUMP permission");
84             return false;
85         } else {
86             return true;
87         }
88     }
```

Context#checkCallingOrSelfPermissionのより、呼び出し元がandroid.Manifest.permission.DUMPのパーミッションを持っているかを確認している。

まとめると、今回対象のCTSアプリのような、android.Manifest.permission.DUMPパーミッションは持っていない呼び出し元は、android.os.ServiceManager#listServiceで取得できる全てのサービスに対してdumpを呼び出した際に、上記の様なフローでcheckCallingOrSelfPermissionのパーミッションチェック処理が機能しはじかれ、不正なファイル書き込みを行えないことを期待している。
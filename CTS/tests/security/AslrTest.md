
ASLR（アドレス空間配置のランダム化）が正しく有効化されているか確認するテスト

https://android.googlesource.com/platform/cts/+/refs/tags/android-cts-8.0_r17/tests/tests/security/src/android/security/cts/AslrTest.java

テストは下記の3つ

- testRandomization
- testOneExecutableIsPie
- testVaRandomize

順にみていく。

### testRandomization

```java
102     public void testRandomization() throws Exception {
103         testMappingEntropy("\\[stack\\]");
104         testMappingEntropy("/system/bin/");
105     }
```
「\\[stack\\]」と「/system/bin/」を引数(mappingName)として渡し、testMappingEntropyメソッドをコール。

```java
89     private void testMappingEntropy(String mappingName) throws Exception {
90         if (readMappingAddress(mappingName) == null) {
91             Log.i(TAG, mappingName + " does not exist");
92             return;
93         }
94 
95         int entropy = calculateEntropyBits(mappingName);
96 
97         assertTrue("Insufficient " + mappingName + " randomization (" +
98             entropy + " bits, >= " + aslrMinEntropyBits + " required)",
99             entropy >= aslrMinEntropyBits);
100     }
```

L90でreadMappingAddressにmappingNameを渡しコールし、戻り値がnullなら存在しないとしスキップ。
名前からマッピングされているアドレスを取得するメソッドと推測される。中身を見ていく。

```java
48     private String readMappingAddress(String mappingName) throws Exception {
49         ParcelFileDescriptor pfd = getInstrumentation().getUiAutomation()
50                 .executeShellCommand("/system/bin/cat /proc/self/maps");
51 
52         String result = null;
53         try (BufferedReader reader = new BufferedReader(
54                 new InputStreamReader(new ParcelFileDescriptor.AutoCloseInputStream(pfd)))) {
55             Pattern p = Pattern.compile("^([a-f0-9]+).*" + mappingName + ".*");
56             String line;
57 
58             while ((line = reader.readLine()) != null) {
59                 // Even after a match is found, read until the end to clean up the pipe.
60                 if (result != null) continue;
61 
62                 Matcher m = p.matcher(line);
63 
64                 if (m.matches()) {
65                     result = m.group(1);
66                 }
67             }
68         }
69         return result;
70     }
```

L49~L50で「/system/bin/cat /proc/self/maps」のコマンドを実行し、結果を正規表現「"^([a-f0-9]+).*" + mappingName + ".*"」
でパターンマッチさせ、一致したら([a-f0-9]+)の部分を取り出しresultとしてリターンしている。

L90に戻ると、もし上記で一致するものがありresultで返されていれば先へ進み、なければスキップ。

---

### testOneExecutableIsPie


---

### testVaRandomize
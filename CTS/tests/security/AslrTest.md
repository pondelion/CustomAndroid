
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

実際に「adb shell /system/bin/cat /proc/self/maps」により上記コマンドを実行すると下記のようになっている。
```
5579c3b000-5579c98000 r-xp 00000000 103:09 417                           /system/bin/toybox
5579c98000-5579c9b000 r--p 0005c000 103:09 417                           /system/bin/toybox
5579c9b000-5579c9d000 rw-p 0005f000 103:09 417                           /system/bin/toybox
5579c9d000-5579ca1000 rw-p 00000000 00:00 0
75fa400000-75fa800000 rw-p 00000000 00:00 0                              [anon:libc_malloc]
75fa98c000-75fa98e000 r-xp 00000000 103:09 1695                          /system/lib64/libnetd_client.so
75fa98e000-75fa98f000 ---p 00000000 00:00 0
75fa98f000-75fa990000 r--p 00002000 103:09 1695                          /system/lib64/libnetd_client.so
75fa990000-75fa991000 rw-p 00003000 103:09 1695                          /system/lib64/libnetd_client.so
75fa9d5000-75fa9ea000 r-xp 00000000 103:09 1805                          /system/lib64/libz.so
75fa9ea000-75fa9eb000 r--p 00014000 103:09 1805                          /system/lib64/libz.so
75fa9eb000-75fa9ec000 rw-p 00015000 103:09 1805                          /system/lib64/libz.so
75faa22000-75fab27000 r-xp 00000000 103:09 1587                          /system/lib64/libcrypto.so
75fab27000-75fab28000 ---p 00000000 00:00 0
75fab28000-75fab39000 r--p 00105000 103:09 1587                          /system/lib64/libcrypto.so
75fab39000-75fab3a000 rw-p 00116000 103:09 1587                          /system/lib64/libcrypto.so
75fab3a000-75fab3b000 rw-p 00000000 00:00 0                              [anon:.bss]
75fab47000-75fab7e000 r-xp 00000000 103:09 1669                          /system/lib64/libm.so
75fab7e000-75fab7f000 r--p 00036000 103:09 1669                          /system/lib64/libm.so
75fab7f000-75fab80000 rw-p 00037000 103:09 1669                          /system/lib64/libm.so
75fabb9000-75fabbb000 r-xp 00000000 103:09 1596                          /system/lib64/libdl.so
75fabbb000-75fabbc000 r--p 00001000 103:09 1596                          /system/lib64/libdl.so
75fabbc000-75fabbd000 rw-p 00002000 103:09 1596                          /system/lib64/libdl.so
75fabbd000-75fabbe000 rw-p 00000000 00:00 0                              [anon:.bss]
75fabf2000-75fabf4000 r-xp 00000000 103:09 1712                          /system/lib64/libpackagelistparser.so
75fabf4000-75fabf5000 r--p 00001000 103:09 1712                          /system/lib64/libpackagelistparser.so
75fabf5000-75fabf6000 rw-p 00002000 103:09 1712                          /system/lib64/libpackagelistparser.so
75fac08000-75fac1f000 r-xp 00000000 103:09 1663                          /system/lib64/liblog.so
75fac1f000-75fac20000 r--p 00016000 103:09 1663                          /system/lib64/liblog.so
75fac20000-75fac21000 rw-p 00017000 103:09 1663                          /system/lib64/liblog.so
75fac4d000-75fac6c000 r-xp 00000000 103:09 1715                          /system/lib64/libpcre2.so
75fac6c000-75fac6d000 ---p 00000000 00:00 0
75fac6d000-75fac6e000 r--p 0001f000 103:09 1715                          /system/lib64/libpcre2.so
75fac6e000-75fac6f000 rw-p 00020000 103:09 1715                          /system/lib64/libpcre2.so
75fac92000-75faca7000 r-xp 00000000 103:09 1736                          /system/lib64/libselinux.so
75faca7000-75faca8000 ---p 00000000 00:00 0
75faca8000-75faca9000 r--p 00015000 103:09 1736                          /system/lib64/libselinux.so
75faca9000-75facaa000 rw-p 00016000 103:09 1736                          /system/lib64/libselinux.so
75facaa000-75facab000 rw-p 00000000 00:00 0                              [anon:.bss]
75facd0000-75fadb2000 r-xp 00000000 103:09 1570                          /system/lib64/libc++.so
75fadb2000-75fadb3000 ---p 00000000 00:00 0
75fadb3000-75fadbb000 r--p 000e2000 103:09 1570                          /system/lib64/libc++.so
75fadbb000-75fadbc000 rw-p 000ea000 103:09 1570                          /system/lib64/libc++.so
75fadbc000-75fadbf000 rw-p 00000000 00:00 0                              [anon:.bss]
75fadc7000-75fae8c000 r-xp 00000000 103:09 1571                          /system/lib64/libc.so
75fae8c000-75fae8d000 ---p 00000000 00:00 0
75fae8d000-75fae93000 r--p 000c5000 103:09 1571                          /system/lib64/libc.so
75fae93000-75fae95000 rw-p 000cb000 103:09 1571                          /system/lib64/libc.so
75fae95000-75fae96000 rw-p 00000000 00:00 0                              [anon:.bss]
75fae96000-75fae97000 r--p 00000000 00:00 0                              [anon:.bss]
75fae97000-75fae9e000 rw-p 00000000 00:00 0                              [anon:.bss]
75faec6000-75faee6000 r--s 00000000 00:0c 11376                          /dev/__properties__/u:object_r:default_prop:s0
75faee6000-75faef6000 r-xp 00000000 103:09 1590                          /system/lib64/libcutils.so
75faef6000-75faef8000 r--p 0000f000 103:09 1590                          /system/lib64/libcutils.so
75faef8000-75faef9000 rw-p 00011000 103:09 1590                          /system/lib64/libcutils.so
75faefb000-75faefc000 r--p 00000000 00:00 0                              [anon:atexit handlers]
75faefc000-75faf1c000 r--s 00000000 00:0c 11377                          /dev/__properties__/properties_serial
75faf1c000-75faf1d000 rw-p 00000000 00:00 0                              [anon:arc4random data]
75faf1d000-75faf1e000 rw-p 00000000 00:00 0                              [anon:linker_alloc]
75faf2b000-75faf2c000 rw-p 00000000 00:00 0                              [anon:linker_alloc_small_objects]
75faf50000-75faf51000 rw-p 00000000 00:00 0                              [anon:linker_alloc_vector]
75faf5a000-75faf5b000 r--p 00000000 00:00 0                              [anon:linker_alloc]
75faf83000-75faf84000 rw-p 00000000 00:00 0                              [anon:linker_alloc_vector]
75faf84000-75faf85000 rw-p 00000000 00:00 0                              [anon:linker_alloc_small_objects]
75faf85000-75faf86000 rw-p 00000000 00:00 0                              [anon:linker_alloc_vector]
75faf94000-75faf95000 rw-p 00000000 00:00 0                              [anon:linker_alloc_small_objects]
75faf96000-75faf97000 rw-p 00000000 00:00 0                              [anon:linker_alloc]
75faf97000-75fafb7000 r--s 00000000 00:0c 11372                          /dev/__properties__/u:object_r:debug_prop:s0
75fafb7000-75fafd7000 r--s 00000000 00:0c 11377                          /dev/__properties__/properties_serial
75fafd7000-75fafd8000 rw-p 00000000 00:00 0                              [anon:linker_alloc_vector]
75fafd8000-75fafd9000 rw-p 00000000 00:00 0                              [anon:linker_alloc_small_objects]
75fafda000-75fafdb000 rw-p 00000000 00:00 0                              [anon:linker_alloc_small_objects]
75fafdb000-75fafdc000 rw-p 00000000 00:00 0                              [anon:linker_alloc_vector]
75fafdc000-75fafdd000 ---p 00000000 00:00 0
75fafdd000-75fafde000 rw-p 00000000 00:00 0
75fafde000-75fafdf000 ---p 00000000 00:00 0
75fafdf000-75fafe0000 rw-p 00000000 00:00 0                              [anon:linker_alloc_lob]
75fafe0000-75fafe1000 r--p 00000000 00:00 0                              [anon:linker_alloc]
75fafe1000-75fafe2000 rw-p 00000000 00:00 0                              [anon:linker_alloc_vector]
75fafe2000-75fafe3000 rw-p 00000000 00:00 0                              [anon:linker_alloc_small_objects]
75fafe3000-75fafe4000 r--p 00000000 00:00 0                              [anon:linker_alloc]
75fafe4000-75fafe5000 rw-p 00000000 00:00 0                              [anon:linker_alloc_small_objects]
75fafe5000-75fafe6000 r--p 00000000 00:00 0                              [anon:atexit handlers]
75fafe6000-75fafe7000 ---p 00000000 00:00 0                              [anon:thread signal stack guard page]
75fafe7000-75fafeb000 rw-p 00000000 00:00 0                              [anon:thread signal stack]
75fafeb000-75fafec000 ---p 00000000 00:00 0                              [anon:bionic TLS guard]
75fafec000-75fafef000 rw-p 00000000 00:00 0                              [anon:bionic TLS]
75fafef000-75faff0000 ---p 00000000 00:00 0                              [anon:bionic TLS guard]
75faff0000-75faff2000 r-xp 00000000 00:00 0                              [vdso]
75faff2000-75fb0e6000 r-xp 00000000 103:09 269                           /system/bin/linker64
75fb0e6000-75fb0e7000 rw-p 00000000 00:00 0                              [anon:arc4random data]
75fb0e7000-75fb0eb000 r--p 000f4000 103:09 269                           /system/bin/linker64
75fb0eb000-75fb0ec000 rw-p 000f8000 103:09 269                           /system/bin/linker64
75fb0ec000-75fb158000 rw-p 00000000 00:00 0
75fb158000-75fb159000 r--p 00000000 00:00 0
75fb159000-75fb160000 rw-p 00000000 00:00 0
7ffe4fe000-7ffe51f000 rw-p 00000000 00:00 0                              [stack]
```

CTS側に下記のような適当なログ出力処理を入れてCTSをビルドし、ビルドしたCTSを実行して確認してみる。


```diff
    private String readMappingAddress(String mappingName) throws Exception {
+        Log.d(TAG, "readMappingAddress called : " + mappingName);
        ParcelFileDescriptor pfd = getInstrumentation().getUiAutomation()
                .executeShellCommand("/system/bin/cat /proc/self/maps");

        String result = null;
        try (BufferedReader reader = new BufferedReader(
                new InputStreamReader(new ParcelFileDescriptor.AutoCloseInputStream(pfd)))) {
            Pattern p = Pattern.compile("^([a-f0-9]+).*" + mappingName + ".*");
            String line;

            while ((line = reader.readLine()) != null) {
                // Even after a match is found, read until the end to clean up the pipe.
                if (result != null) continue;

                Matcher m = p.matcher(line);

                if (m.matches()) {
                    result = m.group(1);
+                    Log.d(TAG, result);
                }
            }
        }
        return result;
    }
```

- ビルド
```
[AOSP root dir] $ make cts 
```

- CTS起動
```
[AOSP root dir] $ ./out/host/linux-x86/cts/android-cts/tools/cts-tradefed
```

- AsirTest#testRandomization実行
```
cts-tf > run cts -m CtsSecurityTestCases --test android.security.cts.AslrTest#testRandomization --skip-preconditions
```

出力されたlogcatファイルのログを見てみる。

```
01-01 11:58:11.349  3247  3264 D AslrTest: readMappingAddress called : \[stack\]
...
01-01 11:58:11.410  3247  3264 I BufffferedReader: 7ffadcc000-7ffaded000 rw-p 00000000 00:00 0                              [stack]
01-01 11:58:11.410  3247  3264 D AslrTest: 7ffadcc000
01-01 11:58:11.410  3247  3264 D AslrTest: readMappingAddress called : \[stack\]
...
01-01 11:58:11.455  3247  3264 I BufffferedReader: 7fdab93000-7fdabb4000 rw-p 00000000 00:00 0                              [stack]
01-01 11:58:11.455  3247  3264 D AslrTest: 7fdab93000
01-01 11:58:11.456  3247  3264 D AslrTest: readMappingAddress called : \[stack\]
...
01-01 11:58:11.503  3247  3264 I BufffferedReader: 7fe146a000-7fe148b000 rw-p 00000000 00:00 0                              [stack]
01-01 11:58:11.504  3247  3264 D AslrTest: 7fe146a000
...
```

```
01-01 11:58:32.998  3247  3264 D AslrTest: readMappingAddress called : /system/bin/
01-01 11:58:33.018  3247  3264 I BufffferedReader: 55863e7000-5586444000 r-xp 00000000 103:09 417                           /system/bin/toybox
01-01 11:58:33.018  3247  3264 D AslrTest: 55863e7000
01-01 11:58:33.018  3247  3264 I BufffferedReader: 5586444000-5586447000 r--p 0005c000 103:09 417                           /system/bin/toybox
01-01 11:58:33.018  3247  3264 I BufffferedReader: 5586447000-5586449000 rw-p 0005f000 103:09 417                           /system/bin/toybox
...
01-01 11:58:33.024  3247  3264 D AslrTest: readMappingAddress called : /system/bin/
01-01 11:58:33.047  3247  3264 I BufffferedReader: 5564d28000-5564d85000 r-xp 00000000 103:09 417                           /system/bin/toybox
01-01 11:58:33.047  3247  3264 D AslrTest: 5564d28000
01-01 11:58:33.047  3247  3264 I BufffferedReader: 5564d85000-5564d88000 r--p 0005c000 103:09 417                           /system/bin/toybox
01-01 11:58:33.047  3247  3264 I BufffferedReader: 5564d88000-5564d8a000 rw-p 0005f000 103:09 417                           /system/bin/toybox
...
01-01 11:58:33.058  3247  3264 D AslrTest: readMappingAddress called : /system/bin/
01-01 11:58:33.076  3247  3264 I BufffferedReader: 558e82f000-558e88c000 r-xp 00000000 103:09 417                           /system/bin/toybox
01-01 11:58:33.076  3247  3264 D AslrTest: 558e82f000
01-01 11:58:33.076  3247  3264 I BufffferedReader: 558e88c000-558e88f000 r--p 0005c000 103:09 417                           /system/bin/toybox
01-01 11:58:33.077  3247  3264 I BufffferedReader: 558e88f000-558e891000 rw-p 0005f000 103:09 417                           /system/bin/toybox
...
```

[stack]と/system/bin/～となってるプロセスの開始アドレスが抜き出されており、後述のcalculateEntropyBitsで何度もコールされるたびに毎回アドレスが異なることが分かる。


L90に戻ると、もし上記で一致するものがありresultで返されていれば先へ進み、なければスキップ。

一致するものがあり先へ進んだ場合、L95でcalculateEntropyBitsにmappingNameを渡しコールしている。
calculateEntropyBitsの処理を見ていく。

```java
72     private int calculateEntropyBits(String mappingName) throws Exception {
73         HashMap<String, Integer> addresses = new HashMap<String, Integer>();
74 
75         // Sufficient number of iterations to ensure we should see at least
76         // aslrMinEntropyBits
77         for (int i = 0; i < 2 * (1 << aslrMinEntropyBits); i++) {
78             addresses.put(readMappingAddress(mappingName), 1);
79         }
80 
81         double entropy = Math.log(addresses.size()) / Math.log(2);
82 
83         Log.i(TAG, String.format("%.1f", entropy) +
84             " bits of entropy for " + mappingName);
85 
86         return (int) Math.round(entropy);
87     }
```

```java
44     private static final int aslrMinEntropyBits = 8
```

なので、L77で2 * (1<<8) = 512回readMappingAddressを同じ引数mappingNameで呼び出し、返ってきたマッピング開始アドレスをキーにHashMapに追加している。

L81で下記式によりエントロピーを計算し、結果を四捨五入したものをリターンしている。

$$  
\frac{\ln{ユニークなマッピング開始アドレス数}}{\ln{2}}  
$$  


L95に戻ると、L97でentropyがaslrMinEntropyBits(=8)以上であればPASS、それ以外はFAILとなっている。

つまり
round(ln(マッピングアドレスユニーク数)/ln2) >= 8
を満たすようなマッピングアドレスユニーク数であればよい。

---

### testOneExecutableIsPie

testOneExecutableIsPieを見ていく。

```java
107     public void testOneExecutableIsPie() throws IOException {
108         assertTrue(ReadElf.read(new File("/system/bin/cat")).isPIE());
109     }
```

名前から/system/bin/cat実行ファイルのELFヘッダを読み込み、それがPIE(Position Independent Executable)であるかをチェックしているテストと推測されるが、中の処理も見ていく。

https://search.siprop.org/android-8.1.0_r1.0/xref/cts/common/util/src/com/android/compatibility/common/util/ReadElf.java

```java
508     public static ReadElf read(File file) throws IOException {
509         return new ReadElf(file);
510     }
```

```java 
590     private ReadElf(File file) throws IOException {
591         mPath = file.getPath();
592         mFile = new RandomAccessFile(file, "r");
593 
594         if (mFile.length() < EI_NIDENT) {
595             throw new IllegalArgumentException("Too small to be an ELF file: " + file);
596         }
597 
598         readHeader();
599     }
```

L592でファイルを読み込み、L594～L596でファイル長がEI_NIDENT(=16)より小さければ、ELFファイルとしては小さすぎるとしてFAIL。

L598でreadHeader()をコールしている。

```java
618     private void readHeader() throws IOException {
619         mFile.seek(0);
620         mFile.readFully(mBuffer, 0, EI_NIDENT);
621 
622         if (mBuffer[0] != ELFMAG[0]
623                 || mBuffer[1] != ELFMAG[1]
624                 || mBuffer[2] != ELFMAG[2]
625                 || mBuffer[3] != ELFMAG[3]) {
626             throw new IllegalArgumentException("Invalid ELF file: " + mPath);
627         }
628 
629         int elfClass = mBuffer[EI_CLASS];
630         if (elfClass == ELFCLASS32) {
631             mAddrSize = 4;
632         } else if (elfClass == ELFCLASS64) {
633             mAddrSize = 8;
634         } else {
635             throw new IOException("Invalid ELF EI_CLASS: " + elfClass + ": " + mPath);
636         }
637 
638         mEndian = mBuffer[EI_DATA];
639         if (mEndian == ELFDATA2LSB) {
640         } else if (mEndian == ELFDATA2MSB) {
641             throw new IOException("Unsupported ELFDATA2MSB file: " + mPath);
642         } else {
643             throw new IOException("Invalid ELF EI_DATA: " + mEndian + ": " + mPath);
644         }
645 
646         mType = readHalf();
647 
648         int e_machine = readHalf();
649         if (e_machine != EM_386
650                 && e_machine != EM_X86_64
651                 && e_machine != EM_AARCH64
652                 && e_machine != EM_ARM
653                 && e_machine != EM_MIPS
654                 && e_machine != EM_QDSP6) {
655             throw new IOException("Invalid ELF e_machine: " + e_machine + ": " + mPath);
656         }
657 
658         // AbiTest relies on us rejecting any unsupported combinations.
659         if ((e_machine == EM_386 && elfClass != ELFCLASS32)
660                 || (e_machine == EM_X86_64 && elfClass != ELFCLASS64)
661                 || (e_machine == EM_AARCH64 && elfClass != ELFCLASS64)
662                 || (e_machine == EM_ARM && elfClass != ELFCLASS32)
663                 || (e_machine == EM_QDSP6 && elfClass != ELFCLASS32)) {
664             throw new IOException(
665                     "Invalid e_machine/EI_CLASS ELF combination: "
666                             + e_machine
667                             + "/"
668                             + elfClass
669                             + ": "
670                             + mPath);
671         }
672 
673         long e_version = readWord();
674         if (e_version != EV_CURRENT) {
675             throw new IOException("Invalid e_version: " + e_version + ": " + mPath);
676         }
677 
678         long e_entry = readAddr();
679 
680         long ph_off = readOff();
681         long sh_off = readOff();
682 
683         long e_flags = readWord();
684         int e_ehsize = readHalf();
685         int e_phentsize = readHalf();
686         int e_phnum = readHalf();
687         int e_shentsize = readHalf();
688         int e_shnum = readHalf();
689         int e_shstrndx = readHalf();
690 
691         readSectionHeaders(sh_off, e_shnum, e_shentsize, e_shstrndx);
692         readProgramHeaders(ph_off, e_phnum, e_phentsize);
693     }
```

ヘーダ―読み込みの処理となっている。  
各処理の深追いはしない。

L108に戻り、メイン処理であるisPIEを見ていく。

```java
586     public boolean isPIE() {
587         return mIsPIE;
588     }
```

mIsPIEをセットしている処理を見てみると、readProgramHeaders内でセットされていることが分かる。

```java
809     private void readProgramHeaders(long ph_off, int e_phnum, int e_phentsize) throws IOException {
810         for (int i = 0; i < e_phnum; ++i) {
811             mFile.seek(ph_off + i * e_phentsize);
812 
813             long p_type = readWord();
814             if (p_type == PT_LOAD) {
815                 if (mAddrSize == 8) {
816                     // Only in Elf64_phdr; in Elf32_phdr p_flags is at the end.
817                     long p_flags = readWord();
818                 }
819                 long p_offset = readOff();
820                 long p_vaddr = readAddr();
821                 // ...
822 
823                 if (p_vaddr == 0) {
824                     mIsPIE = true;
825                 }
826             }
827         }
828     }
```

このメソッドは、先ほどのヘッダ読み込み処理内のL692でコールされている。

処理を見てみると、e_phnum(プログラムヘッダーテーブル数)分forループし、ファイルポインタをプログラムヘッダーテーブルにセットし、readAddr()で返ってきたアドレス(p_vaddr)が0であればmIsPIEがtrueとなっている。

```java
1030     private long readAddr() throws IOException {
1031         return readX(mAddrSize);
1032     }
```

```java
1034     private long readX(int byteCount) throws IOException {
1035         mFile.readFully(mBuffer, 0, byteCount);
1036 
1037         int answer = 0;
1038         if (mEndian == ELFDATA2LSB) {
1039             for (int i = byteCount - 1; i >= 0; i--) {
1040                 answer = (answer << 8) | (mBuffer[i] & 0xff);
1041             }
1042         } else {
1043             final int N = byteCount - 1;
1044             for (int i = 0; i <= N; ++i) {
1045                 answer = (answer << 8) | (mBuffer[i] & 0xff);
1046             }
1047         }
1048 
1049         return answer;
1050     }
```

---

### testVaRandomize
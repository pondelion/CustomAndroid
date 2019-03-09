8.1で追加されたSharedMemoryに関するテスト

https://android.googlesource.com/platform/cts/+/refs/tags/android-cts-8.1_r1/tests/tests/os/src/android/os/cts/SharedMemoryTest.java

テスト以下の11個

- testReadWrite
- testReadOnly
- testUseAfterClose
- testUseAfterUnmap
- testNdkInterop
- testInvalidCreate
- testInvalidMapProt
- testInvalidSetProt
- testSetProtAddProt
- testMapAfterClose
- testWriteToReadOnly

---

testReadWrite

```java
112     public void testReadWrite() throws RemoteException, ErrnoException {
113         try (SharedMemory sharedMemory = SharedMemory.create(null, 1)) {
114             ByteBuffer buffer = sharedMemory.mapReadWrite();
115             mRemote.setup(sharedMemory, PROT_READ | PROT_WRITE);
116 
117             byte expected = 5;
118             buffer.put(0, expected);
119             assertEquals(expected, buffer.get(0));
120             // Memory barrier
121             synchronized (sharedMemory) {}
122             assertEquals(expected, mRemote.read(0));
123             expected = 10;
124             mRemote.write(0, expected);
125             // Memory barrier
126             synchronized (sharedMemory) {}
127             assertEquals(expected, buffer.get(0));
128             SharedMemory.unmap(buffer);
129         }
130     }
```

```java
68     private ISharedMemoryService mRemote;
...
92     public void setUp() throws Exception {
93         mInstrumentation = InstrumentationRegistry.getInstrumentation();
94         final Context context = mInstrumentation.getContext();
95         // Bring up both remote processes and wire them to each other
96         mRemoteIntent = new Intent();
97         mRemoteIntent.setComponent(new ComponentName(
98                 "android.os.cts", "android.os.cts.SharedMemoryService"));
99         mRemoteConnection = new PeerConnection();
100         getContext().bindService(mRemoteIntent, mRemoteConnection,
101                 Context.BIND_AUTO_CREATE | Context.BIND_IMPORTANT);
102         mRemote = mRemoteConnection.get();
103     }
```

全体的な流れとしては、  
L113でSharedMemoryを作成、  
L114でShraredMemory#mapReadWriteをコールしbufferを取得、  
L118でバッファにexpectedである5をput、  
L119でバッファからget(0)で取り出し、5であることを確認、  
L122で、SharedMemoryと紐づけたSharedMomoryService(※1)とのPeerConnectionを通じてread(0)で値を取得し、5であることを確認、  
L123～124で、expectedに10を代入し、PeerConnection経由でwriteで書き込み、  
L125でbufferから直接get(0)で値を取り出し10であることを確認。  

※1 : https://search.siprop.org/android-8.1.0_r1.0/xref/cts/tests/tests/os/src/android/os/cts/SharedMemoryService.java#28
  
テスト対象機能であるSharedMemory#mapReadWriteの処理の詳細を見ていく。  

https://search.siprop.org/android-8.1.0_r1.0/xref/frameworks/base/core/java/android/os/SharedMemory.java#mapReadWrite  

```java
181     public @NonNull ByteBuffer mapReadWrite() throws ErrnoException {
182         return map(OsConstants.PROT_READ | OsConstants.PROT_WRITE, 0, mSize);
183     }
...
213     public @NonNull ByteBuffer map(int prot, int offset, int length) throws ErrnoException {
214         checkOpen();
215         validateProt(prot);
216         if (offset < 0) {
217             throw new IllegalArgumentException("Offset must be >= 0");
218         }
219         if (length <= 0) {
220             throw new IllegalArgumentException("Length must be > 0");
221         }
222         if (offset + length > mSize) {
223             throw new IllegalArgumentException("offset + length must not exceed getSize()");
224         }
225         long address = Os.mmap(0, length, prot, OsConstants.MAP_SHARED, mFileDescriptor, offset);
226         boolean readOnly = (prot & OsConstants.PROT_WRITE) == 0;
227         Runnable unmapper = new Unmapper(address, length, mMemoryRegistration.acquire());
228         return new DirectByteBuffer(length, address, mFileDescriptor, unmapper, readOnly);
229     }
```

mapReadWrite内ではmapメソッドをコールしており、map内を見るとL225のOs#mmapでメモリを確保しアドレスを取得しているとみられる。  

https://search.siprop.org/android-8.1.0_r1.0/xref/libcore/luni/src/main/java/android/system/Os.java#mmap
```java
336   public static long mmap(long address, long byteCount, int prot, int flags, FileDescriptor fd, long offset) throws ErrnoException { return Libcore.os.mmap(address, byteCount, prot, flags, fd, offset); }
```

Javaの標準ライブラリ機能Libcore.os.mmapをコール  

https://search.siprop.org/android-8.1.0_r1.0/xref/libcore/luni/src/main/java/libcore/io/Libcore.java
```java
27     public static Os rawOs = new Linux();
28 
29     /**
30      * Access to syscalls with helpful checks/guards.
31      */
32     public static Os os = new BlockGuardOs(rawOs);
```

https://search.siprop.org/android-8.1.0_r1.0/xref/libcore/luni/src/main/java/libcore/io/BlockGuardOs.java
```
40 public class BlockGuardOs extends ForwardingOs {
```

https://search.siprop.org/android-8.1.0_r1.0/xref/libcore/luni/src/main/java/libcore/io/BlockGuardOs.java
```java
40 public class BlockGuardOs extends ForwardingOs {
```

https://search.siprop.org/android-8.1.0_r1.0/xref/libcore/luni/src/main/java/libcore/io/ForwardingOs.java
```java
53     public ForwardingOs(Os os) {
54         this.os = os;
55     }
...
133     public long mmap(long address, long byteCount, int prot, int flags, FileDescriptor fd, long offset) throws ErrnoException { return os.mmap(address, byteCount, prot, flags, fd, offset); }
```

コンストラクタで渡されたOSのmmapが呼ばれている。  
コンストラクタで渡されているのは、前述のLibcore.javaをみるとLinuxクラスのインスタンスなので、Linuxクラスをみていく。  

https://search.siprop.org/android-8.1.0_r1.0/xref/libcore/luni/src/main/java/libcore/io/Linux.java
```java
124     public native long mmap(long address, long byteCount, int prot, int flags, FileDescriptor fd, long offset) throws ErrnoException;
```

ネイティブメソッドになっている。  

https://search.siprop.org/android-8.1.0_r1.0/xref/libcore/luni/src/main/native/libcore_io_Linux.cpp
```cpp
1799 static jlong Linux_mmap(JNIEnv* env, jobject, jlong address, jlong byteCount, jint prot, jint flags, jobject javaFd, jlong offset) {
1800     int fd = jniGetFDFromFileDescriptor(env, javaFd);
1801     void* suggestedPtr = reinterpret_cast<void*>(static_cast<uintptr_t>(address));
1802     void* ptr = mmap64(suggestedPtr, byteCount, prot, flags, fd, offset);
1803     if (ptr == MAP_FAILED) {
1804         throwErrnoException(env, "mmap");
1805     }
1806     return static_cast<jlong>(reinterpret_cast<uintptr_t>(ptr));
1807 }
...
2533     NATIVE_METHOD(Linux, mmap, "(JJIILjava/io/FileDescriptor;J)J"),
```

---

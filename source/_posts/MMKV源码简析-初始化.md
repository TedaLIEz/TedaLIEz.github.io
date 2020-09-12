---
title: MMKV源码简析---初始化
date: 2020-05-04 15:21:48
tags: android, mmkv
---

- [初始化](#%e5%88%9d%e5%a7%8b%e5%8c%96)
  - [MMKV.initialize](#mmkvinitialize)
- [MMKV 写](#mmkv-%e5%86%99)
  - [MMKV.mmkvWithID](#mmkvmmkvwithid)
    - [JNI getMMKVWithID](#jni-getmmkvwithid)
      - [SCOPED_LOCK](#scopedlock)
      - [mmapedKVKey 和全局映射表绑定](#mmapedkvkey-%e5%92%8c%e5%85%a8%e5%b1%80%e6%98%a0%e5%b0%84%e8%a1%a8%e7%bb%91%e5%ae%9a)
      - [MMKV 构造函数](#mmkv-%e6%9e%84%e9%80%a0%e5%87%bd%e6%95%b0)
        - [MemoryFile](#memoryfile)
        - [MMKV::loadFromFile](#mmkvloadfromfile)

## 初始化

```java
String dir = getFilesDir().getAbsolutePath() + "/mmkv";
String rootDir = MMKV.initialize(dir, new MMKV.LibLoader() {
    @Override
    public void loadLibrary(String libName) {
        ReLinker.loadLibrary(MyApplication.this, libName);
    }
}, MMKVLogLevel.LevelInfo);
Log.i("MMKV", "mmkv root: " + rootDir);

// set log level
MMKV.setLogLevel(MMKVLogLevel.LevelInfo);

// you can turn off logging
//MMKV.setLogLevel(MMKVLogLevel.LevelNone);

MMKV.registerHandler(this);
MMKV.registerContentChangeNotify(this);
```

首先从`MMKV#initialize`开始

### MMKV.initialize

```java
public static String initialize(Context context) {
    String root = context.getFilesDir().getAbsolutePath() + "/mmkv";
    MMKVLogLevel logLevel = BuildConfig.DEBUG ? MMKVLogLevel.LevelDebug : MMKVLogLevel.LevelInfo;
    return initialize(root, null, logLevel);
}

public static String initialize(String rootDir, LibLoader loader, MMKVLogLevel logLevel) {
    if (loader != null) {
        if (BuildConfig.FLAVOR.equals("SharedCpp")) {
            loader.loadLibrary("c++_shared");
        }
        loader.loadLibrary("mmkv");
    } else {
        if (BuildConfig.FLAVOR.equals("SharedCpp")) {
            System.loadLibrary("c++_shared");
        }
        System.loadLibrary("mmkv");
    }
    MMKV.rootDir = rootDir;
    jniInitialize(MMKV.rootDir, logLevel2Int(logLevel));
    return rootDir;
}
```

通过 jniInitialize 来到 JNI 层，找到`native-bridge.cpp`中的`mmkv::jniInitialize`->`MMKV::initializeMMKV(kStr, logLevel)`

```c++
void MMKV::initializeMMKV(const MMKVPath_t &rootDir, MMKVLogLevel logLevel) {
    g_currentLogLevel = logLevel;

    ThreadLock::ThreadOnce(&once_control, initialize); // #1

    g_rootDir = rootDir;
    mkPath(g_rootDir);

    MMKVInfo("root dir: " MMKV_PATH_FORMAT, g_rootDir.c_str());
}
```

ThradLocal::ThreadOnce 就是对 pthread_once 的一次封装，pthread_once 会对首次调用此函数的线程拉起一个`init_routine`调用，此处即 initialize 这个函数

```c++
void initialize() {
    g_instanceDic = new unordered_map<string, MMKV *>;
    g_instanceLock = new ThreadLock();
    g_instanceLock->initialize();

    mmkv::DEFAULT_MMAP_SIZE = mmkv::getPageSize();
    MMKVInfo("page size:%d", DEFAULT_MMAP_SIZE);
}
```

创建了一个全局 MMKV 映射表和一个全局 ThreadLock 对象，ThreadLock 就是对 pthread 的封装; 同时将系统的 PAGE_SIZE 记录在`mmkv::DEFAULT_MMAP_SIZE`中

```c++
ThreadLock::ThreadLock() {
    pthread_mutexattr_t attr;
    pthread_mutexattr_init(&attr);
    pthread_mutexattr_settype(&attr, PTHREAD_MUTEX_RECURSIVE);

    pthread_mutex_init(&m_lock, &attr);

    pthread_mutexattr_destroy(&attr);
}
```

## MMKV 写

典型调用如下:

```kotlin
val mmkv = MMKV.mmkvWithID("testKotlin")
mmkv.encode("bool", true)
println("bool = " + mmkv.decodeBool("bool"))

mmkv.encode("int", Integer.MIN_VALUE)
println("int: " + mmkv.decodeInt("int"))

mmkv.encode("long", java.lang.Long.MAX_VALUE)
println("long: " + mmkv.decodeLong("long"))
```

### MMKV.mmkvWithID

`MMKV.mmkvWithID`本质调用的是`getMMKVWithID`这个接口

```java
public static MMKV mmkvWithID(String mmapID) {
    if (rootDir == null) {
        throw new IllegalStateException("You should Call MMKV.initialize() first.");
    }
    long handle = getMMKVWithID(mmapID, SINGLE_PROCESS_MODE, null, null);
    return new MMKV(handle);
}

public static MMKV mmkvWithID(String mmapID, int mode) {
        if (rootDir == null) {
            throw new IllegalStateException("You should Call MMKV.initialize() first.");
        }

        long handle = getMMKVWithID(mmapID, mode, null, null);
        return new MMKV(handle);
    }

    // cryptKey's length <= 16
    public static MMKV mmkvWithID(String mmapID, int mode, String cryptKey) {
        if (rootDir == null) {
            throw new IllegalStateException("You should Call MMKV.initialize() first.");
        }

        long handle = getMMKVWithID(mmapID, mode, cryptKey, null);
        return new MMKV(handle);
    }

    @Nullable
    public static MMKV mmkvWithID(String mmapID, String relativePath) {
        if (rootDir == null) {
            throw new IllegalStateException("You should Call MMKV.initialize() first.");
        }

        long handle = getMMKVWithID(mmapID, SINGLE_PROCESS_MODE, null, relativePath);
        if (handle == 0) {
            return null;
        }
        return new MMKV(handle);
    }

private native static long getMMKVWithID(String mmapID, int mode, String cryptKey, String relativePath);
```

我们注意到这里有几个参数

mmapID 比较容易理解，应该是一个全局 id，联想初始化时构造的 map，应该是在那里使用。mode 这个跟跨进程模式有关，后续分开分析， cryptKey 为加密用的 key，relativePath 则是表示相对与 rootDir 的路径， 通过 getMMKVWithID 创建的 native 对象通过句柄的方式被 Java 层的 MMKV 对象持有，进行 JNI 通信

#### JNI getMMKVWithID

通过`native-bridge.cpp`找到 JNI 转发的实现`MMKV::mmkvWithID`

```c++

/**
 *
 * @param mmapID JAVA层传入的mmkvid
 * @param size MMKV::DEFAULT_MMAP_SIZE
 * @param mode 1代表单进程模式，2代表多进程模式
 * @param cryptKey Java层传入加密用的key，可以为nullptr
 * @param relativePath Java层传入的相对路径
 * @return
 */
MMKV *MMKV::mmkvWithID(const string &mmapID, int size, MMKVMode mode, string *cryptKey, string *relativePath) {

    if (mmapID.empty()) {
        return nullptr;
    }
    SCOPED_LOCK(g_instanceLock);

    auto mmapKey = mmapedKVKey(mmapID, relativePath);
    auto itr = g_instanceDic->find(mmapKey);
    if (itr != g_instanceDic->end()) {
        MMKV *kv = itr->second;
        return kv;
    }
    if (relativePath) {
        if (!isFileExist(*relativePath)) {
            if (!mkPath(*relativePath)) {
                return nullptr;
            }
        }
        MMKVInfo("prepare to load %s (id %s) from relativePath %s", mmapID.c_str(), mmapKey.c_str(),
                 relativePath->c_str());
    }
    auto kv = new MMKV(mmapID, size, mode, cryptKey, relativePath);
    (*g_instanceDic)[mmapKey] = kv;
    return kv;
}
```

##### SCOPED_LOCK

`SCOPED_LOCK`是一个宏展开后，得到是一个 ScopedLock 对象，通过 RAII 的方式实现在函数内上锁, 其中用到了一个特殊宏`__COUNTER__`

```c++
#define SCOPED_LOCK(lock) _SCOPEDLOCK(lock, __COUNTER__)
#define _SCOPEDLOCK(lock, counter) __SCOPEDLOCK(lock, counter)
#define __SCOPEDLOCK(lock, counter)                                                                                    \
    mmkv::ScopedLock<std::remove_pointer<decltype(lock)>::type> __scopedLock##counter(lock)
```

```
__COUNTER__
    Defined to an integer value that starts at zero and is incremented each time the __COUNTER__ macro is expanded.
```

OK，每次展开自动+1, 那么使用这种方式定义就做到了每个 ScopedLock 对象变量不重名

P.S `remove_pointer`

```
Provides the member typedef type which is the type pointed to by T, or, if T is not a pointer, then type is the same as T.
```

简单来说就是拿到指针的原类型，例如`int*`->`int`

##### mmapedKVKey 和全局映射表绑定

接着上文分析，查看 mmapedKVKey 得知 MMKV 根据 mmapID 和 relativePath 的 md5，生成了一个 id，根据这个 id 在全局 g_instanceDic 进行了 putIfAbsent 操作，同时返回 new 出来的对象

##### MMKV 构造函数

```c++
MMKV::MMKV(const string &mmapID, int size, MMKVMode mode, string *cryptKey, string *relativePath)
    : m_mmapID(mmapedKVKey(mmapID, relativePath)) // historically Android mistakenly use mmapKey as mmapID
    , m_path(mappedKVPathWithID(m_mmapID, mode, relativePath))
    , m_crcPath(crcPathWithID(m_mmapID, mode, relativePath))
    , m_file(new MemoryFile(m_path, size, (mode & MMKV_ASHMEM) ? MMFILE_TYPE_ASHMEM : MMFILE_TYPE_FILE))
    , m_metaFile(new MemoryFile(m_crcPath, DEFAULT_MMAP_SIZE, m_file->m_fileType))
    , m_metaInfo(new MMKVMetaInfo())
    , m_crypter(nullptr)
    , m_lock(new ThreadLock())
    , m_fileLock(new FileLock(m_metaFile->getFd(), (mode & MMKV_ASHMEM)))
    , m_sharedProcessLock(new InterProcessLock(m_fileLock, SharedLockType))
    , m_exclusiveProcessLock(new InterProcessLock(m_fileLock, ExclusiveLockType))
    , m_isInterProcess((mode & MMKV_MULTI_PROCESS) != 0 || (mode & CONTEXT_MODE_MULTI_PROCESS) != 0) {
    m_actualSize = 0;
    m_output = nullptr;

    if (cryptKey && cryptKey->length() > 0) {
        m_crypter = new AESCrypt(cryptKey->data(), cryptKey->length());
    }

    m_needLoadFromFile = true;
    m_hasFullWriteback = false;

    m_crcDigest = 0;

    m_sharedProcessLock->m_enable = m_isInterProcess;
    m_exclusiveProcessLock->m_enable = m_isInterProcess;

    // sensitive zone
    {
        SCOPED_LOCK(m_sharedProcessLock);
        loadFromFile();
    }
}
```

一段段看

m_mmapID 应该和 mmap 中使用的 id 有关，这个值目前看应该和[mmapedKVKey 和全局映射表绑定]中提到的 id 一致

`m_path`通过`mappedKVPathWithID`得到

```c++
MMKVPath_t mappedKVPathWithID(const string &mmapID, MMKVMode mode, MMKVPath_t *relativePath) {
#ifndef MMKV_ANDROID
    if (relativePath) {
#else
    if (mode & MMKV_ASHMEM) {
        return ashmemMMKVPathWithID(encodeFilePath(mmapID));
    } else if (relativePath) {
#endif
        return *relativePath + MMKV_PATH_SLASH + encodeFilePath(mmapID);
    }
    return g_rootDir + MMKV_PATH_SLASH + encodeFilePath(mmapID);
}
```

看来跟匿名内存模式有关，做了一层 Ashmem 的兼容

`m_crcPath`则类似 mappedKVPathWithID 的实现，只是对路径名做了特殊符号 md5 后，再加上了一个.crc 后缀, 暂时还不知道什么作用，先放着

`m_file`则是一个 MemoryFile 对象，`m_metaFile`是一个以`m_crcPath`为路径的 MemoryFile

`m_metaInfo`是一个 MMKVMetaInfo 结构体，记录了一些元数据，等到读写相关信息的时候再看，`m_lock`，`m_fileLock`，`m_sharedProcessLock`，`m_exclusiveProcessLock`均为锁对象

构造器根据传入的 cryptKey 创建了 AESCrypt 对象, 同时进行`loadFromFile`调用

###### MemoryFile

首先来看看`MemoryFile`这个类

```c++
MemoryFile::MemoryFile(const string &path, size_t size, FileType fileType)
    : m_name(path), m_fd(-1), m_ptr(nullptr), m_size(0), m_fileType(fileType) {
    if (m_fileType == MMFILE_TYPE_FILE) {
        reloadFromFile();
    } else {
        // round up to (n * pagesize)
        if (size < DEFAULT_MMAP_SIZE || (size % DEFAULT_MMAP_SIZE != 0)) {
            size = ((size / DEFAULT_MMAP_SIZE) + 1) * DEFAULT_MMAP_SIZE;
        }
        auto filename = m_name.c_str();
        auto ptr = strstr(filename, ASHMEM_NAME_DEF);
        if (ptr && ptr[sizeof(ASHMEM_NAME_DEF) - 1] == '/') {
            filename = ptr + sizeof(ASHMEM_NAME_DEF);
        }
        m_fd = ASharedMemory_create(filename, size);
        if (m_fd >= 0) {
            m_size = size;
            auto ret = mmap();
            if (!ret) {
                doCleanMemoryCache(true);
            }
        }
    }
}
```

这里对 FileType 做了区分，如果是`MMFILE_TYPE_FILE`，则是走文件 IO，否则，则是 ASHMEM 匿名内存文件，我们这里分析下文件 IO 的情况

- MMFILE_TYPE_FILE

```c++
void MemoryFile::reloadFromFile() {
    if (m_fileType == MMFILE_TYPE_ASHMEM) {
        return;
    }
    if (isFileValid()) {
        MMKVWarning("calling reloadFromFile while the cache [%s] is still valid", m_name.c_str());
        MMKV_ASSERT(0);
        clearMemoryCache();
    }

    m_fd = open(m_name.c_str(), O_RDWR | O_CREAT | O_CLOEXEC, S_IRWXU);
    if (m_fd < 0) {
        MMKVError("fail to open:%s, %s", m_name.c_str(), strerror(errno));
    } else {
        FileLock fileLock(m_fd);
        InterProcessLock lock(&fileLock, ExclusiveLockType);
        SCOPED_LOCK(&lock);

        mmkv::getFileSize(m_fd, m_size);
        // round up to (n * pagesize)
        if (m_size < DEFAULT_MMAP_SIZE || (m_size % DEFAULT_MMAP_SIZE != 0)) {
            size_t roundSize = ((m_size / DEFAULT_MMAP_SIZE) + 1) * DEFAULT_MMAP_SIZE;
            truncate(roundSize);
        } else {
            auto ret = mmap();
            if (!ret) {
                doCleanMemoryCache(true);
            }
        }
        // iOS spec code...
    }
}
```

这个函数做了几件事情

1.  打开路径为`m_name`的文件，记录于描述符`m_fd`
2.  将文件对齐为 DEFAULT_PMM_SIZE 的整数倍大小
3.  mmap 打开内存映射，映射到 m_ptr

- MMFILE_TYPE_ASHMEM

如果是匿名内存模式，执行逻辑类似，只是创建文件的实现换成了 ASharedMemory_create

至此，MemoryFile 初始化过程结束，回到 MMKV 的构造器中调用的 loadFromFile 函数

###### MMKV::loadFromFile

```c++
void MMKV::loadFromFile() {
    if (m_metaFile->isFileValid()) {
        m_metaInfo->read(m_metaFile->getMemory()); // 读元数据文件到内存对象m_metaInfo中
    }
    if (m_crypter) {
        if (m_metaInfo->m_version >= MMKVVersionRandomIV) {
            m_crypter->resetIV(m_metaInfo->m_vector, sizeof(m_metaInfo->m_vector));
        }
    }

    if (!m_file->isFileValid()) {
        m_file->reloadFromFile();
    }
    if (!m_file->isFileValid()) {
        MMKVError("file [%s] not valid", m_path.c_str());
    } else {
        // error checking
        bool loadFromFile = false, needFullWriteback = false;
        checkDataValid(loadFromFile, needFullWriteback);  // #1
        MMKVInfo("loading [%s] with %zu actual size, file size %zu, InterProcess %d, meta info "
                 "version:%u",
                 m_mmapID.c_str(), m_actualSize, m_file->getFileSize(), m_isInterProcess, m_metaInfo->m_version);
        auto ptr = (uint8_t *) m_file->getMemory();
        // loading
        if (loadFromFile && m_actualSize > 0) {
            MMKVInfo("loading [%s] with crc %u sequence %u version %u", m_mmapID.c_str(), m_metaInfo->m_crcDigest,
                     m_metaInfo->m_sequence, m_metaInfo->m_version);
            MMBuffer inputBuffer(ptr + Fixed32Size, m_actualSize, MMBufferNoCopy);
            if (m_crypter) {
                decryptBuffer(*m_crypter, inputBuffer);
            }
            clearDictionary(m_dic);
            if (needFullWriteback) {
                MiniPBCoder::greedyDecodeMap(m_dic, inputBuffer);
            } else {
                MiniPBCoder::decodeMap(m_dic, inputBuffer);
            }
            m_output = new CodedOutputData(ptr + Fixed32Size, m_file->getFileSize() - Fixed32Size);
            m_output->seek(m_actualSize);
            if (needFullWriteback) {
                fullWriteback(); // #2 全量写回文件
            }
        } else {
            // file not valid or empty, discard everything
            SCOPED_LOCK(m_exclusiveProcessLock);

            m_output = new CodedOutputData(ptr + Fixed32Size, m_file->getFileSize() - Fixed32Size);
            if (m_actualSize > 0) {
                writeActualSize(0, 0, nullptr, IncreaseSequence);
                sync(MMKV_SYNC);
            } else {
                writeActualSize(0, 0, nullptr, KeepSequence);
            }
        }
        MMKVInfo("loaded [%s] with %zu values", m_mmapID.c_str(), m_dic.size());
    }

    m_needLoadFromFile = false;
}

// MMKV初始化时, loadFromFile=false, needFullWriteback=false, m_metaInfo->m_version=MMKVVersionSequence
void MMKV::checkDataValid(bool &loadFromFile, bool &needFullWriteback) {
    // try auto recover from last confirmed location
    auto fileSize = m_file->getFileSize();
    auto checkLastConfirmedInfo = [&] {
        if (m_metaInfo->m_version >= MMKVVersionActualSize) {
            // downgrade & upgrade support
            uint32_t oldStyleActualSize = 0;
            memcpy(&oldStyleActualSize, m_file->getMemory(), Fixed32Size);
            if (oldStyleActualSize != m_actualSize) {
                MMKVWarning("oldStyleActualSize %u not equal to meta actual size %lu", oldStyleActualSize,
                            m_actualSize);
                if (oldStyleActualSize < fileSize && (oldStyleActualSize + Fixed32Size) <= fileSize) {
                    if (checkFileCRCValid(oldStyleActualSize, m_metaInfo->m_crcDigest)) {
                        MMKVInfo("looks like [%s] been downgrade & upgrade again", m_mmapID.c_str());
                        loadFromFile = true;
                        writeActualSize(oldStyleActualSize, m_metaInfo->m_crcDigest, nullptr, KeepSequence);
                        return;
                    }
                } else {
                    MMKVWarning("oldStyleActualSize %u greater than file size %lu", oldStyleActualSize, fileSize);
                }
            }

            auto lastActualSize = m_metaInfo->m_lastConfirmedMetaInfo.lastActualSize;
            if (lastActualSize < fileSize && (lastActualSize + Fixed32Size) <= fileSize) {
                auto lastCRCDigest = m_metaInfo->m_lastConfirmedMetaInfo.lastCRCDigest;
                if (checkFileCRCValid(lastActualSize, lastCRCDigest)) {
                    loadFromFile = true;
                    writeActualSize(lastActualSize, lastCRCDigest, nullptr, KeepSequence);
                } else {
                    MMKVError("check [%s] error: lastActualSize %u, lastActualCRC %u", m_mmapID.c_str(), lastActualSize,
                              lastCRCDigest);
                }
            } else {
                MMKVError("check [%s] error: lastActualSize %u, file size is %u", m_mmapID.c_str(), lastActualSize,
                          fileSize);
            }
        }
    };

    m_actualSize = readActualSize();

    if (m_actualSize < fileSize && (m_actualSize + Fixed32Size) <= fileSize) {
        if (checkFileCRCValid(m_actualSize, m_metaInfo->m_crcDigest)) {
            loadFromFile = true;
        } else {
            checkLastConfirmedInfo();

            if (!loadFromFile) {
                auto strategic = onMMKVCRCCheckFail(m_mmapID);
                if (strategic == OnErrorRecover) {
                    loadFromFile = true;
                    needFullWriteback = true;
                }
                MMKVInfo("recover strategic for [%s] is %d", m_mmapID.c_str(), strategic);
            }
        }
    } else {
        MMKVError("check [%s] error: %zu size in total, file size is %zu", m_mmapID.c_str(), m_actualSize, fileSize);

        checkLastConfirmedInfo();

        if (!loadFromFile) {
            auto strategic = onMMKVFileLengthError(m_mmapID);
            if (strategic == OnErrorRecover) {
                // make sure we don't over read the file
                m_actualSize = fileSize - Fixed32Size;
                loadFromFile = true;
                needFullWriteback = true;
            }
            MMKVInfo("recover strategic for [%s] is %d", m_mmapID.c_str(), strategic);
        }
    }
}

size_t MMKV::readActualSize() {
    MMKV_ASSERT(m_file->getMemory());
    MMKV_ASSERT(m_metaFile->isFileValid());

    uint32_t actualSize = 0;
    memcpy(&actualSize, m_file->getMemory(), Fixed32Size);

    if (m_metaInfo->m_version >= MMKVVersionActualSize) {
        if (m_metaInfo->m_actualSize != actualSize) {
            MMKVWarning("[%s] actual size %u, meta actual size %u", m_mmapID.c_str(), actualSize,
                        m_metaInfo->m_actualSize);
        }
        return m_metaInfo->m_actualSize;
    } else {
        return actualSize;
    }
}
```

从 readActualSize 的实现中，我们似乎可以看出，m_file 似乎是有一个 header 结构，这个结构的第一个 int32 表示了文件的实际大小，那么`m_actualSize < fileSize && (m_actualSize + Fixed32Size) <= fileSize`这个判断其实表达的意思是除去这个 int32 之外的信息有没有被写入到文件。fileSize 就是 MemoryFile 中传入的 size 对齐 PAGE_SIZE 后的值

走到`checkFileCRCValid`逻辑，可以看出这里实际上是使用了 CRC 对存储内容进行了校验, 如果校验成功，则置 loadFromFile=true

我们初始化时不需要数据写回，因此调用 checkDataValid 返回后`loadFromFile=true， needFullWriteback=false`
走到 else 分支

```c++
SCOPED_LOCK(m_exclusiveProcessLock);

m_output = new CodedOutputData(ptr + Fixed32Size, m_file->getFileSize() - Fixed32Size);
if (m_actualSize > 0) {
    writeActualSize(0, 0, nullptr, IncreaseSequence);
    sync(MMKV_SYNC);
} else {
    writeActualSize(0, 0, nullptr, KeepSequence);
}

// 初始化阶段increaseSequence=false,size=0, crcDigest=0, iv=nullptr
bool MMKV::writeActualSize(size_t size, uint32_t crcDigest, const void *iv, bool increaseSequence) {
    // backward compatibility
    oldStyleWriteActualSize(size);

    if (!m_metaFile->isFileValid()) {
        return false;
    }

    bool needsFullWrite = false;
    m_actualSize = size;
    m_metaInfo->m_actualSize = static_cast<uint32_t>(size);
    m_crcDigest = crcDigest;
    m_metaInfo->m_crcDigest = crcDigest;
    if (m_metaInfo->m_version < MMKVVersionSequence) {
        // 初始化不会走到这里
        m_metaInfo->m_version = MMKVVersionSequence;
        needsFullWrite = true;
    }
    if (unlikely(iv)) {
        // 初始化时走这个分支，m_version改为MMKVVersionRandomIV
        memcpy(m_metaInfo->m_vector, iv, sizeof(m_metaInfo->m_vector));
        if (m_metaInfo->m_version < MMKVVersionRandomIV) {
            m_metaInfo->m_version = MMKVVersionRandomIV;
        }
        needsFullWrite = true;
    }
    if (unlikely(increaseSequence)) {
        // 初始化时走这个分支，m_version改为MMKVVersionActualSize
        m_metaInfo->m_sequence++;
        m_metaInfo->m_lastConfirmedMetaInfo.lastActualSize = static_cast<uint32_t>(size);
        m_metaInfo->m_lastConfirmedMetaInfo.lastCRCDigest = crcDigest;
        if (m_metaInfo->m_version < MMKVVersionActualSize) {
            m_metaInfo->m_version = MMKVVersionActualSize;
        }
        needsFullWrite = true;
    }
    // iOS spec code ...
    if (unlikely(needsFullWrite)) {
        m_metaInfo->write(m_metaFile->getMemory());
    } else {
        m_metaInfo->writeCRCAndActualSizeOnly(m_metaFile->getMemory());
    }
    return true;
}
```

writeActualSize 在初始化阶段做了几件事情

1. m_metaInfo->m_actualSize=0
2. m_metaInfo->m_crcDigest=0
3. m_metaInfo->m_version=MMKVVersionActualSize
4. m_metaInfo->m_sequence++;
5. m_metaInfo->m_lastConfirmedMetaInfo.lastActualSize = static_cast<uint32_t>(0);
6. m_metaInfo->m_lastConfirmedMetaInfo.lastCRCDigest = 0;

将上述信息的修改全部写入 m_metaFile 持有的指针中，同时利用 MemoryFile 中这个指针是由 mmap 创建的特性，同步这些信息到文件描述符中

至此，MMKV 的初始化过程分析完了，总结一下

1. `m_mmapID`相当于是 MMKV 自己的唯一 id，用于全局变量`g_instanceDic`进行内存管理
2. `m_path`表达了 MMKV 对应的文件路径，用于创建对象 m_file
3. `m_file`描述了 MMKV 对应的文件，这个文件可以是 IO 的，也可以是匿名内存的
4. `m_crcPath`表达了 m_path 这个路径对应的 CRC 校验文件路径，用于创建对象 m_metaFile
5. `m_metaFile`创建了 m_crcPath 对应的文件
6. `m_metaInfo`是一个元数据结构体，相当于是`m_metaFile`的一份缓存

`m_file`和`m_metaFile`均为`MemoryFile`对象，这个对象封装了文件描述符，同时构造了一个基于此文件描述符的 mmap 后的内存指针`MemoryFile#m_ptr`, 通过对这个指针的读写，我们可以快速得到 IO 中的内容

初始化阶段，构造函数构造了上述变量的同时，将`m_actualSize`和`m_crcDigest`清 0，同时`m_version=MMKVVersionActualSize`

先写这么多，下一篇文章来分析一下 MMKV 的读写是如何实现的

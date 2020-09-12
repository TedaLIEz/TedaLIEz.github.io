---
title: MMKV源码简析---读写
date: 2020-05-04 15:22:29
tags: android, mmkv
---

- [MMKV 写](#mmkv-%e5%86%99)
  - [checkLoadData()](#checkloaddata)
  - [appendDataWithKey](#appenddatawithkey)
- [MMKV 读](#mmkv-%e8%af%bb)
- [小结](#%e5%b0%8f%e7%bb%93)
- [MMKV 空间优化](#mmkv-%e7%a9%ba%e9%97%b4%e4%bc%98%e5%8c%96)

我们还是从 testcase 出发，一点点看 MMKV 如何实现读写的

## MMKV 写

```kotlin
val mmkv = MMKV.mmkvWithID("testKotlin")
mmkv.encode("bool", true)
mmkv.encode("int", Integer.MIN_VALUE)
```

MMKV#encode 支持所有基本类型, 这里我们挑一个简单的 bool 类型分析一下，根据调用链`MMKV#encode`->`MMKV#encodeBool`来到 native-bridge.cpp 中的 JNI 接口，找到其实现`MMKV::set(bool value, MMKVKey_t key)`

```c++
// MMKVKey_t = const string&
bool MMKV::set(bool value, MMKVKey_t key) {
    if (isKeyEmpty(key)) {
        return false;
    }
    size_t size = pbBoolSize();
    MMBuffer data(size);
    CodedOutputData output(data.getPtr(), size);
    output.writeBool(value);

    return setDataForKey(std::move(data), key);
}
```

这里根据 pb 协议的规范，分配了一块和 pb 协议中商定的类型字段长度相同的 buffer MMBuffer, 将这个 buffer 利用 CodedOutputData 这个工具类写入一个 bool，之后调用`setDataForKey`来记录

```c++
bool MMKV::setDataForKey(MMBuffer &&data, MMKVKey_t key) {
    if (data.length() == 0 || isKeyEmpty(key)) {
        return false;
    }
    SCOPED_LOCK(m_lock);
    SCOPED_LOCK(m_exclusiveProcessLock);
    checkLoadData(); // #1

    auto ret = appendDataWithKey(data, key); // #2
    if (ret) {
        m_dic[key] = std::move(data);
        m_hasFullWriteback = false;
    }
    return ret;
}

```

### checkLoadData()

```c++
void MMKV::checkLoadData() {
    if (m_needLoadFromFile) { // m_needLoadFromFile在MMKV构造和clearMemoryCache调用后才会是true
        SCOPED_LOCK(m_sharedProcessLock);

        m_needLoadFromFile = false;
        loadFromFile(); // 初始化过程中也调用了这个函数，相当于加载存储中的内容
        return;
    }
    if (!m_isInterProcess) {
        // 不是跨进程模式的MMKV，直接返回
        return;
    }

    if (!m_metaFile->isFileValid()) {
        return;
    }
    // TODO: atomic lock m_metaFile?
    MMKVMetaInfo metaInfo;
    metaInfo.read(m_metaFile->getMemory());
    if (m_metaInfo->m_sequence != metaInfo.m_sequence) { // 内存中的seq和文件中的seq不一致，文件有新的改动，清理缓存，重新读文件，通知内容更改
        MMKVInfo("[%s] oldSeq %u, newSeq %u", m_mmapID.c_str(), m_metaInfo->m_sequence, metaInfo.m_sequence);
        SCOPED_LOCK(m_sharedProcessLock);

        clearMemoryCache();
        loadFromFile();
        notifyContentChanged();
    } else if (m_metaInfo->m_crcDigest != metaInfo.m_crcDigest) { // 内存中的crc校验值和文件中crc校验值不一致
        MMKVDebug("[%s] oldCrc %u, newCrc %u, new actualSize %u", m_mmapID.c_str(), m_metaInfo->m_crcDigest,
                  metaInfo.m_crcDigest, metaInfo.m_actualSize);
        SCOPED_LOCK(m_sharedProcessLock);

        size_t fileSize = m_file->getActualFileSize();
        if (m_file->getFileSize() != fileSize) {
            // 有新内容写入，清理memory，重新读文件，通知内容更改
            MMKVInfo("file size has changed [%s] from %zu to %zu", m_mmapID.c_str(), m_file->getFileSize(), fileSize);
            clearMemoryCache();
            loadFromFile();
        } else {
            // 现有内容有变化，存量加载
            partialLoadFromFile();
        }
        notifyContentChanged();  // 通知Java层通过setWantsContentChangeNotify注册的回调
    }
}

```

逻辑比较清晰，主要看看 partialLoadFromFile 这个函数

```c++
// read from last m_position
void MMKV::partialLoadFromFile() {
    m_metaInfo->read(m_metaFile->getMemory());

    size_t oldActualSize = m_actualSize;
    m_actualSize = readActualSize();
    auto fileSize = m_file->getFileSize();
    MMKVDebug("loading [%s] with file size %zu, oldActualSize %zu, newActualSize %zu", m_mmapID.c_str(), fileSize,
              oldActualSize, m_actualSize);

    if (m_actualSize > 0) {
        if (m_actualSize < fileSize && m_actualSize + Fixed32Size <= fileSize) {
            if (m_actualSize > oldActualSize) {
                // 有新内容
                size_t bufferSize = m_actualSize - oldActualSize;
                auto ptr = (uint8_t *) m_file->getMemory();
                MMBuffer inputBuffer(ptr + Fixed32Size + oldActualSize, bufferSize, MMBufferNoCopy);
                // incremental update crc digest
                m_crcDigest =
                    (uint32_t) CRC32(m_crcDigest, (const uint8_t *) inputBuffer.getPtr(), inputBuffer.length());
                if (m_crcDigest == m_metaInfo->m_crcDigest) {
                    if (m_crypter) {
                        decryptBuffer(*m_crypter, inputBuffer);
                    }
                    MiniPBCoder::greedyDecodeMap(m_dic, inputBuffer, bufferSize); // 解析buffer内容到m_dic, buffer的结构是线性的string-MMBuffer KV结构
                    m_output->seek(bufferSize);
                    m_hasFullWriteback = false;

                    MMKVDebug("partial loaded [%s] with %zu values", m_mmapID.c_str(), m_dic.size());
                    return;
                } else {
                    MMKVError("m_crcDigest[%u] != m_metaInfo->m_crcDigest[%u]", m_crcDigest, m_metaInfo->m_crcDigest);
                }
            }
        }
    }
    // something is wrong, do a full load
    clearMemoryCache();
    loadFromFile();
}

```

### appendDataWithKey

```c++
bool MMKV::appendDataWithKey(const MMBuffer &data, MMKVKey_t key) {
    size_t keyLength = key.length();
    // size needed to encode the key
    size_t size = keyLength + pbRawVarint32Size((int32_t) keyLength);
    // size needed to encode the value
    size += data.length() + pbRawVarint32Size((int32_t) data.length());

    SCOPED_LOCK(m_exclusiveProcessLock);

    bool hasEnoughSize = ensureMemorySize(size);
    if (!hasEnoughSize || !isFileValid()) {
        return false;
    }
    m_output->writeString(key);
    m_output->writeData(data); // note: write size of data

    auto ptr = (uint8_t *) m_file->getMemory() + Fixed32Size + m_actualSize;
    if (m_crypter) {
        m_crypter->encrypt(ptr, ptr, size);
    }
    m_actualSize += size;
    updateCRCDigest(ptr, size); // 更新crc校验和seq

    return true;
}
```

## MMKV 读

同样地，通过 JNI 找到`MMKV::getBool`

```c++
bool MMKV::getBool(MMKVKey_t key, bool defaultValue) {
    if (isKeyEmpty(key)) {
        return defaultValue;
    }
    SCOPED_LOCK(m_lock);
    auto &data = getDataForKey(key); // #1
    if (data.length() > 0) {
        try {
            CodedInputData input(data.getPtr(), data.length());
            return input.readBool();
        } catch (std::exception &exception) {
            MMKVError("%s", exception.what());
        }
    }
    return defaultValue;
}
```

实现关键在 getDataForKey

```c++
const MMBuffer &MMKV::getDataForKey(MMKVKey_t key) {
    checkLoadData();
    auto itr = m_dic.find(key);
    if (itr != m_dic.end()) {
        return itr->second;
    }
    static MMBuffer nan;
    return nan;
}
```

checkLoadData 函数通过调用 loadFromFile 会确保所有的文件数据被读进 m_dic 中，这里直接取就可以了

## 小结

MMKV 的读写逻辑比较简单，主要的实现还是依赖了 PB 方式的数据序列化和反序列化，根据代码逻辑，我们可以尝试还原出 MMKV 内部的存储结构如下:

```protocol buffer
message KV {
	string key = 1;
	buffer value = 2;
}

message MMKV {
    int32 size = 1; // 文件大小，用于校验新数据写入
    repeated KV kv = 2;
}
```

同时我们可以通过`MiniPBCoder::decodeOneMap`的代码实现可以发现，MMKV 对于新旧数据问题，采用的方式不是写覆盖，而是直接追加，同时读时以最后一次写为最新值。这种方式显然是会带来大量的 key 字段冗余，因此必然存在一套完整的空间优化

## MMKV 空间优化

核心实现在`MMKV::ensureMemorySize`中

```c++
bool MMKV::ensureMemorySize(size_t newSize) {
    if (!isFileValid()) {
        MMKVWarning("[%s] file not valid", m_mmapID.c_str());
        return false;
    }

    // make some room for placeholder
    constexpr size_t ItemSizeHolderSize = 4;
    if (m_dic.empty()) {
        newSize += ItemSizeHolderSize;
    }
    if (newSize >= m_output->spaceLeft() || m_dic.empty()) {
        // 空间不够，进行一次全量写入，这次全量写入会把过长的冗余字段全部覆盖写
        // try a full rewrite to make space
        auto fileSize = m_file->getFileSize();
        MMBuffer data = MiniPBCoder::encodeDataWithObject(m_dic);
        size_t lenNeeded = data.length() + Fixed32Size + newSize;
        size_t avgItemSize = lenNeeded / std::max<size_t>(1, m_dic.size());
        size_t futureUsage = avgItemSize * std::max<size_t>(8, (m_dic.size() + 1) / 2);
        // 1. no space for a full rewrite, double it
        // 2. or space is not large enough for future usage, double it to avoid frequently full rewrite
        if (lenNeeded >= fileSize || (lenNeeded + futureUsage) >= fileSize) {
            size_t oldSize = fileSize;
            do {
                fileSize *= 2; // 2倍的扩充策略
            } while (lenNeeded + futureUsage >= fileSize);
            MMKVInfo("extending [%s] file size from %zu to %zu, incoming size:%zu, future usage:%zu", m_mmapID.c_str(),
                     oldSize, fileSize, newSize, futureUsage);

            // if we can't extend size, rollback to old state
            if (!m_file->truncate(fileSize)) {
                return false;
            }

            // check if we fail to make more space
            if (!isFileValid()) {
                MMKVWarning("[%s] file not valid", m_mmapID.c_str());
                return false;
            }
        }
        return doFullWriteBack(std::move(data)); // 全量写入
    }
    return true;
}
```

整个读写过程分析到这里，后面将分析一下 MMKV 对于跨进程存储的实现

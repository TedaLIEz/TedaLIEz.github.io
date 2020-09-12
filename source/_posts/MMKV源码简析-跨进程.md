---
title: MMKV源码简析---跨进程
date: 2020-05-04 15:23:07
tags: android, mmkv
---

MMKV 一个重要特性就是增加了 android 侧对跨进程读写的支持，我们单独用一篇文章来分析一下 MMKV 对跨进程存储的实现方式
典型的初始化调用如下:

```java
MMKV mmkv = MMKV.mmkvWithID(MMKV_ID, MMKV.MULTI_PROCESS_MODE, CryptKey);

mmkv.encode(...)
mmkv.decode(...)
```

MMKV 会把每个文件都通过 mmap 到进程的访问空间，因此每个进程本身就可以直接访问存储，但这里没有解决跨进程的并发访问问题，解决并发需要别的手段，我们看看 MMKV.MULTI_PROCESS_MODE 造成了什么影响

## MMKV.MULTI_PROCESS_MODE

参考初始化流程的代码和`MMKV.MULTI_PROCESS_MODE=2`, 来到 MMKV 构造函数, 可以发现此时 MMKV::m_isInterProcess=true, 这个对代码逻辑有什么影响呢?

1. checkLoadData 的后半段代码会执行
2. m_sharedProcessLock, m_exclusiveProcessLock 全部 enable

先来看看 checkLoadData 的后半段代码是啥

```c++
if (!m_isInterProcess) {
    return;
}

if (!m_metaFile->isFileValid()) {
    return;
}
// TODO: atomic lock m_metaFile?
MMKVMetaInfo metaInfo;
metaInfo.read(m_metaFile->getMemory()); // 文件m_metaFile被mmap，每个进程都能读到, 但是m_metaInfo是内存对象，没有被进程共享，因此有可能存储的元数据和文件中的元数据不一致
if (m_metaInfo->m_sequence != metaInfo.m_sequence) {
    MMKVInfo("[%s] oldSeq %u, newSeq %u", m_mmapID.c_str(), m_metaInfo->m_sequence, metaInfo.m_sequence);
    SCOPED_LOCK(m_sharedProcessLock);

    clearMemoryCache();
    loadFromFile();
    notifyContentChanged();
} else if (m_metaInfo->m_crcDigest != metaInfo.m_crcDigest) {
    MMKVDebug("[%s] oldCrc %u, newCrc %u, new actualSize %u", m_mmapID.c_str(), m_metaInfo->m_crcDigest,
                metaInfo.m_crcDigest, metaInfo.m_actualSize);
    SCOPED_LOCK(m_sharedProcessLock);

    size_t fileSize = m_file->getActualFileSize();
    if (m_file->getFileSize() != fileSize) {
        MMKVInfo("file size has changed [%s] from %zu to %zu", m_mmapID.c_str(), m_file->getFileSize(), fileSize);
        clearMemoryCache();
        loadFromFile();
    } else {
        partialLoadFromFile();
    }
    notifyContentChanged();
}
```

相比较于单进程，这里增加了对内存中的 meta 和文件的 meta 的 seq 和 crc 校验比较，如果不相等，则需要走文件重新加载逻辑，这一块，和[https://github.com/Tencent/MMKV/wiki/android_ipc#%E7%8A%B6%E6%80%81%E5%90%8C%E6%AD%A5](https://github.com/Tencent/MMKV/wiki/android_ipc#%E7%8A%B6%E6%80%81%E5%90%8C%E6%AD%A5)描述是一致的

m_sharedProcessLock, m_exclusiveProcessLock 全部 enable 应该就是使锁生效，那么分析一下这个锁是如何实现跨进程锁的

## InterProcessLock

```c++
class InterProcessLock {
    FileLock *m_fileLock;
    LockType m_lockType;

public:
    InterProcessLock(FileLock *fileLock, LockType lockType)
        : m_fileLock(fileLock), m_lockType(lockType), m_enable(true) {
        MMKV_ASSERT(m_fileLock);
    }

    bool m_enable;

    void lock() {
        if (m_enable) {
            m_fileLock->lock(m_lockType);
        }
    }
// ...
}

```

可以看到，MMKV 的跨进程锁是基于文件锁实现的, `InterProcessLock#lock`基于`FileLock.lock`实现, 一路转发来到`FileLock::platformLock()`

```c++

bool FileLock::doLock(LockType lockType, bool wait) {
    if (!isFileLockValid()) {
        return false;
    }
    bool unLockFirstIfNeeded = false;

    if (lockType == SharedLockType) {
        // don't want shared-lock to break any existing locks
        if (m_sharedLockCount > 0 || m_exclusiveLockCount > 0) {
            m_sharedLockCount++;
            return true;
        }
    } else {
        // don't want exclusive-lock to break existing exclusive-locks
        if (m_exclusiveLockCount > 0) {
            m_exclusiveLockCount++;
            return true;
        }
        // prevent deadlock
        if (m_sharedLockCount > 0) {
            unLockFirstIfNeeded = true;
        }
    }

    auto ret = platformLock(lockType, wait, unLockFirstIfNeeded);
    if (ret) {
        if (lockType == SharedLockType) {
            m_sharedLockCount++;
        } else {
            m_exclusiveLockCount++;
        }
    }
    return ret;
}

bool FileLock::platformLock(LockType lockType, bool wait, bool unLockFirstIfNeeded) {
#    ifdef MMKV_ANDROID
    if (m_isAshmem) {
        return ashmemLock(lockType, wait, unLockFirstIfNeeded);
    }
#    endif
    auto realLockType = LockType2FlockType(lockType);
    auto cmd = wait ? realLockType : (realLockType | LOCK_NB);
    if (unLockFirstIfNeeded) {
        // try lock
        auto ret = flock(m_fd, realLockType | LOCK_NB);
        if (ret == 0) {
            return true;
        }
        // let's be gentleman: unlock my shared-lock to prevent deadlock
        ret = flock(m_fd, LOCK_UN);
        if (ret != 0) {
            MMKVError("fail to try unlock first fd=%d, ret=%d, error:%s", m_fd, ret, strerror(errno));
        }
    }

    auto ret = flock(m_fd, cmd);
    if (ret != 0) {
        MMKVError("fail to lock fd=%d, ret=%d, error:%s", m_fd, ret, strerror(errno));
        // try recover my shared-lock
        if (unLockFirstIfNeeded) {
            ret = flock(m_fd, LockType2FlockType(SharedLockType));
            if (ret != 0) {
                // let's hope this never happen
                MMKVError("fail to recover shared-lock fd=%d, ret=%d, error:%s", m_fd, ret, strerror(errno));
            }
        }
        return false;
    } else {
        return true;
    }
}
```

可以看到 MMKV 在文件锁 flock 的基础上，增加了一套计数器机制(代码中的 m_sharedLockCount 和 m_exclusiveLockCount)，来确保

1. 锁的重入特性
2. 锁的共享和互斥性转换特性

关于文件锁的细节，可以参考[https://gavv.github.io/articles/file-locks](https://gavv.github.io/articles/file-locks)

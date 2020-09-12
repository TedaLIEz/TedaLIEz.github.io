---
title: ROOM数据库分析
date: 2020-03-28 23:08:43
tags: Android, ROOM
---

以 2.1.0 版本源码简单分析下 ROOM，主要针对一些自己的困惑做一些记录

为了本文行文方便, 本文的分析全部基于以下的简单的 DB 构造

```kotlin
@Database(entities = {Config.class}, version = 0)
public abstract class AppDatabase extends RoomDatabase {}

@Entity(tableName = "config")
class Config @Ignore constructor(@ColumnInfo(name = "name") var name: String?, @ColumnInfo(name = "value") var value: String?) {
    constructor() : this("", "")

    @PrimaryKey(autoGenerate = true)
    var id: Int = 0

}

@Dao
abstract class ConfigDao {

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    abstract fun insert(config: Config)
}
```

## ROOM 的线程安全是如何保证的

### 先验知识: Transaction 和 Rollback Journals

sqlite 支持通过 transaction 模式来提交事务，一个典型的调用模式如下

```java
db.beginTransaction();
try {
    ...
    db.setTransactionSuccessful();
} finally {
    db.endTransaction();
}
```

可能这里就会有 Error 出现，那么就有可能会 skip`db.setTransactionSuccessful()`, 而直接调用`db.endTransaction()`，根据 Android 文档说明

    The changes will be rolled back if any transaction is ended without being marked as clean (by calling setTransactionSuccessful). Otherwise they will be committed.

也就是说 setTransactionSuccessful()未调用则执行 rollback，这里的 rollback 即 sqlite 的 rollback 策略，需要了解下 sqlite 的 rollback 策略

1. 回滚日志档 (Rollback Journal)：
   先将源文件备份至 Rollback Journal 中，再将要变动的内容直接写入 DB。当需要 rollback 时，再将原内容从 Rollback Journal 写回 DB；若要 commit 变更时，则只要将该档案刪除即可。
   而 rollback 的日志模式又可细分为 4 种：DELETE (SQLite 预设)、TRUNCATE (Android 版 SQLite 预设值)、PERSIST、MEMORY

TRUNCATE 这个模式就是 ROOM 在非 WAL 模式下的默认设置，即 Rollback Journal 总是清空，而非删除

这种模式下的文件目录为一个`db`文件+一个`db-journal`文件

2. WAL (Write-Ahead Log)：
   作法与 Rollback Journal 刚好相反。原内容仍保留在原 DB 之中，但新的变动则 append 至 WAL 文件。而当 COMMIT 发生时，仅代表某个 Transaction 已 append 进 WAL 文件了，但并不一定有写入原 DB (当 WAL 文件大小到达 checkpoint 的阈值时才会写入)。如此可让其他 DB 连接继续对原 DB 内容进行读取操作，而其他连接也可同时将变动 COMMIT 进 WAL 文件。

这种模式下的文件目录为一个`db`文件+一个`db-shm`文件(all SQLite database connections associated with the same database file need to share some memory that is used as an index for the WAL file)+一个`db-wal`文件

ROOM 对 API16 以上机型默认开启 WAL 模式

### getDao 方法都是线程安全的

例如:

```java
  public ConfigDao getConfigDao() {
    if (_configDao != null) {
      return _configDao;
    } else {
      synchronized(this) {
        if(_configDao == null) {
          _configDao = new ConfigDao_Impl(this);
        }
        return _configDao;
      }
    }
  }
```

2. SQLiteOpenHelper#getWritableDatabase 也是线程安全的

```java
public SQLiteDatabase getReadableDatabase() {
    synchronized (this) {
        return getDatabaseLocked(false);
    }
}

@SuppressWarnings("unused")
private SQLiteDatabase getDatabaseLocked(boolean writable) {
    if (mDatabase != null) {
        if (!mDatabase.isOpen()) {
            // Darn!  The user closed the database by calling mDatabase.close().
            mDatabase = null;
        } else if (!writable || !mDatabase.isReadOnly()) {
            // The database is already open for business.
            return mDatabase;
        }
    }

    if (mIsInitializing) {
        throw new IllegalStateException("getDatabase called recursively");
    }

    SQLiteDatabase db = mDatabase;
    try {
        mIsInitializing = true;

        if (db != null) {
            if (writable && db.isReadOnly()) {
                db.reopenReadWrite();
            }
        } else if (mName == null) {
            db = SQLiteDatabase.create(null);
        } else {
            int connectionPoolSize = mForcedSingleConnection ? 1 : 0;
            try {
                if (DEBUG_STRICT_READONLY && !writable) {
                    final String path = mContext.getDatabasePath(mName).getPath();
                    db = SQLiteDatabase.openDatabase(path, mPassword, mCipher, mFactory,
                            SQLiteDatabase.OPEN_READONLY, mErrorHandler, connectionPoolSize);
                } else {
                    mNeedMode = true;
                    mMode = mEnableWriteAheadLogging ? Context.MODE_ENABLE_WRITE_AHEAD_LOGGING : 0;
                    db = Context.openOrCreateDatabase(mContext, mName, mPassword, mCipher,
                            mMode, mFactory, mErrorHandler, connectionPoolSize);
                }
            } catch (SQLiteException ex) {
                if (writable) {
                    throw ex;
                }
                Log.e(TAG, "Couldn't open " + mName
                        + " for writing (will try read-only):", ex);
                final String path = mContext.getDatabasePath(mName).getPath();
                db = SQLiteDatabase.openDatabase(path, mPassword, mCipher, mFactory,
                        SQLiteDatabase.OPEN_READONLY, mErrorHandler);
            }
        }

        return getDatabaseLockedLast(db);

    } finally {
        mIsInitializing = false;
        if (db != null && db != mDatabase) {
            db.close();
        }
    }
}

private SQLiteDatabase getDatabaseLockedLast(SQLiteDatabase db) {
    onConfigure(db);

    final int version = db.getVersion();
    if (version != mNewVersion) {
        if (db.isReadOnly()) {
            throw new SQLiteException("Can't upgrade read-only database from version " +
                    db.getVersion() + " to " + mNewVersion + ": " + mName);
        }

        db.beginTransaction();
        try {
            if (version == 0) {
                onCreate(db);
            } else {
                if (version > mNewVersion) {
                    onDowngrade(db, version, mNewVersion);
                } else {
                    onUpgrade(db, version, mNewVersion);
                }
            }
            db.setVersion(mNewVersion);
            db.setTransactionSuccessful();
        } finally {
            db.endTransaction();
        }
    }

    onOpen(db);

    if (db.isReadOnly()) {
        Log.w(TAG, "Opened " + mName + " in read-only mode");
    }

    mDatabase = db;
    return db;
}
```

### ROOM 的 Dao 层是依赖 Transaction 来执行数据库操作的

例如:

```java

  private final RoomDatabase __db;

  @Override
  public void insert(final Config arg0) {
    __db.assertNotSuspendingTransaction();
    __db.beginTransaction();
    try {
      __insertionAdapterOfConfig.insert(arg0); // 实际上就是SupportSQLiteStatement的操作，这里略去实现代码
      __db.setTransactionSuccessful();
    } finally {
      __db.endTransaction();
    }
  }
```

因此，当我们通过 RoomDatabase#getXXXDao()执行数据库操作时，不同线程将会获得同一个 dao 对象，每个 dao 对象持有同一个 RoomDatabase 对象。那么来看看 RoomDatabase#beginTransaction()

```java
@Deprecated
public void beginTransaction() {
    assertNotMainThread();
    SupportSQLiteDatabase database = mOpenHelper.getWritableDatabase();
    mInvalidationTracker.syncTriggers(database);
    database.beginTransaction();
}
```

1. beginTransaction 委托了给其 SQLiteOpenHelper#getWritableDatabase()

参考上文中 SQLiteOpenHelper#getWritableDatabase()部分代码, getWritableDatabase()也是线程安全

2. 在 SupportSQLiteDatabase#beginTransaction()前，进行了 InvalidationTracer#syncTriggers()调用

```java
void syncTriggers(SupportSQLiteDatabase database) {
    if (database.inTransaction()) {
        // we won't run this inside another transaction.
        return;
    }
    try {
        // This method runs in a while loop because while changes are synced to db, another
        // runnable may be skipped. If we cause it to skip, we need to do its work.
        while (true) {
            Lock closeLock = mDatabase.getCloseLock();
            closeLock.lock();
            try {
                // there is a potential race condition where another mSyncTriggers runnable
                // can start running right after we get the tables list to sync.
                final int[] tablesToSync = mObservedTableTracker.getTablesToSync();
                if (tablesToSync == null) {
                    return;
                }
                final int limit = tablesToSync.length;
                database.beginTransaction();
                try {
                    for (int tableId = 0; tableId < limit; tableId++) {
                        switch (tablesToSync[tableId]) {
                            case ObservedTableTracker.ADD:
                                startTrackingTable(database, tableId);
                                break;
                            case ObservedTableTracker.REMOVE:
                                stopTrackingTable(database, tableId);
                                break;
                        }
                    }
                    database.setTransactionSuccessful();
                } finally {
                    database.endTransaction();
                }
                mObservedTableTracker.onSyncCompleted();
            } finally {
                closeLock.unlock();
            }
        }
    } catch (IllegalStateException | SQLiteException exception) {
        // may happen if db is closed. just log.
        Log.e(Room.LOG_TAG, "Cannot run invalidation tracker. Is the db closed?",
                exception);
    }
}
```

这个 InvalidationTracer 涉及的模块比较大，主要是 ROOM 通过 sqlite 的 trigger 实现对数据变化的监听，下一个部分仔细展开.

## InvalidationTracer 模块的分析

### 先验知识: Temporary Databases

sqlite 在创建表的时候可以用 temp 对 table 进行修饰为 Temporary Database，根据文档的描述[https://www.sqlite.org/inmemorydb.html](https://www.sqlite.org/inmemorydb.html)

    A different temporary file is created each time, so that just like as with the special ":memory:" string, two database connections to temporary databases each have their own private database. Temporary databases are automatically deleted when the connection that created them closes.

简单来说，Temporary 是针对每个数据库链接，在其链接的生命周期下存在的数据表，链接关闭时，这个数据库也就不存在了。

### 先验知识: sqlite trigger

#### 基本语法

- CREATE TRIGGER — command that says that we want to create trigger.
  trigger-name — is a name of the trigger
- BEFORE/AFTER/INSTEAD OF — is a mode of the trigger (when we’d like our operation to work — before our actual query, after or instead)
- DELETE/INSERT/UPDATE ON table-name — is description on the query which will activate our trigger
- BEGIN stmt; END — is actual trigger operation

```sqlite
CREATE TRIGGER update_value INSTEAD OF UPDATE ON persons
BEGIN
UPDATE persons(age) values(21)
END;
```

上面这个例子中，如果 persons 表被更新，那么 update_value 这个 trigger 就会触发, 将 persons 插入的 age 改为 21

#### Android 中如何使用 Trigger

我们知道 SQLiteDatabase 是可以编写 sql 语句的，因此我们可以尝试在 onCreate 阶执行我们的 trigger 构建语句即可

```kotlin
db.execSQL(
    """
    create trigger order_added after insert on data
    begin
    insert into log(timestamp, payload) values(datetime(), new.order_id || ' ' || new.timestamp || ' ' || new.price);
    end;
    """.trimIndent()
)
```

### InvalidationTracer, room_table_modification_log 表

首先我们看下 InvalidationTracer 的构造调用

AppDatabase_Impl.java

```java
  @Override
  protected InvalidationTracker createInvalidationTracker() {
    final HashMap<String, String> _shadowTablesMap = new HashMap<String, String>(0);
    HashMap<String, Set<String>> _viewTables = new HashMap<String, Set<String>>(0);
    return new InvalidationTracker(this, _shadowTablesMap, _viewTables, "config");
  }
```

config 字段就是我们在 AppDatabase 中注册的 Entity 对应的 tableName

InvalidationTracker.java

```java
@SuppressWarnings("WeakerAccess")
@RestrictTo(RestrictTo.Scope.LIBRARY_GROUP_PREFIX)
public InvalidationTracker(RoomDatabase database, Map<String, String> shadowTablesMap,
        Map<String, Set<String>> viewTables, String... tableNames) {
    mDatabase = database;
    mObservedTableTracker = new ObservedTableTracker(tableNames.length);
    mTableIdLookup = new ArrayMap<>();
    mViewTables = viewTables;
    mInvalidationLiveDataContainer = new InvalidationLiveDataContainer(mDatabase);
    final int size = tableNames.length;
    mTableNames = new String[size];
    for (int id = 0; id < size; id++) {
        final String tableName = tableNames[id].toLowerCase(Locale.US);
        mTableIdLookup.put(tableName, id);
        String shadowTableName = shadowTablesMap.get(tableNames[id]);
        if (shadowTableName != null) {
            mTableNames[id] = shadowTableName.toLowerCase(Locale.US);
        } else {
            mTableNames[id] = tableName;
        }
    }
    // Adjust table id lookup for those tables whose shadow table is another already mapped
    // table (e.g. external content fts tables).
    for (Map.Entry<String, String> shadowTableEntry : shadowTablesMap.entrySet()) {
        String shadowTableName = shadowTableEntry.getValue().toLowerCase(Locale.US);
        if (mTableIdLookup.containsKey(shadowTableName)) {
            String tableName = shadowTableEntry.getKey().toLowerCase(Locale.US);
            mTableIdLookup.put(tableName, mTableIdLookup.get(shadowTableName));
        }
    }
}
```

这里有几个数据结构需要注意

1. mTableIdLookup 记录了 tableName 和 index 的关系。例如这里"config"->0
2. mTableNames 数组依次记录了 tableName。例如这里, mTableNames.length = 1, mTableNames[0] = "config"
3. mDatabase 即 AppDatabase_Impl 对象
4. mObservedTableTracker 是一个 ObservedTableTracker 对象，这个对象内部维护了三个数组:
   1. mTableObservers, 一个 long 数组
   2. mTriggerStates, 一个 boolean 数组
   3. mTriggerStateChanges, 一个 int 数组

再来关注下`InvalidationTracker#internalInit`的调用链，查阅源码可以得到

`SupportSQLiteOpenHelper.Callback#onOpen`->`AppDatabase_Impl#internalInitInvalidationTracker`->
`InvalidationTracker#internalInit`

```java
void internalInit(SupportSQLiteDatabase database) {
    synchronized (this) {
        if (mInitialized) {
            Log.e(Room.LOG_TAG, "Invalidation tracker is initialized twice :/.");
            return;
        }

        // These actions are not in a transaction because temp_store is not allowed to be
        // performed on a transaction, and recursive_triggers is not affected by transactions.
        database.execSQL("PRAGMA temp_store = MEMORY;");
        database.execSQL("PRAGMA recursive_triggers='ON';");
        database.execSQL(CREATE_TRACKING_TABLE_SQL);
        syncTriggers(database);
        mCleanupStatement = database.compileStatement(RESET_UPDATED_TABLES_SQL);
        mInitialized = true;
    }
}
```

我们把 sql 语句串联一下

```sql
PRAGMA temp_store = MEMORY;
PRAGMA recursive_triggers='ON';
CREATE TEMP TABLE room_table_modification_log (table_id INTEGER PRIMARY KEY, invalidated INTEGER NOT NULL DEFAULT 0);
UPDATE room_table_modification_log SET invalidated 0 WHERE invalidated = 1;
```

1. 创建了一个 In-memory 的临时表 room_table_modification_log
2. 主键 table_id, 我们这里可以猜测这个 table_id 应该就和构造函数里提到的 id 概念对应
3. invalidated 字段，字面意思应该是表示数据是否有效

在 CREATE 语句和 UPDATE 语句之间，还有一个 syncTriggers 调用

```java
void syncTriggers(SupportSQLiteDatabase database) {
    if (database.inTransaction()) {
        // we won't run this inside another transaction.
        return;
    }
    try {
        // This method runs in a while loop because while changes are synced to db, another
        // runnable may be skipped. If we cause it to skip, we need to do its work.
        while (true) {
            Lock closeLock = mDatabase.getCloseLock();
            closeLock.lock();
            try {
                // there is a potential race condition where another mSyncTriggers runnable
                // can start running right after we get the tables list to sync.
                final int[] tablesToSync = mObservedTableTracker.getTablesToSync();  // #1
                if (tablesToSync == null) {
                    return;
                }
                final int limit = tablesToSync.length;
                database.beginTransaction();
                try {
                    for (int tableId = 0; tableId < limit; tableId++) {
                        switch (tablesToSync[tableId]) {
                            case ObservedTableTracker.ADD:
                                startTrackingTable(database, tableId);
                                break;
                            case ObservedTableTracker.REMOVE:
                                stopTrackingTable(database, tableId);
                                break;
                        }
                    }
                    database.setTransactionSuccessful();
                } finally {
                    database.endTransaction();
                }
                mObservedTableTracker.onSyncCompleted();
            } finally {
                closeLock.unlock();
            }
        }
    } catch (IllegalStateException | SQLiteException exception) {
        // may happen if db is closed. just log.
        Log.e(Room.LOG_TAG, "Cannot run invalidation tracker. Is the db closed?",
                exception);
    }
}

```

接着看下#1 这里， `ObservedTableTracker#getTablesToSync()`

```java
@Nullable
int[] getTablesToSync() {
    synchronized (this) {
        if (!mNeedsSync || mPendingSync) {
            return null;
        }
        final int tableCount = mTableObservers.length;
        for (int i = 0; i < tableCount; i++) {
            final boolean newState = mTableObservers[i] > 0;
            if (newState != mTriggerStates[i]) {
                mTriggerStateChanges[i] = newState ? ADD : REMOVE;
            } else {
                mTriggerStateChanges[i] = NO_OP;
            }
            mTriggerStates[i] = newState;
        }
        mPendingSync = true;
        mNeedsSync = false;
        return mTriggerStateChanges;
    }
}
```

如果不需要同步(!mNeedSync)或者正在同步(mPendingSync)，返回 null

否则

我们对每个 table_id 做判断，在当前场景下, mTableObservers[0] = 0, mTriggerStates[0] = false, mTriggerStateChanges[0] = 0, 经过算法后
mTableObservers[0] = 0, mTriggerStates[0] = false, mTriggerStateChanges[0] = NO_OP

回到 syncTriggers 这里，返回了一个 NO_OP，那么什么事情都不做。直接返回

那么时候才会走到 startTrackingTable 或者 stopTrackingTable 呢？针对这个问题，我们看下 ObservedTableTracker.ADD 的赋值位置:

```java
final boolean newState = mTableObservers[i] > 0;
if (newState != mTriggerStates[i]) {
    mTriggerStateChanges[i] = newState ? ADD : REMOVE;
} else {
    mTriggerStateChanges[i] = NO_OP;
}
```

可以看到 newState = mTableObservers[i] > 0 成立时才会是 ADD, 那么我们看看什么时候 mTableObservers[i] > 0，可以发现是`ObservedTableTracker#onAdded调用`

onAdded 调用链:

`InvalidationTracker#addObserver`->`InvalidationTracker#onAdded()`

`InvalidationTracker#addWeakObserver`->`InvalidationTracker#addObserver`->`InvalidationTracker#onAdded()`

addWeakObserver 则用在了`RoomTrackingLiveData#mRefreshRunnable`或`LimitOffsetDataSource()`

这里我们可以大致做一个推断

1. paging 或 ROOM 中对 LiveData 的支持，依赖了 addObserver 或者 addWeakObserver 来实现，这是必然的。因为二者是一个典型的观察者模式范型的表现

接下来我们看看 addObserver 的实现

```java
@SuppressLint("RestrictedApi")
@WorkerThread
public void addObserver(@NonNull Observer observer) {
    final String[] tableNames = resolveViews(observer.mTables);
    int[] tableIds = new int[tableNames.length];
    final int size = tableNames.length;

    for (int i = 0; i < size; i++) {
        Integer tableId = mTableIdLookup.get(tableNames[i].toLowerCase(Locale.US));
        if (tableId == null) {
            throw new IllegalArgumentException("There is no table with name " + tableNames[i]);
        }
        tableIds[i] = tableId;
    }
    ObserverWrapper wrapper = new ObserverWrapper(observer, tableIds, tableNames);
    ObserverWrapper currentObserver;
    synchronized (mObserverMap) {
        currentObserver = mObserverMap.putIfAbsent(observer, wrapper);
    }
    if (currentObserver == null && mObservedTableTracker.onAdded(tableIds)) {
        syncTriggers();
    }
}
```

observer.mTables 就是 observer 所观察的 tableName 数组, 是 String[], 我们这里可以假设是["config"], resolveViews 主要解决了数据库的视图映射，这里我们不涉及。可以直接理解 tableNames=["config"], 后续算法我们可以得到 tablesIds.length = 1, tablesIds[0] = 0. 之后做了两件事情

1. 将一个 ObserverWrapper 对象放到 mObserverMap 中，map 维护了一个 Observer 和 ObserverWrapper 关系
2. 如果是第一次加入 observerMap 的 observer 对象，调用 onAdded()
3. 如果 2 的基础上，onAdded()返回 true，调用 syncTriggers()

```java
boolean onAdded(int... tableIds) {
    boolean needTriggerSync = false;
    synchronized (this) {
        for (int tableId : tableIds) {
            final long prevObserverCount = mTableObservers[tableId];
            mTableObservers[tableId] = prevObserverCount + 1;
            if (prevObserverCount == 0) {
                mNeedsSync = true;
                needTriggerSync = true;
            }
        }
    }
    return needTriggerSync;
}
```

这里对每个 tableId 做了判断, 如果其之前从没有过 observer 对该 tableId 进行监听, 那么就需要进行一次 sync(needTriggerSync = true)

回忆一下之前对 getTablesToSync 的分析，此时 mTableObservers[tableId]>0, 那么在 syncTriggers()逻辑中就会得到一个 ADD 指令, 我们看看 ADD 对应的`startTrackingTable`函数

```java
private void startTrackingTable(SupportSQLiteDatabase writableDb, int tableId) {
    writableDb.execSQL(
            "INSERT OR IGNORE INTO " + UPDATE_TABLE_NAME + " VALUES(" + tableId + ", 0)");
    final String tableName = mTableNames[tableId];
    StringBuilder stringBuilder = new StringBuilder();
    for (String trigger : TRIGGERS) {
        stringBuilder.setLength(0);
        stringBuilder.append("CREATE TEMP TRIGGER IF NOT EXISTS ");
        appendTriggerName(stringBuilder, tableName, trigger);
        stringBuilder.append(" AFTER ")
                .append(trigger)
                .append(" ON `")
                .append(tableName)
                .append("` BEGIN UPDATE ")
                .append(UPDATE_TABLE_NAME)
                .append(" SET ").append(INVALIDATED_COLUMN_NAME).append(" = 1")
                .append(" WHERE ").append(TABLE_ID_COLUMN_NAME).append(" = ").append(tableId)
                .append(" AND ").append(INVALIDATED_COLUMN_NAME).append(" = 0")
                .append("; END");
        writableDb.execSQL(stringBuilder.toString());
    }
}
```

我们尝试把所有 sql 展开得到

```sql
INSERT OR IGNORE INTO room_table_modification_log VALUES($tableId, 0)

CREATE TEMP TRIGGER IF NOT EXISTS `room_table_modification_trigger_$tableName_UPDATE` AFTER UPDATE ON `$tableName` BEGIN UPDATE room_table_modification_log SET invalidated = 1 WHERE tableId = $tableId AND invalidated = 0; END

CREATE TEMP TRIGGER IF NOT EXISTS `room_table_modification_trigger_$tableName_REMOVE` AFTER REMOVE ON `$tableName` BEGIN UPDATE room_table_modification_log SET invalidated = 1 WHERE tableId = $tableId AND invalidated = 0; END

CREATE TEMP TRIGGER IF NOT EXISTS `room_table_modification_trigger_$tableName_INSERT` AFTER INSERT ON `$tableName` BEGIN UPDATE room_table_modification_log SET invalidated = 1 WHERE tableId = $tableId AND invalidated = 0; END
```

这一段就是在 DB，建立了对不同的表的有效值 trigger 机制, 通过这个 trigger 的设置，我们能进一步理解 room_table_modification_log 的作用

| 表名        | 作用                                              |
| ----------- | ------------------------------------------------- |
| tableId     | 唯一值，与上层 Entity 中的 tableName 构成一一对应 |
| invalidated | 0 或 1，1 表示数据有更新                          |

那么综合一下上述分析:

1. 当且仅当一个表首次被监听时, 我们会创建一组 trigger，对这个表的数据更新状态通过 invalidated 进行表示
2. 在 Java 层，我们使用一个 Map 保存了 Observer 的映射关系，以便后续观察到数据变化时唤起

看完了监听的建立，我们就要来看看如何唤起这个监听了

### InvalidationTracker 观察者模式的触发

首先需要明确的是，我们肯定是要获得 DB 中哪些表的数据进行了更新，根据上述分析。我们可以从 room_table_modification_log 中 invalidated=1 的项进行分析。那么遵循这个思路，我们查阅代码中对 room_table_modification_log 的查询操作

InvalidationTracker#mRefreshRunnable

```java
Runnable mRefreshRunnable = new Runnable() {
    @Override
    public void run() {
        final Lock closeLock = mDatabase.getCloseLock();
        Set<Integer> invalidatedTableIds = null;
        try {
            closeLock.lock();

            if (!ensureInitialization()) {
                return;
            }

            if (!mPendingRefresh.compareAndSet(true, false)) {
                // no pending refresh
                // 防止重入
                return;
            }

            if (mDatabase.inTransaction()) {
                // current thread is in a transaction. when it ends, it will invoke
                // refreshRunnable again. mPendingRefresh is left as false on purpose
                // so that the last transaction can flip it on again.
                return;
            }

            if (mDatabase.mWriteAheadLoggingEnabled) { // #1
                // This transaction has to be on the underlying DB rather than the RoomDatabase
                // in order to avoid a recursive loop after endTransaction.
                SupportSQLiteDatabase db = mDatabase.getOpenHelper().getWritableDatabase();
                db.beginTransaction();
                try {
                    invalidatedTableIds = checkUpdatedTable();
                    db.setTransactionSuccessful();
                } finally {
                    db.endTransaction();
                }
            } else {
                invalidatedTableIds = checkUpdatedTable();
            }
        } catch (IllegalStateException | SQLiteException exception) {
            // may happen if db is closed. just log.
            Log.e(Room.LOG_TAG, "Cannot run invalidation tracker. Is the db closed?",
                    exception);
        } finally {
            closeLock.unlock();
        }
        if (invalidatedTableIds != null && !invalidatedTableIds.isEmpty()) {
            synchronized (mObserverMap) {
                for (Map.Entry<Observer, ObserverWrapper> entry : mObserverMap) {
                    entry.getValue().notifyByTableInvalidStatus(invalidatedTableIds); // #2
                }
            }
        }
    }

    private Set<Integer> checkUpdatedTable() {
        ArraySet<Integer> invalidatedTableIds = new ArraySet<>();
        Cursor cursor = mDatabase.query(new SimpleSQLiteQuery(SELECT_UPDATED_TABLES_SQL));  // #3
        //noinspection TryFinallyCanBeTryWithResources

        // 通过cursor读到 invalidatedTableIds，略去
        if (!invalidatedTableIds.isEmpty()) {
            mCleanupStatement.executeUpdateDelete(); // 有数据更新，重置下invalidated标志位
        }
        return invalidatedTableIds;
    }
};
```

通过 checkUpdatedTable 函数，我们得到了所有 invalidated=1 的 tableId，之后在 mObserverMap 中通知观察者 notifyByTableInvalidStatus

```java
void notifyByTableInvalidStatus(Set<Integer> invalidatedTablesIds) {
    Set<String> invalidatedTables = null;
    final int size = mTableIds.length;
    for (int index = 0; index < size; index++) {
        final int tableId = mTableIds[index];
        if (invalidatedTablesIds.contains(tableId)) {
            if (size == 1) {
                // Optimization for a single-table observer
                invalidatedTables = mSingleTableSet;
            } else {
                if (invalidatedTables == null) {
                    invalidatedTables = new ArraySet<>(size);
                }
                invalidatedTables.add(mTableNames[index]);
            }
        }
    }
    if (invalidatedTables != null) {
        mObserver.onInvalidated(invalidatedTables);
    }
}
```

invalidatedTables 记录了 tableId 对应的原 Entity 对应的 table_name, onInvalidated 则是接口，由 Observer 自己实现，这里我们以 RoomTrackingLiveData 分析一下，找到其内部聚合的`onInvalidated`实现

```java

@SuppressLint({"RestrictedApi"})
RoomTrackingLiveData(RoomDatabase database, InvalidationLiveDataContainer container, boolean inTransaction, Callable<T> computeFunction, String[] tableNames) {
    // ...
    this.mObserver = new Observer(tableNames) {
        public void onInvalidated(@NonNull Set<String> tables) {
            ArchTaskExecutor.getInstance().executeOnMainThread(RoomTrackingLiveData.this.mInvalidationRunnable); // #1
        }
    };
}

final Runnable mInvalidationRunnable = new Runnable() {
    @MainThread
    public void run() {
        boolean isActive = RoomTrackingLiveData.this.hasActiveObservers();
        if (RoomTrackingLiveData.this.mInvalid.compareAndSet(false, true) && isActive) {
            RoomTrackingLiveData.this.getQueryExecutor().execute(RoomTrackingLiveData.this.mRefreshRunnable); // #2
        }

    }
};

final Runnable mRefreshRunnable = new Runnable() {
@WorkerThread
public void run() {
    if (RoomTrackingLiveData.this.mRegisteredObserver.compareAndSet(false, true)) {
        RoomTrackingLiveData.this.mDatabase.getInvalidationTracker().addWeakObserver(RoomTrackingLiveData.this.mObserver);
    }

    boolean computed;
    do {
        computed = false;
        if (RoomTrackingLiveData.this.mComputing.compareAndSet(false, true)) {
            try {
                Object value = null;

                while(RoomTrackingLiveData.this.mInvalid.compareAndSet(true, false)) {
                    computed = true;

                    try {
                        value = RoomTrackingLiveData.this.mComputeFunction.call();
                    } catch (Exception var7) {
                        throw new RuntimeException("Exception while computing database live data.", var7);
                    }
                }

                if (computed) {
                    RoomTrackingLiveData.this.postValue(value); // #3
                }
            } finally {
                RoomTrackingLiveData.this.mComputing.set(false);
            }
        }
    } while(computed && RoomTrackingLiveData.this.mInvalid.get());

}
};
```

篇幅问题这里不分析 RoomTrackingLiveData 的仔细实现，从关键的#1, #2, #3 看，RoomTrackingLiveData 利用这个 Observer，当`Observer#onInvalidated`被触发的时候, 对 DB 进行了`mComputeFunction#call()`操作，这个我们可以直接猜测就是上层的 DB 查询具体实现，这个由具体的 DAO 层解释。然后调用 postValue 来刷新 LiveData 的值

分析完整个监听被触发的过程，有一个问题没有解释，就是`InvalidationTracker#mRefreshRunnable`是如何被调用的

追溯源码

```java
@SuppressWarnings("WeakerAccess")
public void refreshVersionsAsync() {
    // TODO we should consider doing this sync instead of async.
    if (mPendingRefresh.compareAndSet(false, true)) {
        mDatabase.getQueryExecutor().execute(mRefreshRunnable);
    }
}

@RestrictTo(RestrictTo.Scope.LIBRARY_GROUP_PREFIX)
@WorkerThread
public void refreshVersionsSync() {
    syncTriggers();
    mRefreshRunnable.run();
}
```

追溯调用链

1. `RoomDatabase#endTransaction()`->`InvalidationTracker#refreshVersionsAsync()`

2. `LimitOffsetDataSource#isInvalid()`->`InvalidationTracker#refreshVersionsSync()`

篇幅问题，不分析跟 Paging Library 关联大的 2。分析一下 1

实际上，在[ROOM 的线程安全是如何保证的]这节的第三部分，我们已经能发现 Dao 层的 DB 操作都是依赖 beginTransaction, endTransaction 来完成 DB 事务的。也就是说，只要 DAO 层有 DB 事务发生，那么 ROOM 必定会在 getQueryExecutor()的线程池中，执行 mRefreshRunnable, 如果发现了有数据更新的 table，就将这些 table 信息全部抛给`Observer#onInvalidated`处理

## InvalidationTracker 小结

`InvalidationTracker`的职责即建立了业务对 DB 数据写的观察者模式, 业务的 Observer 被`InvalidationTracker`聚合持有，同时建立一个临时表`room_table_modification_log`

| 表名        | 作用                                              |
| ----------- | ------------------------------------------------- |
| tableId     | 唯一值，与上层 Entity 中的 tableName 构成一一对应 |
| invalidated | 0 或 1，1 表示数据有更新                          |

当某个表被业务监听时，对这个表创建 trigger 建立观察者模式，监听这张表的写操作，trigger 被触发时就写`room_table_modification_log`

DAO 层业务代码委托`RoomDatabase`通过`beginTransaction`, `endTransaction`来完成 DB 的事务提交，这些函数被调用时，顺带触发了`InvalidationTracker`内部对表`room_table_modification_log`的异步查询，查询到有数据更新`(invalidated=1）`的表名信息, 将这些表名信息带给`Observer#onInvalidated`的参数中。例如 RoomTrackingLiveData 中, 利用`InvalidationTracker`建立的这套机制，实现了 DAO 层 LiveData 的持有数据实时更新的特性

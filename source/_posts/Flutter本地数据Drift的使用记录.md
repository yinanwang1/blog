---
title: Flutter本地数据Drift的使用记录
date: 2026-10-12 23:31:18
tags:
---

需要持久化的数据量小，结构简单，直接采用系统自带的就够。如果数据量大，结构复杂，那么还是需要数据库来保存。当然文件保存也是可行，不过读取数据的灵活性还是数据库好。使用系统的持久化时，需要注意数据类型，是否存在，设置好默认值。使用数据库保存，那么表结构需要设计好，数据库迁移要考虑到跨版本升级。

项目的新功能都是用`flutter`编写的，尝试下`Drfit`的数据创建，读写，升级。代码可以查看[drift_demo](https://github.com/yinanwang1/FlutterSnippet/tree/main/lib/drift_demo)。



## 一、`Drift`的了解

Drift是基于 SQLite的离线数据读写，并让数据库操作更方便、更易维护。支持Android、iOS、Linux、MacOS、Web和Windows平台。它有三个特点：

- **类型安全**：根据表结构生成 Dart 数据对象和查询接口，减少手动转换及字段使用错误。
- **响应式查询**：通过 `watch()` 监听查询结果，数据变化后自动通知页面更新。
- **数据库迁移**：通过版本号和迁移逻辑调整表结构，在升级时保留已有数据。（减少了开发人员手动处理数据）



## 二、集成`Drift`

#### 1. 添加依赖

在项目根目录执行：

```
flutter pub add drift drift_flutter path_provider
flutter pub add --dev drift_dev build_runner
```

`drift` 提供数据库 API，`drift_flutter` 负责打开数据库，`path_provider` 获取存储目录；`drift_dev` 和 `build_runner` 用于生成代码。

#### 2. 创建数据库

新建 `app_database.dart`：

```dart
import 'package:drift/drift.dart';
import 'package:drift_flutter/drift_flutter.dart';
import 'package:path_provider/path_provider.dart';

part 'app_database.g.dart';

@DriftDatabase(tables: [])
class AppDatabase extends _$AppDatabase {
  AppDatabase()
      : super(
          driftDatabase(
            name: 'app_database',
            native: const DriftNativeOptions(
              databaseDirectory: getApplicationSupportDirectory,
            ),
          ),
        );

  @override
  int get schemaVersion => 1;
}
```

- `tables`：注册表，目前为空，下一步添加。(比如添加`User`表)
- `name` 和 `databaseDirectory`：指定数据库文件名及存储目录，升级时保持不变。
- `schemaVersion`：数据库结构版本，后续修改表结构时配合迁移逻辑递增。（数据库的不同版本，每次数据结构变了，这个版本号就+1，只能向上升）

#### 3. 生成代码

创建好`AppDatabase`即定义数据库后，让工具生成配套代码。以后新增表或修改表，都需要执行：

```
dart run build_runner build
```

生成的 `app_database.g.dart` 包含 `_$AppDatabase` 基类，以及后续表定义对应的数据对象和操作辅助代码。**不要手动修改它**；修改数据库声明后重新生成即可。

数据库**按需打开**，首次执行查询或写入时才真正访问数据库文件。



## 三、创建数据表

#### 1. 定义心率记录表

```dart
import 'package:drift/drift.dart';

@DataClassName('HeartRateRecord')
class HeartRateRecords extends Table {
  IntColumn get id => integer().autoIncrement()();
  TextColumn get deviceId => text()();
  IntColumn get bpm => integer()();
  DateTimeColumn get measuredAt => dateTime()();
  DateTimeColumn get createdAt =>
      dateTime().withDefault(currentDateAndTime)();
}
```

在文件夹`tables`中新建数据库的表，以后新增的表都放在这个文件夹中。`id` 为自增主键，`deviceId` 标识设备，`bpm` 保存心率，`measuredAt` 保存测量时间，`createdAt` 默认使用数据库当前时间。

#### 2. 注册表并生成代码

在 `AppDatabase` 中注册：

```
@DriftDatabase(tables: [HeartRateRecords])
```

保持 `schemaVersion` 为 `1`，然后执行：

```
dart run build_runner build
```

更新`app_database.g.dart`中内容，。四个类型的职责不同：

| 类型                        | 作用                                                         |
| --------------------------- | ------------------------------------------------------------ |
| `HeartRateRecords`          | 手写的表结构定义                                             |
| `$HeartRateRecordsTable`    | 自动生成，根据表定义生成实际表实现，让`HeartRateRecords`能数据库操作 |
| `HeartRateRecord`           | 自动生成，表示查询得到的一条完整记录                         |
| `HeartRateRecordsCompanion` | 自动生成，用于插入或更新，可区分“提供字段值”和“不修改该字段” |

#### 3. 首次打开时建表

在 `AppDatabase` 中配置：

```dart
@override
MigrationStrategy get migration => MigrationStrategy(
  onCreate: (migrator) => migrator.createAll(),
);
```

首次实际访问新数据库时，`Drift` 自动创建已注册的表；再次打开已有数据库，不会重复建表。

#### 4. drift_schema_v1.json 各版本表结构快照

`drift_schema_v1.json`是`drift_dev`生成的：

```dart
dart run drift_dev schema dump \
  lib/drift_demo/database/app_database.dart \
  drift_schemas/
```

工具读取数据库定义，根据 `schemaVersion` 导出对应的表结构快照，所以 V1 得到 `drift_schema_v1.json`。已发布版本的快照应保留不变，新版本另存一份。

**迁移测试**：按快照创建一个 V1 数据库，再运行新版本迁移，验证旧数据能否保留、表结构是否正确。

**历史对照**：查看旧版本有哪些字段、类型、默认值和约束，方便编写或检查迁移逻辑。

## 四、 读写数据并展示

#### 1. 插入、查询、修改、删除

在AppDatabase新增的方法：

1. `MigrationStrategy get migration => MigrationStrategy` 配置首次建表和版本升级逻辑，由 Drift 打开数据库时调用。
2. `Future<int> addHeartRateRecord` 插入一条心率记录。
3. `Future<int> updateHeartRate(int id, int bpm)`修改指定 ID 的心率值。
4. `Future<int> deleteHeartRateRecord(int id)`删除指定 ID 的记录。
5. `Future<String> databaseFilePath()`查询当前数据库文件的实际路径，方便调试。

其中 `Future` 表示异步操作，需要用 `await` 获取结果；`Stream` 会持续发送结果，需要订阅或通过 `StreamBuilder` 展示。如果需要其他的数据库操作，继续在`AppDatabase`新增。

#### 2. 监听数据并展示

`get()` 查询一次，`watch()` 持续监听查询结果：

```dart
Stream<List<HeartRateRecord>> watchHeartRateRecords() {
    return (select(heartRateRecords)
          ..orderBy([
            (table) => OrderingTerm.desc(table.measuredAt),
            (table) => OrderingTerm.desc(table.id),
          ]))
        .watch();
  }
```

查询心率记录并持续监听变化，按测量时间倒序返回，适合自动刷新列表。

页面初始化时保存 Stream：

```
// database 使用应用内共享的 AppDatabase 实例
_records = _database.watchHeartRateRecords();
```

通过 `StreamBuilder` 展示：

```dart
Expanded(
  child: StreamBuilder<List<HeartRateRecord>>(
    stream: _records,
    builder: (context, snapshot) {
      if (snapshot.hasError) {
        return Center(child: Text('读取失败：${snapshot.error}'));
      }
      if (!snapshot.hasData) {
        return const Center(child: CircularProgressIndicator());
      }
      final records = snapshot.data!;
      return Column(
        children: [
          Text('共 ${records.length} 条记录（按测量时间倒序）'),
          Expanded(
            child: records.isEmpty
                ? const Center(child: Text('暂无记录，点击按钮添加'))
                : ListView.builder(
                    itemCount: records.length,
                    itemBuilder: (context, index) {
                      final record = records[index];
                      return ListTile(
                        key: ValueKey(record.id),
                        title: Text(
                            '${record.bpm} bpm · #${record.id}'),
                        subtitle: Text(
                          '设备：${record.deviceId}\n'
                          '测量：${record.measuredAt.toLocal()}\n'
                          '创建：${record.createdAt.toLocal()}',
                        ),
                        isThreeLine: true,
                        trailing: Row(
                          mainAxisSize: MainAxisSize.min,
                          children: [
                            IconButton(
                              tooltip: '修改心率',
                              onPressed: _saving
                                  ? null
                                  : () => _edit(record),
                              icon: const Icon(Icons.edit_outlined),
                            ),
                            IconButton(
                              tooltip: '删除记录',
                              onPressed: _saving
                                  ? null
                                  : () => _runOperation(() async {
                                        await _database
                                            .deleteHeartRateRecord(
                                                record.id);
                                      }),
                              icon:
                                  const Icon(Icons.delete_outline),
                            ),
                          ],
                        ),
                      );
                    },
                  ),
          ),
        ],
      );
    },
  ),
)
```

通过 Drift 增删改数据后，Stream 自动发送最新列表，页面随之更新。

#### 3. 验证持久化

等待写入完成，再彻底关闭并重新打开 App。使用相同数据库文件查询，记录仍应存在。不要卸载或清除应用数据。

## 五、修改表结构和版本升级

修改表结构时，需要同时完成三件事：（1）**修改表定义、（2）提升 `schemaVersion`、（3）编写迁移逻辑**。然后重新生成代码`app_database.g.dart`和`drift_schema_v*.json`。

#### 1. V1 → V2：新增同步状态

在`HeartRateRecords`表中增加：

```dart
BoolColumn get isSynced => boolean().withDefault(const Constant(false))();
```

版本改为 `2`，升级时添加列：

```dart
await migrator.addColumn(
  heartRateRecords,
  heartRateRecords.isSynced,
);
```

旧记录和未指定该字段的新记录，默认值都是 `false`。 执行命令生成`drift_schema_v2.json`。

在`AppDatabase`中方法修改`isSynced`为`true`:

```dart
 Future<int> markHeartRateSynced(int id) {
    return (update(heartRateRecords)..where((table) => table.id.equals(id)))
        .write(const HeartRateRecordsCompanion(isSynced: Value(true)));
  }
```

#### 2. V2 → V3：新增温度

在`HeartRateRecords`表中增加：

```dart
RealColumn get temperature => real().withDefault(const Constant(36.8))();
```

版本改为 `3`，升级时添加列：

```dart
await migrator.addColumn(
  heartRateRecords,
  heartRateRecords.temperature,
);
```

旧记录默认温度为 `36.8`。 执行命令生成`drift_schema_v3.json`。

#### 3. V3 → V4：删除同步状态

删除`HeartRateRecords`表定义中的 `isSynced`，版本改为 `4`。使用 Drift 重建表，复制其余字段的数据：

```dart
await migrator.alterTable(
  TableMigration(heartRateRecords),
);
```

心率、温度等记录保留，原同步状态被删除。重建表时，如需保证历史 ID 不复用，还应保留自增序列。

执行命令生成`drift_schema_v4.json`。

#### 4. 首次建库、升级与跨版本处理

V4 的迁移核心代码：

```dart
@override
int get schemaVersion => 4;

@override
MigrationStrategy get migration => MigrationStrategy(
  onCreate: (migrator) => migrator.createAll(),
  onUpgrade: (migrator, from, to) async {
    await transaction(() async {
      if (from < 3) {
        await migrator.addColumn(
          heartRateRecords,
          heartRateRecords.temperature,
        );
      }

      if (from < 4) {
        await migrator.alterTable(
          TableMigration(heartRateRecords),
        );
      }
    });
  },
);
```

- `onCreate`：新数据库直接创建当前 V4 表结构。
- `onUpgrade`：已有数据库版本较低时执行迁移。
- `transaction`：迁移全部成功才提交，异常时整体回滚。
- `from`：旧版本号。V1/V2 直接升级 V4 时先补温度，再调整为 V4 结构；无需添加已废弃的同步字段。

最后执行：

```bash
dart run build_runner build
```

**代码生成更新 Dart 类型，迁移更新设备上已有的数据库，两者不能互相替代。**

## 六、迁移成功

验证迁移，应先安装旧版本并写入数据，再**保留应用数据覆盖安装新版本**。卸载重装只能验证首次建库，无法验证旧库升级。

升级后重点检查：

- **旧数据**：记录数量、ID 和保留字段的值不变。
- **表结构**：新增字段及默认值正确，删除字段已不存在。
- **读写操作**：查询、插入、修改、删除正常，页面展示正确。
- **重新启动**：数据仍保留，不会重复执行迁移。

还应验证跨版本升级，例如 V1 直接升级 V4，并分别在 Android、iOS 上测试。

生产环境应**谨慎删除**字段：**先停止使用，确认不再需要且兼容旧版本升级后，再通过迁移删除**，避免数据丢失或影响版本回退。



## 结尾

使用工具进行生成代码，减少了好多的工作量。但是还是要每一个函数的作用，阅读大致的流程，做到心里有数。

又是面向AI学习的一天，谁是主人，该怎么判定。!_! 




































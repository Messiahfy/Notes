#### 1.概述
SQLite不是独立进程，和APP运行在同一进程。

SQLiteOpenHelper包含了用于管理数据库的API，当使用此类获取对数据库的引用时，系统仅在创建和更新时才执行可能的长时间操作，而不是整个app启动期间。我们需要的只是调用getWritableDatabase()和getReadableDatabase()。

getWritableDatabase()和getReadableDatabase()可能执行时间较长，应该放在子线程执行。

查询的cursor在查询完要调用close()
不需要使用数据库时，应该调用dbHelper.close();

[Android ：SQLlite数据库 使用手册](https://www.jianshu.com/p/8e3f294e2828)

#### 2.创建数据库
1.创建SQLiteOpenHelper子类
2.重写构造函数、onCreate方法、onUpgrade方法
3.构造SQLiteOpenHelper子类实例，如果传入版本号大于当前版本，则会调用onUpgrade，onCreate只有第一次创建数据库的时候才会调用。

> 第一次调用`getWritableDatabase()`或者`getReadableDatabase()`才真正创建数据库文件。
#### 3.增删改查

###### 3.1 添加数据
`SQLiteDatabase.insert(String table, String nullColumnHack, ContentValues values)`

`table` 表名
`nullColumnHack` 当values参数为空或者里面没有内容的时候，insert是会失败的（底层数据库不允许插入一个每列数据都为空的空行），为了防止这种情况，我们要在这里指定一个 列名，到时候如果发现将要插入的行为空行时，就会将你指定的这个列名（仅能指定一个列）的值设为null，然后再向数据库中插入。
`values` 列名和值的键值对。
返回插入的行ID，如果错误发生则返回-1.

###### 3.2 更新数据
`update(String table, ContentValues values, String whereClause, String[] whereArgs)`

`table` 表名
`values` 新的列名的值的映射
`whereClause ` 更新的WHERE约束，为空则更新所有
`whereArgs` 如果whereClause中使用任意个?，则会按序替换为字符串数组`whereArgs`中的字符串。

```
//两种写法等价
database.update("person",contentValues,"name=\"huang\"",null);
database.update("person",contentValues,"name=?",new String[]{"huang"});
```

###### 3.3 删除数据
`delete(String table, String whereClause, String[] whereArgs)`
参数意义同更新数据

###### 3.4 查询数据
`query(String table, String[] columns, String selection,
            String[] selectionArgs, String groupBy, String having,
            String orderBy)`
`table` 表名
`columns` 查询哪些列
`selection ` WHERE约束
`selectionArgs ` WHERE中使用?时的替换值
`groupBy` 指定需要group by 的列
`having` 对group by后的值进一步约束
`orderBy` 指定排序方式

返回`Cursor`实例
```
if (cursor.moveToFirst()) {
    do {
        String name = cursor.getString(cursor.getColumnIndex("name"));
        String address = cursor.getString(cursor.getColumnIndex("address"));
        Log.d("hhhhhhhhh", "name: " + name + " address: " + address);
    } while (cursor.moveToNext());
}
```

#### 3.5 直接使用sql语句
`execSQL(String sql)`  增删改可以使用此方法
`rawQuery(String sql, String[] selectionArgs)` 查询必须使用此方法

#### 3.6 数据库事务
使用事务的两大好处是原子提交和更优性能。
`SQLite`默认会为每个插入、更新操作创建一个事务，并且在每个插入、更新操作后立即提交。如果连续插入1000次数据，执行过程就重复1000次：创建事务-->执行语句-->提交。而如果使用事务，则为：创建事务-->执行1000次语句-->提交，这样可以大幅提升性能。
```
db.beginTransaction();
try {
       ...
       db.setTransactionSuccessful();
 } finally {
      db.endTransaction();
 }
```
### 依赖引用

```groovy
dependencies {
  	implementation 'androidx.room:room-runtime:2.4.2'
    annotationProcessor 'androidx.room:room-compiler:2.4.2'
}
```

官网上推荐了很多可选的依赖能力库，如果需要考虑包大小、以及简单使用的话，其实是不需要的。

### 使用步骤

#### 新建数据库

##### 基于SQLiteOpenHelper

先来看一下最原始的Android SQLite是怎么创建数据库的

```java
public class CustomHelper extends SQLiteOpenHelper{
 
 public static String CREATE_TABLE = "create table "+ DatabaseStatic.TABLE_NAME +"(" + 
   DatabaseStatic.BOOK_NAME + " varchar(30), " + 
   DatabaseStatic.ID + " Integer primary key autoincrement, " + 
   DatabaseStatic.AUTHOR + " varchar(20) not null, " + 
   DatabaseStatic.PRICE + " real)"; // 用于创建表的SQL语句
 
 public MyHelper(Context context, String name, CursorFactory factory, int version) {
	super(context, DatabaseStatic.DATABASE_NAME, null, DatabaseStatic.DATABASE_VERSION);
 }
 
 public MyHelper(Context context) {
  super(context, DatabaseStatic.DATABASE_NAME, null, DatabaseStatic.DATABASE_VERSION);
 }
 
 @Override
 public void onCreate(SQLiteDatabase db) {
  Log.i("UseDatabase", "创建数据库");
  Toast.makeText(myContext, "创建数据库", Toast.LENGTH_SHORT).show();
  db.execSQL(CREATE_TABLE);
 }
 
 @Override
 public void onUpgrade(SQLiteDatabase arg0, int arg1, int arg2) {
 
 } 
}
```

可以看到数据库的创建过程基本需要靠手动拼装，调用`SQLiteOPenHelper.getWriteableDatabase()`或者`SQLiteOPenHelper.getReadableDatabase()`时，如果数据库不存在，那么会调用`onCreate`方法，然后执行数据库创建语句创建数据库，如果库已经存在了，则不会调用。

##### 使用Room

- 声明数据库抽象类

```java
@Database(entities = {CustomEntity.class}, version = 1)
public abstract class CustomDB extends RoomDatabase {
    public abstract CustomDao customDao();
}
```

- 初始化数据库对象

```java
CustomDB customDB = Room.databaseBuilder(this, CustomDB.class, "db_study").build();
```

注意：上面这段初始化的代码千万不能在主线程里执行，不然会报错。那能否在主线程里操作呢？其实是可以的：

```java
CustomDB customDB = Room.databaseBuilder(this, CustomDB.class, "db_study")
.allowMainThreadQueries()
.build();
```

加上一句`allowMainThreadQueries()`即可，但是⚠️非常⚠️不推荐数据库操作在主线程，数据库的操作时间非常吃配置，CPU和磁盘读写效率非常影响耗时，尤其是低配手机，数据库的耗时特别严重。

从数据库的创建可以发现3个元素：`@Database`、`@Entity`、`@Dao`，这里开始就很熟悉了，学习的思路也是按照这个顺序来的，由简单使用的注解，到略微复杂的注解。

#### 实体定义

```java
@Entity(tableName = "custom")
public class CustomEntity {
    @PrimaryKey(autoGenerate = true)
    public long id;

    @ColumnInfo(name = "dev_id")
    public String deviceId;

    @ColumnInfo(name = "dev_name")
    public String deviceName;
  
  	@Ignore
  	public String iconUrl;
}
```

实体类的定义跟`fastJson`这个库的思路几乎完全一致，上面的代码中定义常用的注解，下面逐个解释一下：

##### @Entity

用来声明数据库的列，创建列表和读写数据都需要使用到。

参数`tableName = "custom"`声明了这个实体对应的表的名称，数据库的名称是初始化的时候传入的。

如果不声明tableName，那么创建表的名称默认就是类名。

##### @ColumnInfo

用来声明成员变量与数据表列的对应关系，即`dev_id`列的值取出来之后将会赋值给`deviceId`，这里的操作跟`fastJson`是一样的。如果不声明的话，那么就需要列名变为deviceId，或者变量名改为dev_id，这样其实会使数据库列名风格或者代码变量名风格变得很奇怪，所以推荐所有需要查询的字段都使用这个注解。

##### @Ignore

声明不参与数据库查询的字段。这个字段有啥意义呢？其实实际业务实施的时候，一个实体既会当做数据库解析的实体用，也会作为上层业务的实体使用，业务层独有的字段可以使用这个注解，最典型的就是`checked`这种标记性的字段。

##### @PrimaryKey

主键注解，声明当前变量对应数据库中的主键，也就是一行数据的唯一标识，一般都会使用`id`作为主键，`autoGenerate = true`意思是字段值在新增时为自增。当然也可以选择其他字段作为主键，需要注意的是主键绝对不能冲突，否则新增行的时候会报错。

#### 数据访问对象

Data Access Object，也就是常见的缩写`Dao`的完整短句，用来写一些与数据库交互的方法，因为数据库交互无非就是`增、删、改、查`，所以下面就会从这4方面对比原始的Android SQLite操作学习。

##### 增

```java
@Dao
public interface CustomDao {
    @Insert(onConflict = OnConflictStrategy.REPLACE)
	List<Long> insertList(List<CustomEntity> CustomEntities);
    
    @Insert(onConflict = OnConflictStrategy.REPLACE)
	long insert(CustomEntity CustomEntity);
}
```

`@Insert`注解用来表示该方法是用来插入数据的，一般就建议写上面2个方法就好了。

`onConflict = OnConflictStrategy.REPLACE`表示当插入主键冲突的时候，旧数据将会被新数据替换。

`返回值`是插入数据的`rowId`，返回的类型也可以写void，不接收结果。

对比以前老的数据插入方式：

```java
public void addData(String tableName, String[] keys, String[] values) {
    SQLiteDataBase db = helper.getWritableDatabase();
    ContentValues cv = new ContentValues();
    for (int i = 0; i < key.length; i++) {
        cv.put(key[i], values[i]);
    }
    db.insert(tableNames, null, cv);
    cv.clear();
}
```

对比就会发现，增加数据的过程还需要自己手动去拆分字段和值，模型大了之后，写着非常繁琐，很影响效率。

##### 删

```java
@Dao
public interface CustomDao {
    @Delete
	int deletetList(List<CustomEntity> CustomEntities);
    
    @Delete
	int delete(CustomEntity CustomEntity);
}
```

删除的写法也很简单，只需要写注解就好了，int返回的是删除的行数，如果不需要结果，方法的返回值可以定义为void。那删除时带的参数有什么意义呢？其实删除的时候，只是对比了传入参数的主键，有相同的主键才会执行删除动作。

##### 改

```java
@Dao
public interface CustomDao {
    @Update
	int updatetList(List<CustomEntity> CustomEntities);
    
    @Update
	int update(CustomEntity CustomEntity);
}
```

更新和删除基本上都是一样的，匹配机制也是通过主键，只是一个执行的是删除，一个执行的是覆盖数据操作。

##### 查

```java
@Dao
public interface CustomDao {
    @Query("select * from custom")
	int List<CustomEntity> getAll();
}
```

通过这里其实我就发现了问题，`@Query`注解其实就是需要自己写sql语句的，没有办法靠简单的传参处理问题，不过相对于原始的方式还是很简单的：

```java
public List<CustomEntity> getAll() {
    List<CustomEntity> list = new ArrayList<>();
    SQLiteDataBase db = helper.getWritableDatabase();
    Cursor cursor = db.query("custom", null, null, null, null, null, null);
    if (cursor == null) { 
        return list;
    }
    CustomEntity entity;
    while(cursor.moveToNext()) {
        entity = new CustomEntity();
        entity.devId = cursor.getString(cursor.getColumIndex("dev_id"));
        entity.devName = cursor.getString(cursor.getColumIndex("dev_name"));
        list.add(entity);
    }
    cursor.close();
    return list;
}
```

原始的方式里需要通过游标逐个取字段，然后填充实体后再返回数据。

这里说明一下特殊情况:

```java
@Query("DELETE FROM custom where dev_id=:devId")
void delete(String devId);
```

因为Query注解的作用，其实拿它来删除数据也是可以的。

##### 拓展

上面的查询语法其实是最简单的，但是有时候会有一些特殊需要，比如想查所有设备的名字：

```java
@Query("SELECT dev_name FROM custom")
List<String> getAllNames();
```

如果想通过设备名称查询设备，可以这么做：

```java
@Query("SELECT * FROM custom WHERE dev_name LIKE :search")
List<CustomEntity> searchByName(String search);
```

一般情况下，APP端的数据库都不会寸比较复杂的数据，因为查询效率问题和功能设计的问题，很多时候关键的数据都是存在云端的，客户端很少会有复杂的表，但学习的时候还是触及一下联表查询的知识，这里以官方demo为例：

```java
@Query("SELECT * FROM book " +
       "INNER JOIN loan ON loan.book_id = book.id " +
       "INNER JOIN user ON user.id = loan.user_id " +
       "WHERE user.name LIKE :userName")
public List<Book> findBooksBorrowedByNameSync(String userName);
```

这里的语法是这么一个意思：

从`user`表中查到`user.name`列与`userName`能匹配到的所有用户，然后将获得的`user`数据的`user.id`与表`loan`中的`loan.user_id`匹配获取到所有的借用数据，最后通过所有借用信息的`loan.book_id`字段在`book`表中查出所有的书本信息，这些书本信息最终放在`List<Book>`集合中返回给业务层。

### 总结

对于需要快速渲染的客户端来说，数据库使用前必须考虑合理性和需求，例如本来就很少的数据，而且需要反复读写，也不一定需要持久化就不要用数据库来存储。数据量不太的情况，json文件也是一个比较好的方式，从实战的经验看来，低性能手机上的文件解析比数据库的频繁查询要高效很多。

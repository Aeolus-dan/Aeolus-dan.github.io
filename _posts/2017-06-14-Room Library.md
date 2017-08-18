# <font color=#88ffb0>Room Persistence Library</font>

当使用sqlite全部功能的时候, room在sqlite上提供了一种抽象层以允许流畅的访问数据.

*Note:如何在自己的项目中使用room, 请参考<a href="https://developer.android.com/topic/libraries/architecture/adding-components.html">添加库到你的项目</a>*

处理大量级结构化数据的应用将数据持久化在本地是很有意义的，最常见的办法是缓存相关的数据,这样的话,当设备无法访问网络时离线用户仍然可以
相关内容。当设备连接到网络后,热河用户初始化的内容更改都将同步到服务器.<br><br>

Android核心的框架提供了处理原始数据的内在支持, 它们相当的低级并且需要大量的时间和精力尽管这些API相当有用.

- 没有对原始SQL查询的编译验证.当你的数据图发生改变时你需要手动更新受影响的SQL查询,这个操作不但耗时还容易出错.
- 在SQL查询和Java数据对象的转换上 需要大量的模板代码.

Room在sqlite上提供抽象层的同时解决了这些问题的.

下面是Room的主要组成部分:

- Database: 你可以使用它来创建一个数据库的holder, 这个注解定义了实体的清单和数据访问对象的清单,同时也是底层数据库连接点.
  该注解类应该是一个继承RoomDatabase的抽象类. 在运行时,你可以使用Room.databaseBuilder或者Room.inMemoryDatabaseBuilder方法来获取它的实例.
- Entity: 声明一个持有一行数据的类,对于每个entity，都会创建一个数据库表来保存item.在Database类上必须通过entities来引用一个实体对象.
  实体中的每个字段都将会保存到数据库中,除非某个字段被声明未@Ignore。<br>

  *Note:实体类可以有空的构造函数(如果DAO可以访问它的每个字段)或者一个包含全部字段的构造函数,Room也可以使用全部字段的构造或者部分字段的构造函数.*

- DAO: 代表一个类或者接口作为DAO(Data Access Object)数据访问对象. DAO是room的主要部分而且定义了访问数据库的方法.
  用@Database注解的类必须包含一个具有0个参数的抽象方法，并且返回用@Dao注解的类. 当在编译代码时，Room创建一个该类的实现.

  *Note:通过使用DAO类而不是查询构建器或者直接访问数据库,可以分离数据库体系结构的不同组件。DAO允许您在测试应用程序时轻松地模拟数据库访问*

![Room Components relationships](../../../../../img/room_architecture.png)

*这些内容的关系以及它们的功能*

下面的代码片段包含了具有一个实体和一个dao的示例数据库配置:<br><br>
User.java
```
@Entity
public class User {
    @PrimaryKey
    private int uid;

    @ColumnInfo(name = "first_name")
    private String firstName;

    @ColumnInfo(name = "last_name")
    private String lastName;

    // Getters and setters are ignored for brevity,
    // but they're required for Room to work.
}
```

UserDao.java
```
@Dao
public interface UserDao {
    @Query("SELECT * FROM user")
    List<User> getAll();

    @Query("SELECT * FROM user WHERE uid IN (:userIds)")
    List<User> loadAllByIds(int[] userIds);

    @Query("SELECT * FROM user WHERE first_name LIKE :first AND "
           + "last_name LIKE :last LIMIT 1")
    User findByName(String first, String last);

    @Insert
    void insertAll(User... users);

    @Delete
    void delete(User user);
}
```

AppDatabase.java
```
@Database(entities= {User.class}, version=1)
public abstract class AppDatabase extends RoomDatabase{
    public abstract UserDao userDao();
}
```

创建以上文件之后, 你可以通过如下代码创建数据库:
```
AppDatabase db = Room.databaseBuilder(getApplicationContext(),
        AppDatabase.class, "database-name").build();
```

*当实例化AppDatabase对象时最好使用单例模式，因为每个RoomDatabase的创建开销都比较大.而且你很少需要访问多个实例.*

## Entities
当类被@Entity注释并且在@Database注释的entities属性中引用时，Room会在数据库中为该实体创建一个数据库表。<br>
默认情况下，Room为实体中定义的每个字段创建一个列。如果一个实体具有不想持久化的字段，可以使用@Ignore对它们进行注释，如下面的代码片段所示：
```
@Entity
class User {
    @PrimaryKey
    public int id;

    public String firstName;
    public String lastName;

    @Ignore
    Bitmap picture;
}
```

为了能持久化这些字段, Room必须要能访问到它， 你可以声明该字段为public，或者提供该字段的getter和setter方法. 如果使用后者,那么请记住它们是基于Java bean 转换的在room上.

### Primary key
  每一个实体类必须至少有一个字段作为主键,即使它只有一个字段, 你仍然要用@Primary key声明这个字段.如果您希望room为实体分配自动ID，可以设置@PrimaryKey的autoGenerate属性。
  如果实体具有复合主键，则可以使用@Entity注释的primaryKeys属性，如以下代码片段所示：
  ```
  @Entity(primaryKeys = {"firstName", "lastName"})
  class User {
      public String firstName;
      public String lastName;

      @Ignore
      Bitmap picture;
  }
  ```
默认情况下, room用该类的类名作为数据库的表名，如果你想单独定义表名,可以给@Entity注解添加tableName属性,参考如下:
```
@Entity(tableName = "users")
class User{
    ...
}
```
*注意:Sqlite表名不区分大小写*

类似于tableName属性，room使用字段名称作为列名称,如果你不想这样的话，可以使用@ColumnInfo注解的name属性指定 like blow:
```
@Entity(tableName = "users")
class User{
    @PrimaryKey
    public int id;

    @ColumnInfo(name = "first_name")
    public String firstName;

    @ColummnInfo(name = "last_name")
    public String lastName;

    @Ignore
    Bitmap picture;
}
```
### Indices and uniqueness

根据访问数据的方式，可能希望对数据库中的某些字段进行索引，以加快查询速度。要向实体添加索引，
请在@Entity注释中包含indices属性，列出要包含在索引或组合索引中的列的名称。以下代码片段演示了此注释过程:
```
@Entity(indices = {@Index("name"),@Index(value= {"last_name","address"})})
class User {
    @PrimaryKey
    public int id;

    public String firstName;
    public String address;

    @ColumnInfo(name = "last_name")
    public String lastName;

    @Ignore
    Bitmap picture;
}
```

有时,数据库中的某些字段或者某些组字段必须是唯一的,通过将@Index注释的唯一属性设置为true来强制执行此唯一性属性。
以下代码示例可防止表具有包含firstName和lastName列的相同值集合的两行:
```
@Entity(indices = {@Index(value = {"first_name","last_name"}, unique = true)})
class User {
    @PrimaryKey
    public int id;

    @ColumnInfo(name = "first_name")
    public String firstName;

    @ColumnInfo(name = "last_name")
    public String lastName;

    @Ignore
    Bitmap picture;
}
```

### RelationShips
因为Sqlite是关系型数据库,因此你可以指定对象之间的关系,即使大多数ORM库允许实体对象相互引用，但是Room明确禁止此操作。具体请查看<a href="https://developer.android.com/topic/libraries/architecture/room.html#no-object-references">实体之间没有对象引用</a>
<br>即使不能使用直接关系，Room仍允许在实体之间定义Foreign Key约束。<br>
例如，如果有另一个名为Book的实体，可以使用@ForeignKey注释来定义与User实体的关系，如以下代码片段所示:
```
@Entity(foreignKeys = @ForeginKey(entity = User.class,
                                  parentColumns = "id",
                                  childColumns = "user_id" ))
class Book{
    @PrimaryKey
    public int bookId;

    public String title;

    @ColumnInfo(name = "user_id")
    public int userId;
}
```
外键非常强大，因为它们允许指定引用实体更新时发生的情况。例如，如果通过在@ForeignKey注释中包含onDelete = CASCADE来删除用户的相应实例，则可以让SQLite删除用户的所有图书。
*注意:SQLite处理@Insert（OnConflict = REPLACE）作为一组REMOVE和REPLACE操作，而不是单个UPDATE操作。这种替换冲突值的方法可能会影响您的外键约束。有关更多详细信息请查看<a href="https://sqlite.org/lang_conflict.html">Sqlite 文档</a>*

### Nested objects
有时，希望在数据库逻辑中表达一个实体或POJO对象，即使该对象包含多个字段。在这些情况下，可以使用@Embedded注释来表示要在表中分解为子字段的对象。然后，可以像其他单独的列一样查询嵌入的字段。
<br>例如，我们的User类可以包含一个Address类型的字段，它表示一个名为street，city，state和postCode的字段的组合。要将表格中单独存储组合列，在User类中包含一个用@Embedded注释的Address字段，如下面的代码段所示:
```
class Address {
    public String street;
    public String state;
    public String city;

    @ColumnInfo(name = "post_code")
    public int postCode;
}

@Entity
class User {
    @PrimaryKey
    public int id;

    public String firstName;

    @Embedded
    public Address address;
}
```
User对象的表包含以下名称的列：id，firstName，street，state，city和post_code。<br>

*注意:嵌入字段还可以包括其他嵌入字段。*
如果实体具有多个相同类型的嵌入字段，则可以通过设置prefix属性来保留每个列的唯一性。room然后将提供的值添加到嵌入对象中的每个列名的开头.

## Data Access Objects (DAOs)
Room的主要部分是Dao类，它们以轻便的方式抽象访问数据库<br>

每个Dao可以是一个接口或者抽象类。 如果是抽象类， 它可以可选地具有使RoomDatabase作为其唯一参数的构造函数。
*注意:room不允许访问主线程上的数据库，除非在构建器上调用allowMainThreadQueries（），因为它可能会长时间地阻塞用户界面。异步查询（返回LiveData或RxJava Flowable的查询）将免除此规则，因为它们在需要时异步运行后台线程上的查询*

### Method for convenience
可以使用DAO类来表示多个方便的查询。本文档包括几个常见的例子。
#### Insert
当你创建一个DAO并且以@Insert注解某个方法时，Room生成一个在单个事务中将所有参数插入数据库的实现。以下代码片段显示了几个示例查询:
```
@Dao
public interface MyDao{
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    public void insertUsers(Usert... users);

    @Insert
    public void insertBothUsers(User user1,User user2);

    @Insert
    public void insertUsersAndFriends(User user, List<User> friends);
}
```
如果@Insert方法只接收到一个参数，它可以返回一个long，这是插入项的rowId. 如果参数是数组或集合，它应该返回long[]或List <Long>。<br>

#### Update
更新是一种方便的方法，用于更新数据库中以参数给出的一组实体。它使用与每个实体的主键匹配的查询。以下代码段演示了如何定义此方法:
```
@Dao
public interface MyDao {
    @Update
    public void updateUsers(User... users);
}
```
#### Delete
删除是从数据库中删除一组作为参数给出的实体的方便方法。它使用主键来查找要删除的实体。以下代码段演示了如何定义此方法:
```
@Dao
public interface MyDao {
    @Delete
    public void deleteUsers(User... users);
}
```
虽然通常不是必需的，您可以使用此方法返回一个int值，指示从数据库中删除的行数.

### Method using @Query
@Query是DAO类中的主要注解, 它允许用户读写数据库。每个@Query方法在编译时进行验证，因此如果查询有问题，则会发生编译错误而不是运行时故障。
<br>
room还会验证查询的返回值，以便如果返回对象中的字段名称与查询响应中的相应列名称不匹配，则room将以以下两种方式之一提醒:

-  如果只有一些字段名称匹配，则会发出警告。
-  如果没有一个字段名称匹配, 则会发出错误.

#### 简单的查询
    @Dao
    public interface MyDao{
        @Query("SELECT * FROM user")
        public User[] loadAllUsers();
    }
这是一个非常简单的查询，加载所有用户。在编译时，Room知道它正在查询用户表中的所有列。如果查询包含语法错误，或者如果用户表不存在于数据库中，则Room会在应用程序编译时显示相应消息的错误。
#### 传递参数到查询注解
大多数情况下，需要将参数传递给查询以执行过滤操作，例如仅显示某个年龄以上的用户。要完成此任务，请在Room注释中使用方法参数，如以下代码片段所示:
```
@Dao
public interface MyDao{
    @Query("SELECT * FROM user WHERE age > :minAge")
    public User[] loadAllUsersOlderThan(int minAge);
}
```
当在编译时处理此查询时，Room将:minAge绑定参数与minAge方法参数匹配。room使用参数名称执行匹配。如果不匹配，应用程序编译时会发生错误。
<br><br>还可以在查询中传递多个参数或多次引用它们，如以下代码片段所示：
```
@Dao
public interface MyDao{
    @Query("SELECT * FROM user WHERE age BETWEEN :minAge AND :maxAge")
    public User[] loadAllUserBetweenAges(int minAge,int maxAge);

    @Query("SELECT * FROM user WHERE first_name LIKE :search" + "OR last_name LIKE :search")
    public List<User> findUserWithName(String search);
}
```

#### 返回列的子集
大多数时候，你只需要获得一个实体的几个字段。例如，UI可能仅显示用户的名字和姓氏，而不是每个用户的详细信息。通过仅获取应用程序UI中显示的列，可以节省宝贵的资源，并且查询更快速地完成。
<br><br>
只要可以将结果列的集合映射到返回的对象，可以从查询返回任何Java对象.例如，可以创建以下POJO来获取用户的名字和姓氏:
```
public class NameTuple {
    @ColumnInfo(name="first_name")
    public String firstName;

    @ColumnInfo(name="last_name")
    public String lastName;
}
```
现在，可以在查询方法中使用此POJO:
```
    @Dao
    public interface MyDao {
        @Query("SELECT first_name, last_name FROM user")
        public List<NameTuple> loadFullName();
    }
```
Room了解查询返回first_name和last_name列的值，并将这些值映射到NameTuple类的字段。因此，Room可以生成正确的代码。如果查询返回太多的列，或者NameTuple类中不存在的列，则会显示警告。
*注意:POJOs也是可以使用@Embedded注解*

#### 传递集合给参数
一些查询可能需要传递可变数量的参数，直到运行时才知道参数的确切数量。例如，可能希望从区域的子集中检索有关所有用户的信息。一个参数表示一个集合，并在运行时根据所提供的参数数量自动展开它，room就会明白。
```
@Dao
public interface MyDao{
    @Query("SELECT first_name,last_name FROM user WHERE region IN (:regins)")
    public List<NameTuple> loadUsersFromRegions(List<String> regions);
}
```

#### 可观察的查询
执行查询时，经常希望应用程序的UI在数据更改时自动更新。为了达到这个目的, 可以返回<a href="https://developer.android.com/reference/android/arch/lifecycle/LiveData.html">LiveData<a>
当数据库更新时，room会生成所有必需的代码来更新LiveData。
```
@Dao
public interface MyDao {
    @Query("SELECT first_name, last_name FROM user WHERE region IN (:regions)")
    public LiveData<List<User>> loadUsersFromRegionsSync(List<String> regions);
}
```

#### RxJava
room还可以从定义的查询中返回RxJava2 Publisher和Flowable对象。要使用此功能，请将来自Room组的android.arch.persistence.room:rxjava2库添加到构建Gradle依赖项中。然后，可以返回RxJava2中定义的类型对象，如以下代码片段所示:
```
@Dao
public interface MyDao {
    @Query("SELECT * from user where id = :id LIMIT 1")
    public Flowable<User> loadUserById(int id);
}
```
#### 游标直接访问
如果应用程序的逻辑需要直接访问返回行，则可以从查询中返回一个Cursor对象，如以下代码片段所示:
```
@Dao
public interface MyDao {
    @Query("SELECT * FROM user WHERE age > :minAge LIMIT 5")
    public Cursor loadRawUsersOlderThan(int minAge);
}
```
*警告:非常不鼓励使用Cursor API，因为它不能保证行是存在还是行包含什么值,仅当已经具有期望光标的代码并且不能轻易重构时，才能使用此功能*

#### 查询多张表
一些查询可能需要访问多个表以计算结果。room允许你编写任何查询，所以你也可以连接表。此外，如果响应是可观察的数据类型，例如Flowable或LiveData，Room会监视查询中引用的无效的所有表。
<br><br>
以下代码片段显示了如何执行表连接以整合包含借阅书的用户的表格和包含目前借出的图书数据的表格之间的信息:
```
@Dao
public interface MyDao {
   @Query("SELECT * FROM book"+
            "INNER JOIN load ON load.book_id = book.id"+
            "INNER JOIN user ON user.id = load.user_id"+
            "WHERE user.name LIKE :userName"
            )
   public List<Book> findBooksBorrowedByNameSync(String userName);
}
```
也可以从这些查询返回POJO。例如，可以编写一个加载用户的查询及其宠物的名称，如下所示:
```
@Dao
public interface MyDao {
   @Query("SELECT user.name AS userName, pet.name AS petName "
          + "FROM user, pet "
          + "WHERE user.id = pet.user_id")
   public LiveData<List<UserPet>> loadUserAndPetNames();

   // You can also define this class in a separate file, as long as you add the
   // "public" access modifier.
   static class UserPet {
       public String userName;
       public String petName;
   }
 }
```


### 使用类型转换器
room内置了原始转换器及其包装。但是，有时会使用希望将数据存储在单个列中的自定义数据类型。要为自定义类型添加这种支持，可以提供一个TypeConverter，它将一个自定义类转换为Room可以保留的已知类型。
<br><br>
例如，如果我们要保留Date的实例，我们可以编写以下TypeConverter来存储数据库中的等效的Unix时间戳记:
```
public class Converters {
    @TypeConverter
    public static Date fromTimestamp(Long value) {
        return value == null ? null : new Date(value);
    }

    @TypeConverter
    public static Long dateToTimestamp(Date date) {
        return date == null ? null : date.getTime();
    }
}
```
接下来，将@TypeConverters注释添加到AppDatabase类，以便Room可以使用为该AppDatabase中的每个实体和DAO定义的转换器:
```
@Database(entities = {User.java}, version = 1)
@TypeConverters({Converters.class})
public abstract class AppDatabase extends RoomDatabase {
    public abstract UserDao userDao();
}
```
使用这些转换器，可以在其他查​​询中使用自定义类型，就像使用原始类型一样，如以下代码片段所示:
```
@Entity
public class User {
    ...
    private Date birthday;
}
```
```
@Dao
public interface UserDao {
    ...
    @Query("SELECT * FROM user WHERE birthday BETWEEN :from AND :to")
    List<User> findUsersBornBetweenDates(Date from, Date to);
}
```
还可以将@TypeConverter限制为不同的范围，包括单个实体，DAO和DAO方法


## 数据库版本迁移
当添加和更改应用程序中的功能时，需要修改实体类以反映这些更改。当用户更新到最新版本的应用程序时，不希望它们丢失所有现有数据，特别是如果无法从远程服务器恢复数据。<br><br>
room允许以这种方式编写迁移类来保留用户数据。每个Migration类都指定一个startVersion和endVersion。在运行时，Room运行每个Migration类的migrate方法，使用正确的顺序将数据库迁移到更高版本。
*警告:如果没有提供必要的迁移，则房间将重建数据库，这意味着将丢失数据库中的所有数据*
```
Room.databaseBuilder(getApplicationContext(), MyDb.class, "database-name")
        .addMigrations(MIGRATION_1_2, MIGRATION_2_3).build();

static final Migration MIGRATION_1_2 = new Migration(1, 2) {
    @Override
    public void migrate(SupportSQLiteDatabase database) {
        database.execSQL("CREATE TABLE `Fruit` (`id` INTEGER, "
                + "`name` TEXT, PRIMARY KEY(`id`))");
    }
};

static final Migration MIGRATION_2_3 = new Migration(2, 3) {
    @Override
    public void migrate(SupportSQLiteDatabase database) {
        database.execSQL("ALTER TABLE Book "
                + " ADD COLUMN pub_year INTEGER");
    }
};
```
*警告:为了使迁移逻辑保持正常运行，请使用完整查询，而不是引用代表查询的常量。*

迁移过程完成后，Room会验证模式以确保迁移正确。如果Room发现问题，它会引发包含不匹配信息的异常.

### 测试迁移
迁移并不是简单的写入，并且无法正确写入它们可能会导致应用程序中的崩溃循环。为了保持应用的稳定性，应该事先测试迁移。 Room提供了一个测试Maven工件来协助测试过程。但是，要使此工件正常工作，需要导出数据库的架构

### Exporting schemas
汇编后，Room将数据库的架构信息导出为JSON文件。要导出架构，请在build.gradle文件中设置room.schemaLocation注解处理器属性，如以下代码片段所示:
```
android {
    ...
    defaultConfig {
        ...
        javaCompileOptions {
            annotationProcessorOptions {
                arguments = ["room.schemaLocation":
                             "$projectDir/schemas".toString()]
            }
        }
    }
}
```
应该将导出的JSON文件（其表示数据库的架构历史记录）存储在版本控制系统中，因为它允许Room创建旧版本的数据库以进行测试。<br><br>
要测试这些迁移，请将来自Room的android.arch.persistence.room:testing Maven工件添加到测试依赖项中，并将模式位置添加为资产文件夹，如以下代码片段所示:
```
android {
    ...
    sourceSets {
        androidTest.assets.srcDirs += files("$projectDir/schemas".toString())
    }
}
```
测试包提供了一个MigrationTestHelper类，可以读取这些模式文件。它也是一个Junit4 TestRule类，所以它可以管理创建的数据库。<br><br>
示例迁移测试将显示在以下代码段中:
```
@RunWith(AndroidJUnit4.class)
public class MigrationTest {
    private static final String TEST_DB = "migration-test";

    @Rule
    public MigrationTestHelper helper;

    public MigrationTest() {
        helper = new MigrationTestHelper(InstrumentationRegistry.getInstrumentation(),
                MigrationDb.class.getCanonicalName(),
                new FrameworkSQLiteOpenHelperFactory());
    }

    @Test
    public void migrate1To2() throws IOException {
        SupportSQLiteDatabase db = helper.createDatabase(TEST_DB, 1);

        // db has schema version 1. insert some data using SQL queries.
        // You cannot use DAO classes because they expect the latest schema.
        db.execSQL(...);

        // Prepare for the next version.
        db.close();

        // Re-open the database with version 2 and provide
        // MIGRATION_1_2 as the migration process.
        db = helper.runMigrationsAndValidate(TEST_DB, 2, true, MIGRATION_1_2);

        // MigrationTestHelper automatically verifies the schema changes,
        // but you need to validate that the data was migrated properly.
    }
 }
```

### 测试数据库
当应用程序运行测试时，如果没有测试数据库本身，则不需要创建完整的数据库。room允许你轻松地模拟测试中的数据访问层。这个过程是可能的，因为你的DAO不会泄漏数据库的任何细节。测试其余的应用程序时，应该创建DAO类的模拟或假的实例。
<br><br>
以下有2种方式测试你的数据库:

- on your host development machine.
- on an android device.

#### Testing on an Android device
用于测试数据库实现的推荐方法是编写在Android设备上运行的JUnit测试。因为这些测试不需要创建一个活动，所以它们应该比UI测试执行得更快。
<br><br>
设置测试时，应该创建数据库的内存中版本，使测试更加密封，如以下示例所示:
```
@RunWith(AndroidJUnit4.class)
public class SimpleEntityReadWriteTest {
    private UserDao mUserDao;
    private TestDatabase mDb;

    @Before
    public void createDb() {
        Context context = InstrumentationRegistry.getTargetContext();
        mDb = Room.inMemoryDatabaseBuilder(context, TestDatabase.class).build();
        mUserDao = mDb.getUserDao();
    }

    @After
    public void closeDb() throws IOException {
        mDb.close();
    }

    @Test
    public void writeUserAndReadInList() throws Exception {
        User user = TestUtil.createUser(3);
        user.setName("george");
        mUserDao.insert(user);
        List<User> byName = mUserDao.findUsersByName("george");
        assertThat(byName.get(0), equalTo(user));
    }
}
```

## 附录:实体之间没有对象引用
将数据库中的关系映射到相应的对象模型是一个常见的做法，在服务器端可以很好地运行，在访问它们时，它们可以很方便地加载字段。<br><br>

然而，在客户端，延迟加载是不可行的，因为它可能发生在UI线程上，并且在UI线程中查询磁盘上的信息会产生显着的性能问题。 UI线程有大约16ms的时间来计算和绘制一个活动的更新的布局，所以即使一个查询只需要5 ms，你的应用程序仍然可能耗尽时间来绘制框架，引起明显的破坏。更糟糕的是，如果并行运行单独的事务，或者设备忙于其他磁盘重的任务，则查询可能需要更多时间才能完成。但是，如果您不使用延迟加载，则应用程序将获取比其需要的更多数据，从而创建内存消耗问题。
<br><br>ORM通常将此决定留给开发人员，以便他们可以为应用程序的用例做最好的事情。不幸的是，开发人员通常最终在他们的应用程序和UI之间共享模型。随着UI随着时间的推移而变化，难以预料和调试的问题出现。
<br><br>例如，使用加载Book对象列表的UI，每本书都有一个Author对象。可能最初设计查询以使用延迟加载，以便Book的实例使用getAuthor方法来返回作者。第一次调用getAuthor调用查询数据库。一段时间后,你会意识到您需要在应用程序的使用者介面中显示作者姓名。可以轻松添加方法调用，如以下代码片段所示:
```
authorNameTextView.setText(user.getAuthor().getName());
```
然而，这个看似无辜的变化导致在主线程上查询作者表。<br><br>
如果你热切地查询作者信息，如果您不再需要数据，就会难以更改数据的加载，例如应用程序的UI不再需要显示有关特定作者的信息的情况。因此，你的应用程序必须继续加载不再显示的数据。如果作者类引用另一个表，例如使用getBooks方法，这种情况会更糟。
<br><br>
由于这些原因，Room禁止实体类之间的对象引用。相反，你必须显式请求你的应用程序需要的数据。

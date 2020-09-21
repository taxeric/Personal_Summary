没有比官方文档更好的文档了
# 依赖
```java
dependencies {
    ......你的其他依赖
    
    def room_version = "2.2.5"
    implementation "androidx.room:room-runtime:$room_version"
    
    //如果你是kotlin就使用
    kapt "androidx.room:room-compiler:$room_version"
    //如果你是java就使用
    annotationProcessor "androidx.room:room-compiler:$room_version"
    
    //可选-Kotlin扩展和协程对Room的支持
    implementation "androidx.room:room-ktx:$room_version"
    
    //可选-RxJava对Room的支持
    implementation "androidx.room:room-rxjava2:$room_version"
    
    //可选-Guava对Room的支持，包括Optional和ListenableFuture
    implementation "androidx.room:room-guava:$room_version"
    
    //可选-需要用到相关测试工具的话
    testImplementation "androidx.room:room-testing:$room_version"
}
```
# 相关注解
- @Entity 标识数据库映射关系的类，可以指定表名等，不指定的话默认表=名和类名一致，索引等信息
- @PrimaryKey 指定主键，可设置自动增长
- @ColumnInfo 列信息
- @Ignore 标识忽略此属性，不生成对应字段

- @Dao 标识Dao层注解，编译时会生成该接口或抽象类的实现
- @Insert 标识该抽象方法为插入数据库，入参为@Entity标注的对象
- @Query 标识该抽象方法为查询数据库，需传入**SQL语句**
- @Delete 标识该抽象方法为删除数据库，入参为@Entity标注的对象
- @Update 标识该抽象方法为更新数据库，入参为@Entity标注的对象

- @Database 标识该类为数据库操作类，一般为**单例**对象，入参为@Entity映射的class集合，version处理版本升级相关，通过此类可拿到相关Dao对象进行操作
# 相关类
- RoomDatabase 抽象类，处理数据库初始化及Dao对象生成相关操作
- Room 普通类，采用Builder模式，用于生成具体的RoomDatabase子类对象
# 简单使用
### 建实体类
```java
@Entity(tableName = "tb_user")
public class User {

    @PrimaryKey
    @NonNull
    private String IDCard;
    @ColumnInfo
    private String name;
    @ColumnInfo
    private int age;

    @NonNull
    public String getIDCard() {
        return IDCard;
    }

    public void setIDCard(@NonNull String IDCard) {
        this.IDCard = IDCard;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }

    public static String createID(){
        return UUID.randomUUID().toString().replace("-", "");
    }

    @Override
    public String toString() {
        return "User{" +
                "IDCard='" + IDCard + '\'' +
                ", name='" + name + '\'' +
                ", age=" + age +
                '}';
    }
}
```
PS：
1. 一定要有get、set方法，否则会报错，起码我报错了（:D）
2. 除空参构造外的其他构造都应使用@Ignore标识

### 建Dao类
```java
@Dao
public interface UserDao {

    /**
     * onConlict表示插入冲突时的处理策略，默认ABORT，有如下值
     * REPLACE： 替换旧数据同时继续事务
     * ABORT：   终止事务
     * IGNORE：  忽略冲突
     * 此外还有 ROLLBACK和FAIL，分别表示回滚事务和事务失败，但已被废弃
     * @param user
     */
    //--------增--------
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    void insert(User user);

    //--------删--------
    @Delete
    void delete(User user);

    @Query("delete from tb_user where IDCard = :id")
    void deleteByID(String id);

    //--------改--------
    @Query("update tb_user set name = :name and age = :age where IDCard = :id")
    void update(String name, int age, String id);

    //--------查--------
    @Query("select * from tb_user")
    List<User> queryAll();

    @Query("select * from tb_user where IDCard = :id")
    User queryByID(String id);

    @Query("select * from tb_user where name like :name")
    List<User> queryByName(String name);
    //关于模糊查询，可以在传参时传入%，比如调用queryByName("赵%")，就查找赵xxx
}
```

### 使用Database
```java
@Database(entities = {User.class}, version = 1)
public abstract class AppDatabase extends RoomDatabase {

    private static AppDatabase INSTANCE;
    public static void init(Context context){
        if (INSTANCE == null){
            synchronized (AppDatabase.class){
                if (INSTANCE == null){
                    INSTANCE = Room.databaseBuilder(
                            context,
                            AppDatabase.class,
                            "data.db"
                    ).allowMainThreadQueries().build();
                }
            }
        }
    }

    public abstract UserDao userDao();
}
```

PS：
1. @Database的`entities`参数可接受多class
2. 应该使用单例来创建RoomDatabase，因为实例化应该RoomDatabase是非常消耗资源的
3. allowMainThreadQueries方法作用是禁用主线程检查，如果没有配置该方法，则**必须在子线程进行数据操作**

### Activity使用
```java
public class MainActivity extends AppCompatActivity {

    private final String TAG = "EricLog";
    private final ArrayList<User> users = new ArrayList<>();

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        users.addAll(AppDatabase.getInstance().userDao().queryAll());
        if (users.isEmpty()) {
            insert();
        } else {
            String ID = users.get(1).getIDCard();
            Log.i(TAG, "query size = " + users.size());
            AppDatabase.getInstance().userDao().deleteByID(ID);
            users.clear();
            users.addAll(AppDatabase.getInstance().userDao().queryAll());
            Log.i(TAG, "deleted size = " + users.size());
            if (!users.isEmpty()){
                for (User user : users){
                    Log.i(TAG, user.toString());
                }
            }
        }
    }

    private void insert() {
        User user1 = new User();
        user1.setIDCard(User.createID());
        user1.setName("Eric");
        user1.setAge(20);
        User user2 = new User();
        user2.setIDCard(User.createID());
        user2.setName("Gerry");
        user2.setAge(22);
        User user3 = new User();
        user3.setIDCard(User.createID());
        user3.setName("Chad");
        user3.setAge(24);
        
        users.add(user1);users.add(user2);users.add(user3);
        AppDatabase.getInstance().userDao().insert(users);
        users.clear();
        users.addAll(AppDatabase.getInstance().userDao().queryAll());
        Log.i(TAG, "size = " + users.size());
        if (!users.isEmpty()){
            for (User user : users){
                Log.i(TAG, user.toString());
            }
        }
    }
}
```

# 其他
1. 在创建Bean时要按规范写，如果报错了就看看Build报的啥错，按着改就行
2. Room无法回退版本，即你当前versoin是2，你不能安装version为1的版本，需要卸载重装

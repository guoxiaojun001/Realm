# Realm
Realm使用小记


Realm是一个能够代替SQLite和Core Data的手机数据库。使用 C++ 内核，所以效率非常的高，是 sqlite 的近几倍。 
其实我们自己也可以写demo测试一下存储相同数据，两个数据库所消耗的时间。 
Realm在多个平台上都可以使用，据说数据库文件可以在多个平台上通用。 

添加依赖,在工程目录下的build.gradle中加上
classpath "io.realm:realm-gradle-plugin:1.0.0"

在使用的module中加上
apply plugin: 'realm-android'

首先我们先建立一个数据库 
最好是写一个Dao类，也可以写多个，但是数据库最好不要建多个，维护起来比较困难。

Realm myRealm = null;
    RealmConfiguration defaultConfig;

    public BaseDao(Context context) {
        //定义了一个
        defaultConfig = new RealmConfiguration.Builder(context)
                .name("xxx").schemaVersion(1).deleteRealmIfMigrationNeeded().build();
        myRealm = Realm.getInstance(defaultConfig);
    }

    public Realm getMyRealm() {
        if(myRealm != null)
        return myRealm;
        else return Realm.getInstance(defaultConfig);
    }
    
    其实Realm数据库的创建方式有非常多种。这里只展示了我自己写的。 
Realm数据库的名称是可以不用指定的，如果只有一个数据库，可以不指定，在调试的时候也会省很多事。 
还可以创建一个默认的Config。 
Realm.setDefaultConfiguration(defaultConfig); 
获取Realm的时候 
Realm.getDefaultInstance(); 
看上去简单很多，而且可以不用传context。 
Realm初始化是有非常多配置的。 
.schemaVersion(1) 设置一个版本号，如果没设置，默认是0 
.encryptionKey() 数据库加密 
.deleteRealmIfMigrationNeeded() 迁移方案的一种，当数据库结构改变时，删除原数据库中的所有数据， 这在debug版本的时候非常有用，一旦应用上线，需要保留原数据数据，这个方案就不适用了。 
.migration() 指定一个迁移方案，如果应用上线之后，数据库结构需要发生改变，可以通过配置Migration来解决，用于将原数据库中的数据迁移到新数据库中。

其实还有很多，详细的还是看官方文档（中文版）吧。 
https://realm.io/cn/docs/java/latest/#section

存储

接下来，我们来实现存储一个对象 
比如我们要存储的对象是User 
需要让User对象继承RealmObject,接下来就是在Dao类中添加一个存储User对象的方法。

public void setUser(User user) {
            myRealm.beginTransaction();
            myRealm.copyToRealm(user);
            myRealm.commitTransaction();
    }


是不是非常简单。

存储list也一样，在copyToRealm位置添加一个循环即可。

查询

把刚存进去的user查出来

public User getUser() {
        return myRealm.where(User.class).findFirst();
    }


简单到不行。 
查询list，加一些条件即可。如：.equalTo("type", type) 
等等。 
不过要注意的是，从Realm中查询出来的对象，当realm被关闭的时候，就无法再使用，所以最好的方式是，查询出来的对象，复制到一个新的list当中。

表关联

Realm数据库允许我们存一个list，例如这样

public class User extends RealmObject{
    private int userid;
    private String username;
    private RealmList<Clazz> clazzs;
}


但前提是必须用RealmList保存，并且list中的对象也必须是继承RealmObject的，实际上就和表一样。 
这里写图片描述 
这里写图片描述 
存储的时候直接和存普通对象一样即可。

过滤、排序等等

myRealm.where(Article.class)
                //articleName 字段 以职场动态 开头
                .beginsWith("articleName","职场动态")
                //age 字段在1~6之间的
                .between("age",1,6)
                //name 字段包含 fancy的
                .contains("name","fancy")
                //对articleName 去重
                .distinct("articleName")
                //age 总共
                .sum("age")
                //最大
                .max()
                //数量
                .count()
                //平均值
                .average()
                //按照type字段排序
                .findAllSorted("type");


上述代码不能执行，只是为了说明用法。

Migration 数据库迁移

当数据库结构发生改变的时候，我们需要配置Migration来实现数据迁移的操作。 
首先新建一个MyMigration

public class MyMigration implements RealmMigration {
    @Override
    public void migrate(DynamicRealm realm, long oldVersion, long newVersion) {

    }
}


将新建的Migration配置进去，并删除deleteRealmIfMigrationNeeded 
对应版本号要增加一下，如果开始没有设置版本号，则新版本号设置为1

defaultConfig = new RealmConfiguration.Builder(context)
                .name("xxx")
                .schemaVersion(2)
                .migration(new MyMigration())
                .build();


假如版本号为1的数据又一张 user表，里面有userid，username，password属性。 
现在我要增加一张新的表dog，并且要在user表中加一个description属性。 
我们的Migration中应该如何去写呢。

public class MyMigration implements RealmMigration {
    @Override
    public void migrate(DynamicRealm realm, long oldVersion, long newVersion) {
        RealmSchema schema = realm.getSchema();
        if (oldVersion == 1) {
            schema.create("Dog")
                    .addField("name", String.class)
                    .addField("age", int.class);
            schema.get("User")
                    .addField("description", String.class);
            oldVersion++;
        }
    }
}



也还算比较简单的。

数据库内容查看

首先在整个工程目录下build.gradle中添加：

allprojects {
    repositories {

        maven {
            url 'https://github.com/uPhyca/stetho-realm/raw/master/maven-repo'
        }
    }
}

然后在使用到realm的module中添加：

debugCompile 'com.facebook.stetho:stetho:1.3.1'
debugCompile 'com.uphyca:stetho_realm:0.9.0'
1
2
在Application中添加一些代码：

Stetho.initialize(
                Stetho.newInitializerBuilder(this)
                        .enableDumpapp(Stetho.defaultDumperPluginsProvider(this))
                        .enableWebKitInspector(RealmInspectorModulesProvider.builder(this)
                                .databaseNamePattern(Pattern.compile("search"))
                                .build())
                        .build());

如果数据库名字是默认的则删掉上面那句 .databaseNamePattern(Pattern.compile("search")) 
即可。

选择编译debug版本，成功运行到手机后，在谷歌浏览器中输入：

chrome://inspect/#devices
1
记住是谷歌浏览器，其他浏览器不行。 
然后就可以查看数据库中的数据了。 


这里面也可以查看sharedPreference中的数据 
点击Local Storage中的自己应用程序的包名就可以查看内容了。


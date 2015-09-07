#Android SQLite最佳实践

##SqliteOpenHelper

它保持一个数据的连接，看起来它可以提供读写连接，实际上则不是，即使向它要一个Read Only的连接，它也会返回一个write连接。

因此，一个SqliteOpenHelper对应着一个connection。即使用多个线程去使用它，SqliteDatabase实例也会使用锁来保证顺序化的操作。

因此，无论有多少个线程，使用同一个SqliteOpenHelper的实例可以保证代码的正确性。如果多个线程同时使用多个SqliteOpenHelper的实例进行读写，则会引发冲突，导致有部分写不成功，更严重的是，如果此时的写操作代码有误，也不会抛出异常，而只是会在Logcat里面打印一条message。

##错误示例一

假设自己实现了SQLiteOpenHelper

	public class DatabaseHelper extends SQLiteOpenHelper { ... }

如果我们分别通过不同的线程去使用它

	// Thread 1
	Context context = getApplicationContext();
	DatabaseHelper helper = new DatabaseHelper(context);
	SQLiteDatabase database = helper.getWritableDatabase();
	database.insert(…);
	database.close();

	// Thread 2
	Context context = getApplicationContext();
	DatabaseHelper helper = new DatabaseHelper(context);
 	SQLiteDatabase database = helper.getWritableDatabase();
	database.insert(…);
	database.close();

这是你将会得到一个错误信息

	android.database.sqlite.SQLiteDatabaseLockedException: database is locked (code 5)

如上面提到的，DatabaseHelper将会建立一个connection，多个connection同时进行写操作会导致错误。

##错误示例二

我们尝试改进上述代码，将DatabaseHelper改为单例模式

	public class DatabaseManager {

		private static DatabaseManager instance;
		private static SQLiteOpenHelper mDatabaseHelper;

		public static synchronized void initializeInstance(SQLiteOpenHelper helper) {
			if (instance == null) {
				instance = new DatabaseManager();
				mDatabaseHelper = helper;
			}
		}

		public static synchronized DatabaseManager getInstance() {
			if (instance == null) {
				throw new IllegalStateException(DatabaseManager.class.getSimpleName() +
					" is not initialized, call initialize(..) method first.");
			}

			return instance;
		}

		public SQLiteDatabase getDatabase() {
			return new mDatabaseHelper.getWritableDatabase();
		}
}

我们尝试调用如下

	// In your application class
	DatabaseManager.initializeInstance(new MySQLiteOpenHelper());
	// Thread 1
	DatabaseManager manager = DatabaseManager.getInstance();
	SQLiteDatabase database = manager.getDatabase()
	database.insert(…);
	database.close();

	// Thread 2
	DatabaseManager manager = DatabaseManager.getInstance();
	SQLiteDatabase database = manager.getDatabase()
	database.insert(…);
	database.close();

这时会导致异常如下：

	java.lang.IllegalStateException: attempt to re-open an already-closed object: SQLiteDatabase

原因是我们使用了同一个数据库连接，getDatabase方法对两个线程返回同一个SQLiteDatabase对象。在线程一关掉它之后，线程二会崩溃。我们需要保证不关闭有人正在使用的SQLiteDatabase，有些人推荐永远不关闭SQLiteDatabase，但是它会导致异常

	Leak found Caused by: java.lang.IllegalStateException: SQLiteDatabase created and never closed

##可用版本

我们可以自行实现一个引用计数

	public class DatabaseManager {

		private int mOpenCounter;

		private static DatabaseManager instance;
		private static SQLiteOpenHelper mDatabaseHelper;
		private SQLiteDatabase mDatabase;

		public static synchronized void initializeInstance(SQLiteOpenHelper helper) {
			if (instance == null) {
				instance = new DatabaseManager();
				mDatabaseHelper = helper;
			}
		}

		public static synchronized DatabaseManager getInstance() {
			if (instance == null) {
				throw new IllegalStateException(DatabaseManager.class.getSimpleName() +
					" is not initialized, call initializeInstance(..) method first.");
			}

			return instance;
		}

		public synchronized SQLiteDatabase openDatabase() {
			mOpenCounter++;
			if(mOpenCounter == 1) {
				// Opening new database
				mDatabase = mDatabaseHelper.getWritableDatabase();
			}
			return mDatabase;
		}

		public synchronized void closeDatabase() {
			mOpenCounter--;
			if(mOpenCounter == 0) {
				// Closing database
				mDatabase.close();
			}
		}
	}

使用方法如下

	SQLiteDatabase database = DatabaseManager.getInstance().openDatabase();
	database.insert(...);
	// database.close(); 不应该直接关闭它
	DatabaseManager.getInstance().closeDatabase(); //正确的关闭方式

这个例子中，我们需要做的是在application里面调用initializeInstance进行一次实例化，然后随用随关。
### 安卓期中作业 NotePad功能扩展

----

实验准备：下载NotePad源码：https://github.com/llfjfz/NotePad

####  1、添加时间戳功能

#### 添加线性布局

在NotePad原应用中，笔记列表只显示了笔记的标题。要对它做时间扩展，可以把时间放在标题的下方。
首先，找到列表中item的布局：noteslist_item.xml。
在这个布局文件中，能看到一个TextView，这个便是笔记列表的标题item了。 

```
<TextView xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@android:id/text1"
    android:layout_width="match_parent"
    android:layout_height="?android:attr/listPreferredItemHeight"
    android:textAppearance="?android:attr/textAppearanceLarge"
    android:gravity="center_vertical"
    android:paddingLeft="5dip"
    android:singleLine="true"
/>
```

要在标题下方加时间显示，就要在标题的TextView下再加一个时间的TextView。但是由于原应用列表item只需要一个标题，所以不需要用上别的布局，而我要多加一个时间TextView，就要把标题TextView和时间TextView放入垂直的线性布局。 添加后代码如下：

```
<RelativeLayout android:layout_height="match_parent"
    android:layout_width="match_parent"
    xmlns:android="http://schemas.android.com/apk/res/android">
    <TextView xmlns:android="http://schemas.android.com/apk/res/android"
        android:id="@android:id/text1"
        android:layout_width="match_parent"
        android:layout_height="?android:attr/listPreferredItemHeight"
        android:textAppearance="?android:attr/textAppearanceLarge"
        android:gravity="center_vertical"
        android:paddingLeft="5dip"
        android:singleLine="true"
    />
    <TextView
        android:id="@+id/text2"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:paddingLeft="5dip"
        android:singleLine="true"
        android:gravity="center_vertical"
    />
</RelativeLayout>
```

####  分析如何实现时间戳

先查看程序如何定义数据库结构的，NotePadProvider.java中: 

```
@Override
    public void onCreate(SQLiteDatabase db) {
        db.execSQL("CREATE TABLE " + NotePad.Notes.TABLE_NAME + " ("
            + NotePad.Notes._ID + " INTEGER PRIMARY KEY,"
            + NotePad.Notes.COLUMN_NAME_TITLE + " TEXT,"
            + NotePad.Notes.COLUMN_NAME_NOTE + " TEXT,"
            + NotePad.Notes.COLUMN_NAME_CREATE_DATE + " INTEGER,"
            + NotePad.Notes.COLUMN_NAME_MODIFICATION_DATE + " INTEGER"
            + ");");
    }
```

 可以看出，NotePad数据库已经存在时间信息。
再到NoteList.java文件中查看，是如何将数据装填到列表中。
可以发现，当前Activity所用到的数据被定义在PROJECTION中 

```
private static final String[] PROJECTION = new String[] {
            NotePad.Notes._ID, // 0
            NotePad.Notes.COLUMN_NAME_TITLE, // 1
    };

```

 通过Cursor从数据库中读取出： 

```
Cursor cursor = managedQuery(
            getIntent().getData(),      // Use the default content URI for the provider.
            PROJECTION,                 // Return the note ID and title for each note.
            null,                       // No where clause, return all records.
            null,                       // No where clause, therefore no where column values.
            NotePad.Notes.DEFAULT_SORT_ORDER  // Use the default sort order.
        );
```

 通过SimpleCursorAdapter装填： 

```
String[] dataColumns = { NotePad.Notes.COLUMN_NAME_TITLE } ;
int[] viewIDs = { android.R.id.text1 };
SimpleCursorAdapter adapter
    = new SimpleCursorAdapter(
            this, // The Context for the ListView
            R.layout.noteslist_item, // Points to the XML for a list item
            cursor,   // The cursor to get items from
            dataColumns,
            viewIDs
    );
// Sets the ListView's adapter to be the cursor adapter that was just created.
setListAdapter(adapter);
```

#### 实现添加

时间戳 这并不是我们平常所见到的时间格式，需要对这些数据进行转化，使人能看的懂。
我选则的方法时把时间戳改为以时间格式存入，改动地方分别为NotePadProvider中的insert方法和NoteEditor中的updateNote方法。前者为创建笔记时产生的时间，后者为修改笔记时产生的时间。下面代码中的dateTime即为转化后的时间格式，将其用ContentValues的put方法存入数据库。 

 insert(): 

```
	//系统默认显示的时间使用毫秒数进行表示，很难阅读
		Long now = Long.valueOf(System.currentTimeMillis());
        Date date = new Date(now);
		//将时间格式化为常见的年月日表示形式
        SimpleDateFormat simpleDateFormat = new SimpleDateFormat("yy-MM-dd HH:mm:ss");
        simpleDateFormat.setTimeZone(TimeZone.getTimeZone("GMT+08:00"));
        String dateFormat = simpleDateFormat.format(date);
        if (values.containsKey(NotePad.Notes.COLUMN_NAME_CREATE_DATE) == false) {
            values.put(NotePad.Notes.COLUMN_NAME_CREATE_DATE, dateFormat);
        }
        if (values.containsKey(NotePad.Notes.COLUMN_NAME_MODIFICATION_DATE) == false) {
            values.put(NotePad.Notes.COLUMN_NAME_MODIFICATION_DATE, dateFormat);
        }
```

 updateNote(): 

```
  private final void updateNote(String text, String title) {
        ContentValues values = new ContentValues();
        long now = System.currentTimeMillis();
        Date date = new Date(now);
        SimpleDateFormat simpleDateFormat = new SimpleDateFormat("yy-MM-dd HH:mm:ss");
        simpleDateFormat.setTimeZone(TimeZone.getTimeZone("GMT+08:00"));
        String dateFormat = simpleDateFormat.format(date);
        values.put(NotePad.Notes.COLUMN_NAME_MODIFICATION_DATE, dateFormat);
        if (mState == STATE_INSERT) {

            if (title == null) {
                int length = text.length();
                title = text.substring(0, Math.min(30, length));
                if (length > 30) {
                    int lastSpace = title.lastIndexOf(' ');
                    if (lastSpace > 0) {
                        title = title.substring(0, lastSpace);
                    }
                }
            }
            values.put(NotePad.Notes.COLUMN_NAME_TITLE, title);
        } else if (title != null) {
            values.put(NotePad.Notes.COLUMN_NAME_TITLE, title);
        }
        values.put(NotePad.Notes.COLUMN_NAME_NOTE, text);
        getContentResolver().update(
                mUri,    
                values,
                null, 
                null    
            );
    }
```

 接下来我们找到创建Note时的源码： 

```
public class NotesList extends ListActivity {
    private static final String TAG = "NotesList";
    private static final String[] PROJECTION = new String[] {
            NotePad.Notes._ID, // 0
            NotePad.Notes.COLUMN_NAME_TITLE, // 1
    };
    private static final int COLUMN_INDEX_TITLE = 1;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setDefaultKeyMode(DEFAULT_KEYS_SHORTCUT);
        Intent intent = getIntent();
        if (intent.getData() == null) {
            intent.setData(NotePad.Notes.CONTENT_URI);
        }
        getListView().setOnCreateContextMenuListener(this);
        Cursor cursor = managedQuery(
            getIntent().getData(),
            PROJECTION,
            null,
            null,
            NotePad.Notes.DEFAULT_SORT_ORDER
        );
        String[] dataColumns = { NotePad.Notes.COLUMN_NAME_TITLE } ;
        int[] viewIDs = { android.R.id.text1 };
        SimpleCursorAdapter adapter
            = new SimpleCursorAdapter(
                      this,
                      R.layout.noteslist_item,
                      cursor,
                      dataColumns,
                      viewIDs
              );

```





#### 运行结果：

#### 2、添加查询功能

#### 添加图标

我们在顶部的菜单栏添加一个查询的图标，找到list_option_menu.xml，添加代码如下：

```
    <item
        android:id="@+id/menu_search"
        android:icon="@android:drawable/ic_search_category_default"
        android:showAsAction="always"
        android:title="search">
    </item>
```

#### 添加选择事件

找到函数onOptionsItemSelected，添加选择事件：

```
case R.id.menu_search:
                Intent intent = new Intent();
                intent.setClass(this, NoteSearch.class);
                this.startActivity(intent);
                return true;
```

#### 新建一个用于查找功能实现以及布局的Activity

新建Activity：NoteSearch以及它的布局文件note_search.xml 编辑布局文件，代码如下：

```
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">
    <SearchView
        android:id="@+id/search_view"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:iconifiedByDefault="false"
    </SearchView>
    <ListView
        android:id="@+id/list_view"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        >
    </ListView>
</LinearLayout>
```

在NoteSeach中查找功能的实现，代码如下:

```
public boolean onQueryTextChange(String newText) {    
    String selection1 = NotePad.Notes.COLUMN_NAME_TITLE+" like ? or "+NotePad.Notes.COLUMN_NAME_NOTE+" like ?";
    String[] selection2 = {"%"+string+"%","%"+string+"%"};
    Cursor cursor=sqLiteDatabase.query(
        NotePad.Notes.TABLE_NAME,
        PROJECTION,
        selection1, 
        selection2,
        null,
        null,
        NotePad.Notes.DEFAULT_SORT_ORDER
    );
    listview.setAdapter(adapter);
        return true;
}
```

显示以及获取代码的实现：

```
ListView listview;
SQLiteDatabase sqLiteDatabase;
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    requestWindowFeature(Window.FEATURE_NO_TITLE);   								
    super.setContentView(R.layout.note_search);
    Intent intent = getIntent();
    if (intent.getData() == null) {
        intent.setData(NotePad.Notes.CONTENT_URI);
    }
    listview= (ListView) findViewById(R.id.list_view);//获取listview
    sqLiteDatabase=new NotePadProvider.DatabaseHelper(this).getReadableDatabase();
    SearchView search= (SearchView) findViewById(R.id.search_view);//获取搜索视图
    search.setOnQueryTextListener(NoteSearch.this);  
}
```



#### 运行结果:



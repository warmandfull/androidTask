# 对开源笔记本应用进行功能拓展

## 一、 原应用
原应用已经包含基本增删改查功能
 
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200530225044448.png)
## 二、 拓展功能要求

 - NoteList中显示条目增加时间戳显示
 - 添加笔记查询功能（根据标题查询）

## 三、NoteList中显示条目增加时间戳显示的实现

 1. 先修改列表项的布局文件notelist_item.xml，添加一个TextView来显示时间戳，并用约束布局来调整组件的位置。

```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="70dp">

    <!-- 标题 -->
    <TextView
        android:id="@android:id/text1"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:paddingLeft="15dp"
        android:text="测试标题"
        android:textSize="20sp"
        app:layout_constraintBottom_toTopOf="@android:id/text2"
        app:layout_constraintTop_toTopOf="parent"
        app:layout_constraintVertical_bias="1.0"
        tools:layout_editor_absoluteX="0dp" />

    <!-- 时间戳 -->
    <TextView
        android:id="@android:id/text2"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_marginBottom="10dp"
        android:text="2020-01-01 12:00:01"
        android:textSize="15sp"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintHorizontal_bias="0.054"
        app:layout_constraintStart_toStartOf="parent" />
</androidx.constraintlayout.widget.ConstraintLayout>
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200530230855923.png)


2. 源码中显示数据库已经有了与时间戳相关的字段

```java
    @Override
       public void onCreate(SQLiteDatabase db) {
           db.execSQL("CREATE TABLE " + NotePad.Notes.TABLE_NAME + " ("
                   + NotePad.Notes._ID + " INTEGER PRIMARY KEY,"   //主键ID
                   + NotePad.Notes.COLUMN_NAME_TITLE + " TEXT,"  //标题
                   + NotePad.Notes.COLUMN_NAME_NOTE + " TEXT,"   //笔记内容
                   + NotePad.Notes.COLUMN_NAME_CREATE_DATE + " INTEGER," //创建时间
                   + NotePad.Notes.COLUMN_NAME_MODIFICATION_DATE + " INTEGER" //最后修改时间
                   + ");");
       }
```

3. 分析一下例表投影的过程

（1）NotesList这个类是负责显示列表的Activity，它继承了ListActivity。

```java
public class NotesList extends ListActivity {

```
（2）在这个Activity被创建时会调用onCreate(Bundle savedInstanceState) 方法

```java
  /**
     * onCreate is called when Android starts this Activity from scratch.
     */
    @Override
    protected void onCreate(Bundle savedInstanceState) {
```
（3）在onCreate方法里包含了设置Adapter的相关代码

```java
// Sets the ListView's adapter to be the cursor adapter that was just created.
        setListAdapter(adapter);
```
（4）上一步的参数中的adapter是SimpleCursorAdapter类的实例对象。

```java
    // Creates the backing adapter for the ListView.
        SimpleCursorAdapter adapter
            = new SimpleCursorAdapter(
                      this,                             // The Context for the ListView
                      R.layout.noteslist_item,          // Points to the XML for a list item
                      cursor,                           // The cursor to get items from
                      dataColumns,
                      viewIDs
              );
```

（5）SimpleCursorAdapter类的构造参数dataColumns是数据列名，它来源于

```java
   // The names of the cursor columns to display in the view, initialized to the title column
   String[] dataColumns = { NotePad.Notes.COLUMN_NAME_TITLE } ;
```
我们给他新增一个值，对应修改时间

```java
String[] dataColumns = { NotePad.Notes.COLUMN_NAME_TITLE, NotePad.Notes.COLUMN_NAME_MODIFICATION_DATE } ;
```
（6）SimpleCursorAdapter类的构造参数viewIDs是notelist_item.xml中对应的组件ID，要让它于dataColumns 的预期位置相对应，所以我们修改这个变量为

```java
// The view IDs that will display the cursor columns, initialized to the TextView in
// noteslist_item.xml
int[] viewIDs = { android.R.id.text1 ,android.R.id.text2 };
```
（7）SimpleCursorAdapter类的构造参数cursor是查询数据库得到的对象
```java
 Cursor cursor = managedQuery(
            getIntent().getData(),            // Use the default content URI for the provider.
            PROJECTION,                       // Return the note ID and title for each note.
            null,                             // No where clause, return all records.
            null,                             // No where clause, therefore no where column values.
            NotePad.Notes.DEFAULT_SORT_ORDER  // Use the default sort order.
        );
```
PROJECTION数组里是要投影的列名

```java
/**
     * The columns needed by the cursor adapter
     */
    private static final String[] PROJECTION = new String[] {
            NotePad.Notes._ID, // id
            NotePad.Notes.COLUMN_NAME_TITLE, // 标题
    };
```
其中只有标题和ID，所以增加一个修改时间

```java
/**
     * The columns needed by the cursor adapter
     */
    private static final String[] PROJECTION = new String[] {
            NotePad.Notes._ID, // id
            NotePad.Notes.COLUMN_NAME_TITLE, // 标题
            NotePad.Notes.COLUMN_NAME_MODIFICATION_DATE //修改时间
    };
```
（8）做以上修改步骤，就能把时间投影出来。但投影出来的时间只是毫秒数，需要进一步的处理。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200530235816405.png)

（9）修改adapter的视图绑定器，创建新的匿名内部类SimpleCursorAdapter.ViewBinder，实现setViewValue方法。

```java
adapter.setViewBinder(new SimpleCursorAdapter.ViewBinder() {
            @Override
            public boolean setViewValue(View view, Cursor cursor, int columnIndex) {
                //找到修改时间对应的列
                if(cursor.getColumnIndex(NotePad.Notes.COLUMN_NAME_MODIFICATION_DATE) == columnIndex)
                {
                    //根据columnIndex获取时间来创建Date对象
                    Date date = new Date(cursor.getLong(columnIndex));
                    //新建时间格式
                    SimpleDateFormat simpleDateFormat = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
                    //格式化时间
                    String datetime = simpleDateFormat.format(date);
                    //数据绑定
                    TextView textView = (TextView)view;
                    textView.setText(datetime);
                    return true;
                }
                return false;
            }
        });
```
（10）最终效果

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200531032857626.png)

## 四、添加笔记查询功能（根据标题查询）的实现
1. 先在选项菜单上添加一个搜索按钮，添加一个item，设置他的actionViewClass为android.widget.SearchView

```xml
<?xml version="1.0" encoding="utf-8"?>

<menu xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto" >
    <!--  search  -->
    <item android:id="@+id/menu_search"
        android:title="search"
        android:actionViewClass="android.widget.SearchView"
        android:showAsAction="ifRoom" />
    <!--  This is our one standard application action (creating a new note). -->
    <item android:id="@+id/menu_add"
          android:icon="@drawable/ic_menu_compose"
          android:title="@string/menu_add"
          android:alphabeticShortcut='a'
          android:showAsAction="ifRoom" />
    <!--  If there is currently data in the clipboard, this adds a PASTE menu item to the menu
          so that the user can paste in the data.. -->
    <item android:id="@+id/menu_paste"
          android:icon="@drawable/ic_menu_compose"
          android:title="@string/menu_paste"
          android:alphabeticShortcut='p' />
</menu>
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200531030241149.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200531030402600.png)

2.  在契约类NotePad中添加一个新的静态变量，该变量是按标题模糊查询的where条件子句

```java
public static final String SELECT_BY_LIKE_TITLE = COLUMN_NAME_TITLE + " like ?";
```


3. 在NoteList类中的onCreateOptionsMenu方法中对searchView添加事件监听。主要是实现当文字改变时的方法，获取数据的过程和显示所有条目的过程差不多，只是where条件子句和参数不同。

```java
  SearchView searchView = (SearchView) menu.findItem(R.id.menu_search).getActionView();
        //设置搜索框文字变化事件监听
        searchView.setOnQueryTextListener(new SearchView.OnQueryTextListener() {
            @Override
            public boolean onQueryTextSubmit(String query) {
                return false;
            }
            //当搜索框内文字变化时调用
            @Override
            public boolean onQueryTextChange(String newText) {
                //设置where语句参数
                String[] selectionArgs = {"%" + newText + "%"};

                Cursor cursor = managedQuery(
                        getIntent().getData(),            // Use the default content URI for the provider.
                        PROJECTION,                       // Return the note ID and title for each note.
                        NotePad.Notes.SELECT_BY_LIKE_TITLE,
                        selectionArgs,
                        NotePad.Notes.DEFAULT_SORT_ORDER  // Use the default sort order.
                );
                // The names of the cursor columns to display in the view, initialized to the title column
                String[] dataColumns = { NotePad.Notes.COLUMN_NAME_TITLE, NotePad.Notes.COLUMN_NAME_MODIFICATION_DATE} ;

                // The view IDs that will display the cursor columns, initialized to the TextView in
                // noteslist_item.xml
                int[] viewIDs = { android.R.id.text1, android.R.id.text2 };

                // Creates the backing adapter for the ListView.
                SimpleCursorAdapter adapter
                        = new SimpleCursorAdapter(
                        getApplicationContext(),          // The Context for the ListView
                        R.layout.noteslist_item,          // Points to the XML for a list item
                        cursor,                           // The cursor to get items from
                        dataColumns,
                        viewIDs
                );
                //格式化时间数据
                changeDateFormat(adapter);
                setListAdapter(adapter);
                return true;
            }
        });
```

4. 运行效果

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200531032611378.png)

---
layout: post
title:  "#土豆记事#教你开发Android App之 —— 真的开始写App了"
date:   2015-07-03 18:30:20 +0800
categories: android
---

# 基础界面
我们要开发的App界面如下

![clipboard.png](https://segmentfault.com/img/bVmzuq)

> 1. 有一个title
> 2. 一个列表
> 3. 右下角一个按钮

1. title 可以用系统自带的`ActionBar`实现（Lollipop以上为Toolbar）。
2. 下面的列表可以用`ListView`或者`android-support-compact-v7`提供的新的`RecyclerView`。展示一个列表。
3. 按钮可以使用普通的Button，我这里为了符合`Material Design`规范，使用了`FloatingActionButton`，没什么不同，只是展示出来的样式不一样而已。

整个界面就这么简单。

我们可以看到`Toolbar`以下的部分是列表和按钮，它们的排列不是属于线性排列，所以我们就想到用`RelativeLayout`布局 —— 列表占满空间，而按钮存在容器的右下角。
我们的布局文件就如下：
```
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".MainActivity">

    <org.lifefortheorc.tudounotepad.widget.EmptyRecyclerView
        android:id="@+id/recycler_note"
        android:layout_width="match_parent"
        android:layout_height="match_parent" />

    <TextView
        android:id="@+id/tv_empty"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="@string/empty"
        android:gravity="center"
        android:layout_centerInParent="true" />

    <android.support.design.widget.FloatingActionButton
        android:id="@+id/fab_add"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_alignParentEnd="true"
        android:layout_alignParentBottom="true"
        android:layout_marginEnd="@dimen/activity_horizontal_margin"
        android:layout_marginBottom="@dimen/activity_vertical_margin"
        android:src="@drawable/ic_add" />

</RelativeLayout>
```
这其中的`tv_empty`是我们用来做一个列表空内容的提醒的，触发的结果如下
**当列表不为空时隐藏，当列表为空时就显示**
![clipboard.png](https://segmentfault.com/img/bVmzuO)

# 关于列表
不管是`ListView`还是`RecylerView`，如果你熟悉`iOS`开发的话，会理解`item`重用的机制——`Android`为了省资源，只会创建一屏幕的`item`，如果你的列表项目数目大于一个屏幕，当你滚下列表时，系统会把顶上的`item`回收，在回调函数里提供给你重新使用，然后重新在底部显示出来，这样的好处是你创建的`item`不论你实际数据的大小，它至多只创建一个屏幕多的数量，能节省资源。

列表、子视图和数据之间的交互使用了`适配器模式`，在`Adapter`中把你的数据渲染到`item view`上，然后应用给父视图。
关于`ListView`的介绍，请看传送门：http://developer.android.com/guide/topics/ui/layout/listview.html

如果你看懂了`ListView`，那么`RecylerView`只是对其的一种优化，使用了更多的自定义的属性和更方便自由的布局管理系统。

我这里使用了`RecylerView`，我们来看下适配器的源码
```
public class NoteAdapter extends RecyclerView.Adapter<NoteAdapter.ViewHolder> {

    private static final SimpleDateFormat dateFormat = new SimpleDateFormat("yyyy年MM月dd日 HH:mm:ss");

    private Context mContext;
    private List<NoteModel> noteList;

    public NoteAdapter(Context ctx) {
        this.noteList = new ArrayList<>();
        this.mContext = ctx;
    }

    public void setNoteList(List<NoteModel> noteList) {
        //设置数据
        this.noteList.clear();
        this.noteList.addAll(noteList);
    }

    @Override
    public ViewHolder onCreateViewHolder(ViewGroup viewGroup, int i) {
        LayoutInflater layoutInflater = LayoutInflater.from(mContext);
        View view = layoutInflater.inflate(R.layout.item_note, viewGroup, false);
        //返回默认界面布局
        return new ViewHolder(view);
    }

    @Override
    public void onBindViewHolder(ViewHolder viewHolder, int position) {
        NoteModel note = noteList.get(position);

        String content = note.getContent();
        long time = note.getTime();

        //根据数据渲染界面
        viewHolder.mTextViewContent.setText(content);
        viewHolder.mTextViewTitle.setText(dateFormat.format(time));
    }

    public void remove(int position) {
        NoteModel note = this.noteList.get(position);
        note.delete();
        this.noteList.remove(position);
    }

    @Override
    public int getItemCount() {
        return noteList.size();
    }

    public class ViewHolder extends RecyclerView.ViewHolder implements View.OnClickListener {
        @Bind(R.id.tv_title)
        public TextView mTextViewTitle;
        @Bind(R.id.tv_content)
        public TextView mTextViewContent;

        public ViewHolder(View itemView) {
            super(itemView);
            ButterKnife.bind(this, itemView);
            itemView.setOnClickListener(this);
        }

        @Override
        public void onClick(View v) {
            int position = this.getAdapterPosition();
            NoteModel note = noteList.get(position);


            long id = note.getId();
            Intent intent = new Intent(mContext, EditActivity.class);
            intent.putExtra("id", id);
            ActivityOptions options =  ActivityOptions.makeSceneTransitionAnimation((Activity) mContext,
                    v, "content");
            mContext.startActivity(intent,
                    options.toBundle());
        }
    }

}
```

靠这一套`Adapter`，`RecyclerView`就能一项一项渲染出我们要的列表了。

# 关于数据
渲染界面少不了数据，我们创建数据的入口在右下角，点"+"号就可以跳转到新建的页面
代码如下：
```
Intent intent = new Intent(this, EditActivity.class);
startActivity(intent);
```
两行即可搞定。

我们跳转到新的界面

![clipboard.png](https://segmentfault.com/img/bVmzvt)
在这里用户可以输入想要的内容，点右上角的保存即可保存。保存到内存很简单，我们如何把数据持久化呢？
Android系统提供了`SQLite`来给我们持久化数据。`SQLite`是一种轻量级的文件关系型数据库，可以满足我们的需求，我们使用`SQLite`存储用户输入的数据，然后保存到本地，就是这样。
> 关于Android中，`SQLite`的部分，可以看这个传送门：http://developer.android.com/training/basics/data-storage/databases.html

在这里，我们为了方便直接使用`ActiveAndroid`的ORM方案(http://www.activeandroid.com/)，可以
把`Model`直接持久化到数据库，然后从数据库中读取数据。
我们的模型如下：
```
@Column(name = "content")
private String content;

@Column(name = "time")
private long time;

public String getContent() {
    return content;
}

public void setContent(String content) {
    this.content = content;
}

public long getTime() {
    return time;
}

public void setTime(long time) {
    this.time = time;
}

public static List<NoteModel> queryNoteList() {
    return new Select().from(NoteModel.class).execute();
}

public static NoteModel queryById(long id) {
    return new Select().from(NoteModel.class)
                       .where("Id = ?", id).executeSingle();
}
```
我们记录用户存入的内容，存入的事件，然后定义两个方法——列出所有存着的列表和根据id查询其中一个对象，我们使用这两个方法就能获取到列表和详情两个数据。

列表数据提供给首页，详情数据提供给编辑页面。这样就满足了一个记事本App的基础需求——增加与修改。

列表页：

![clipboard.png](https://segmentfault.com/img/bVmzwd)

详情页：

![clipboard.png](https://segmentfault.com/img/bVmzwe)

如果对于写App入门还有什么问题，欢迎留言以及在[Github](https://github.com/geminiwen)上follow我

> 本文提到的项目源码地址：https://github.com/geminiwen/tudounotepad
> 欢迎留言[Github](https://github.com/geminiwen)或者[@geminiwen](http://weibo.com/coffeesherk/home?wvr=5)
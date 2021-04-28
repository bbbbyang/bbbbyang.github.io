---
title: Starting from ListView
date: 2018-06-23 14:50:03
tags: View
categories: Android
---

In Android development, _ListView_ and _RecyclerView_ are very important widgets. When I was first time to use ListView. The adapter and getView function really confused me. Functionality of adapter is to display data on the item view of the ListView. You can easily implement it follow the step in [here][ListView]. The following picture shows the simple principle of the adapter.

![][Adapter]

How the adapter works?

#### 1. getListItemView
Let's put ListView aside. We can use the Linear layout to implement a list structure to display data.

```xml
// Main activity layout
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/linear_list"
    android:layout_width="match_parent"
    android:layout_height="match_parent">
</LinearLayout>

// Item layout: list_item
<LinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="wrap_content">

    <TextView
        android:id="@+id/list_item_text"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"/>

</LinearLayout>

```

And we have a List of Items
```java
private List<Item> listItems;
```
When we setup UI, we need to find the item layout and put a item into the TextView then add the TextView into the main layout. Repeat this process through all items.

```java
protected void onCreate(Bundle savedInstanceState) { 
    // set up....

    LinearLayout linearLayout = (LinearLayout) findViewById(R.id.linear_list);
    for (Item item : listItems) {
        View view = getListItemView(item);
        linearLayout.addView(view);
    }
}

private View getListItemView(Item item) {
    View view = getLayoutInflater().inflate(R.layout.list_item, null);
    ((TextView) view.findViewById(R.id.list_item_text)).setText(item.text);
    return view;
}
```
Now you can see that MainActivity is mainly a list structure. The function _getListItemView_ can be considered as a simple adapter that converts the item data to a view.

#### 2. ItemConverter
So let's drag this function to a separate class ItemConverter.
```java
public class ItemConverter {

    private Context context;
    private List<Item> data;

    public ItemConverter(Context context, List<Item> data) {
        this.context = context;
        this.data = data;
    }

    public View getView(int position) {
        View view = LayoutInflater.from(context).inflate(R.layout.list_item, null);
        Item item = data.get(position);

        ((TextView) view.findViewById(R.id.list_item_text)).setText(item.text);
        return view;
    }
}
```
The mainly functionality of _ItemConverter_ is _getView_ function. It is the same function with _getListItemView_. Only difference is the augments changed from the item of the list to the position of the item in the list. And the _oncreate_ function in MainActivity will be like this.
```java
    LinearLayout linearLayout = (LinearLayout) findViewById(R.id.linear_list);
    ItemConverter converter = new ItemConverter(this, listItems);

    for (int i = 0; i < listItems.size(); ++i) {
        View view = converter.getView(i);
        linearLayout.addView(view);
    }
```
So we use a converter to do an adapter job in this case. And we can see very clearly how the getView function works and how the adapter converts data to a view.

#### 3. OnBindViewHolder and CreateViewHolder
You can check the document for how to use ListView. Here not going to talk much about the convertView. There is one difference between ListView and RecyclerView is that RecyclerView is to force use to use ViewHolder. Let's do it in ListView. For ListView the layout and basic setup you can check [here][ListView]. In this document, _MyAdapter_ extends from _BaseAdapter_
```java
public View getView(int position, View convertView, ViewGroup parent) {
    ViewHolder vh;
    if (convertView == null) {
        convertView = LayoutInflater.from(context).inflate(R.layout.main_list_item, parent, false);

        vh = new ViewHolder();
        vh.itemText = (TextView) convertView.findViewById(R.id.main_list_item_text);
        convertView.setTag(vh);
    } else {
        vh = (ViewHolder) convertView.getTag();
    }

    Item item = data.get(position);
    vh.itemText.setText(item.text);
    return convertView;
}

private static class ViewHolder {
    TextView itemText;
}
```
Here you can see a simply way to cache a viewHolder. (time consuming of findViewById). In order to force user to use this ViewHolder, we need to drag this ViewHolder to a new abstract class called _ViewHolderAdapter_ which extends _BaseAdapter_. And our _MyAdapter_ extends _ViewHolderAdapter_.

In _ViewHolderAdapter_ we override _getView_ function here. Because in _getView_, we only care how the view created and how the data binded with the view. So we set two abstract functions here to create view and bind view holder, which will be implemented in _MyAdapter_.

```java
public abstract class ViewHolderAdapter extends BaseAdapter {

    @Override
    public View getView(int position, View convertView, ViewGroup parent) {
        ViewHolder vh;
        if (convertView == null) {
            vh = onCreateViewHolder(parent, position);
            convertView = vh.view;
            vh.view.setTag(vh);
        } else {
            vh = (ViewHolder) convertView.getTag();
        }

        onBindViewHolder(vh, position);
        return convertView;
    }

    protected abstract ViewHolder onCreateViewHolder(ViewGroup parent, int position);

    protected abstract void onBindViewHolder(ViewHolder viewHolder, int position);

    public static abstract class ViewHolder {
        protected View view;
        public ViewHolder(View view) {
            this.view = view;
        }
    }
}
```

```java
private class MyAdapter extends ViewHolderAdapter {
    @Override
    protected ViewHolderAdapter.ViewHolder onCreateViewHolder(ViewGroup parent, int position) {
        View view = LayoutInflater.from(context).inflate(R.layout.list_item, parent, false);
        return new ItemViewHolder(view);
    }

    @Override
    protected void onBindViewHolder(ViewHolderAdapter.ViewHolder viewHolder, int position) {
        // getItem should be override inside MyAdapter
        Item item = (Item) getItem(position);
        // this viewHolder is upcast from ItemViewHolder created in onCreateViewHolder
        // send to its father class in ViewHolderAdapter
        // then downcast to ItemViewHolder again to get itemText view
        ((ItemViewHolder) viewHolder).itemText.setText(item.text);
    }

    private static class ItemViewHolder extends ViewHolderAdapter.ViewHolder {
        TextView itemText;

        public ItemViewHolder(View view) {
            super(view);
            itemText = (TextView) view.findViewById(R.id.list_item_text);
        }
    }
}
```
Now you can check inside _MyAdapter_, that is how we use onCreateViewHolder and onBindViewHolder. For how to use _RecyclerView_, please check [here][RecyclerView]. It is basically same process with our adpater.


[ListView]: https://developer.android.com/reference/android/widget/ListView
[Adapter]: https://www.careerride.com/Image/MCA/Adapter-view.png
[RecyclerView]: https://developer.android.com/guide/topics/ui/layout/recyclerview

---
title: Interface Callback and Observer Pattern
date: 2018-10-16 16:48:26
tags: Interface
categories: Java
---

Callback is a very important feature in Java. For button click listener, it uses interface callback. I used to use interface callback often, but not very clearly understand it.

#### Interface and Callback

It is very easy to define an interface. Here we define a _EatHelperInterface_ interface and it has a _eat_ function.
```java
interface EatHelperInterface {
    void eat();
}
```
If this person is a Chinese, we just implement this interface.
```java
public class Chinese implements EatHelperInterface{
    @Override
    public void eat() {
        System.out.println("I need chopsticks to eat");
    }
}
```
This is how we implement an interface. Also we can define many classes for different people. However, we don't want so many class defined. It is better to define a _People_ class to keep this interface, so that we can use callback to help different people to eat.
```java
public class People {
    interface EatHelperInterface eatHelperInterface;
    public void setEatHelperInterface(EatHelperInterface eatHelperInterface){
        this.eatHelperInterface = eatHelperInterface;
    }

    public void eatService(){
        eatHelperInterface.eat();
    }
}
```
_setEatHelperInterface_ is to register the interface. _People.eatService()_ is to execute the _eat_ function of the interface. Our test can be like this.
```java
public class Test {
    public static void main(String[] args) {
        People chinese = new people();
        chinese.setEatHelperInterface(new Chinese());
        chinese.eatService();
    }
}
```
In many cases, we would like to use an anonymous inner class to define an interface.
```java
public class Test {
    public static void main(String[] args) {
        People people = new people();
        people.setEatHelperInterface(new EatHelperInterface() {
            @Override
            public void eat() {
                System.out.println("I need **** to eat");
            }
        });
        people.eatService();
    }
}
```

Callback can separate the class which use this function and the one implement the function. Because the they do not care about each other. For any _People_, we just use _people.eatService()_ to perform _eat_ action. Different people pass different implementation into the interface.

#### Callback in Android

In Android development, for button click function, we have an interface _OnClickListener_ and _Button_(View) contains this interface.

```java
public interface OnClickListener {   
    void onClick(View v);  
}
```

```java
public class View implements Drawable.Callback, KeyEvent.Callback, AccessibilityEventSource {  
    protected OnClickListener mOnClickListener;  

    public void setOnClickListener(OnClickListener l) {  
        if (!isClickable()) {  
            setClickable(true);  
        }  
        mOnClickListener = l;  
    }  
    
    public boolean performClick() {  
        if (mOnClickListener != null) {
            mOnClickListener.onClick(this);  
            return true;  
        }  
        return false;  
    }  
}
```
This button that extends the view class is very similar with the People class, which contains the interface and execute its function.

For _MainActivity_ we should ask for help how to perform this click function. We can implement the interface in _MainActivity_ and pass _this_ into _OnClickListener_, or we can use anonymous inner class.

```java
public class MainActivity extends Activity{  
    private Button button; 
    @Override 
    public void onCreate(Bundle savedInstanceState) {  
        super.onCreate(savedInstanceState);  
        setContentView(R.layout.activity_main);  
        button = (Button)findViewById(R.id.button1); 
        button.setOnClickListener(new OnClickListener() {
            @Override 
            public void onClick(View v) {
            }
        });
    }
}
```

- _Button_ -> _People_
- _setOnClickListener_ -> _setEatHeplerInterface_
- _onClickListener_ -> _EatHelperInterface_
- _onClick_ -> _eat_

The difference of these two cases is that _people.eatService()_ called in _Test_, but for button, only case to trigger the _performClick_ is when we click the button (Not showing in code).

#### Observer Pattern

This is a simple Observer Pattern example. Observer A is very sensitive to some actions of observable B. A needs to do actions at the moment changes in B. Also look at the button example. When trigger _button.performClick_, _OnClickListener_ need to react immediately.

Trigger: button Click -> view.performClick -> onClick

Hierarchy: (Button)View.setOnClickListener -> OnClickListener -> onClick

For Observer Patthern, there are _FOUR_ component: Observer, subscribe, Observable, event. Observer subscribe Observable. when state of Observable changed, observer will know that(subscribe) and trigger its event based on this change.

![][1]

- _Button_ -> observable
- _setOnClickListener_ -> subscribe
- _onClickListener_ -> Observer
- _onClick_ -> event

![][2]

For observer pattern, it is an very useful pattern and we have many needs of it.

In Android development, we can easily to achieve infinite loading feature by observer pattern. For a recyclerview, every time it will show 10 items with data and one more item showing loading layout. In adapter _onCreateViewHolder_ and _onBindViewHolder_ we should implement this way.

```java
@Override
public RecyclerView.ViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
    if(viewType == VIEW_TYPE_SHOT) {
        View view = LayoutInflater.from(parent.getContext()).inflate(R.layout.list_item_shot, parent, false);
        return new ShotViewHolder(view);
    } else {
        View view = LayoutInflater.from(parent.getContext()).inflate(R.layout.list_item_loading, parent, false);
        return new RecyclerView.ViewHolder(view) {};
    }
}

@Override
public void onBindViewHolder(final RecyclerView.ViewHolder holder, int position) {
    int viewType = getItemViewType(position);
    if(viewType == VIEW_TYPE_LOADING) {
        loadMoreListener.onLoadMore();
    } else {
        final Shot shot = data.get(position);
        // ....
    }
}
```
So everytime when we scrolling to the last item which is VIEW_TYPE_LOADING, it will trigger _onLoadMore_ function. It is like everytime we click button, _performClick_ will triiger _onClick_ function. When we setup adapter, we should implement this loadMoreListenner.onLoadmore function.

```java
shotListAdapter = new ShotListAdapter(new ArrayList<Shot>(), new ShotListAdapter.LoadMoreListener() {
    @Override
    public void onLoadMore() {
        int page = shotListAdapter.getDataCount() / COUNT_PRE_PAGE + 1;
        AsyncTaskCompat.executeParallel(new LoadShotTask(page));
    }
});
```
This _LoadShotTask_ extends _AsyncTask_ will use RESTful API to ask for next page of shots. This is a simple use of Interface Callback. The entire source code you can check from my [Dribbble app][3].

Another place to use it is that when we want to refresh the content of a fragment, while the refresh button is not inside the fragment. We need to fresh the fragment from its parent activity. We should implement the listener by the observer pattern. Here is the [simple example][4].

Another widely used of observer pattern is the RxJava.

[1]:http://ww3.sinaimg.cn/mw1024/52eb2279jw1f2rx4446ldj20ga03p74h.jpg
[2]:http://ww4.sinaimg.cn/mw1024/52eb2279jw1f2rx42h1wgj20fz03rglt.jpg
[3]:https://github.com/bbbbyang/DribbbleApp
[4]:https://stackoverflow.com/questions/26606527/android-refresh-a-fragment-list-from-its-parent-activity

# realm-recyclerview-lite

realm-recyclerview-lite is an implementation of a RecyclerView which supports Realm data.  
It is intended to be a slim library which fulfills the functionality requirements I have for my open source Android app, [Minerva][Minerva]; no more, no less. I decided to release it as a library in case others had similar use cases to mine. Take a look at the table of contents for a better idea of what features are offered.

If you just want to know what's new, [the changelog is here][CHANGELOG].

Please be sure to take a moment to look at the [Origin][Origin] section. You'll find a link to Thorben Primke's [realm-recyclerview][RRV] library, which this library is a heavily reworked derivative of.

#### Table of Contents
* [Installation](#installation)  
* [Basic Usage](#usage)
* [Drag and Drop](#drag-and-drop)  
    * [Long Click as the Drag Trigger](#long-click-drag-trigger)  
* [Multi-Select](#multi-select)  
* [Fast Scrolling](#fast-scrolling)  
    * [Handle State Notifications](#handle-state-notifications)  
    * [Fast Scroller Customization](#fast-scroller-customization)  

<a name="installation"/>
## Installation
Including realm-recyclerview-lite in your app is pretty simple, just make sure that you have the following in your root `build.gradle` file:

```groovy
buildscript {
    repositories {
        jcenter()
    }
}
```

And then add this to your app's `build.gradle` file:
```groovy
dependencies {
    compile ('com.bkromhout:realm-recyclerview-lite:{latest version}@aar') {
        transitive = true
    }
}
```
Please note that at this time, realm-recyclerview-lite has been tested and is verified to work with **Realm 0.89.1**. Don't be afraid to try a newer version of Realm, just be sure to open an issue if you run into problems.

**realm-recyclerview-lite is compatible with Android API Levels >= 11.**

<a name="usage"/>
## Basic Usage
Adding a `RealmRecyclerView` to your layout is simple:
```xml
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
                android:layout_width="match_parent"
                android:layout_height="match_parent">

    <com.bkromhout.rrvl.RealmRecyclerView
            android:id="@+id/recycler"
            android:layout_width="match_parent"
            android:layout_height="match_parent"/>
</RelativeLayout>
```

The other important thing is to make sure that your adapter extends `RealmBasedRecyclerViewAdapter`. In keeping with the provided sample application, here is a (very slimmed down) version of an adapter:
```java
public class ItemAdapter extends RealmBasedRecyclerViewAdapter<Item, ItemAdapter.ItemVH> {

    public ItemAdapter(Context context, RealmResults<Item> realmResults) {
        super(context, realmResults, true, true, null);
        setHasStableIds(true);
    }

    @Override
    public long getItemId(int position) {
        return realmResults.get(position).uniqueId;
    }

    @Override
    public ItemVH onCreateViewHolder(ViewGroup parent, int viewType) {
        return new ItemVH(inflater.inflate(R.layout.item_card, parent, false));
    }

    @Override
    public void onBindViewHolder(final ItemVH holder, int position) {
        Item item = realmResults.get(position);
        holder.name.setText(item.name);
    }

    static class ItemVH extends RecyclerView.ViewHolder {
        @Bind(R.id.content)
        RelativeLayout content;
        @Bind(R.id.drag_handle)
        ImageView dragHandle;
        @Bind(R.id.name)
        TextView name;

        public ItemVH(View itemView) {
            super(itemView);
            ButterKnife.bind(this, itemView);
        }
    }
}
```

If you look at the actual [`ItemAdapter` class][ItemAdapter Class], you'll notice that there are many other things present in it. We'll get to those as we discuss the features, this is just a bare-bones implementation.

To set your adapter to a `RealmRecyclerView`, you simply call its `setAdapter` method.

A couple more points of note:
* `RealmRecyclerView` supports **`LinearLayoutManager` only**
* `RealmRecyclerView` is *not* actually a `RecyclerView` subclass, it's a `RelativeLayout`. If you need access to the real `RecyclerView` or `LinearLayoutManager` instances for some reason, you can use the `getRecyclerView` and `getLayoutManager` methods
* When you're done using an adapter (such as when an Activity or Fragment is being destroyed), be sure to call its `close` method to prevent any possible Realm instance leaks

<a name="drag-and-drop"/>
## Drag and Drop
Drag and drop can be a tricky feature to implement in the first place since your data model usually must have some field which keeps track of a position. Combine this with Realm's auto-updating nature, and you can quickly get lost in a sea of troubles. Luckily, I've done most of the work for you 😉.

Keep in mind as you read through these steps that my preferred implementation choices may not line up exactly with yours; I've tried to keep this in mind to allow you maximum flexibility.

First, we need to enable drag and drop functionality. This can be done in two ways:
* In our layout, with the `dragAndDrop` attribute:  
```xml
<com.bkromhout.rrvl.RealmRecyclerView
        android:id="@+id/recycler"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        app:dragAndDrop="true"/>
```
* Or programmatically, with the `RealmRecyclerView.setDragAndDrop` method:  
```java
recyclerView.setDragAndDrop(true);
```

Next, some work needs to be done in our adapter. For drag and drop to work, we need to override the `onMove` method. We also need to add a bit more to our overridden `onBindViewHolder` method so that our items' drag handle views actually initiate drags when touched.  
These are the full methods from the sample app's [`ItemAdapter` class][ItemAdapter Class]:
```java
@Override
public void onBindViewHolder(final ItemVH holder, int position) {
    Item item = realmResults.get(position);
    holder.name.setText(item.name);
    // We set the unique ID as the tag on a view so that we will be able to get it
    // in the onMove() method.
    holder.content.setTag(item.uniqueId);
    // Grabbing the drag handle should trigger a drag.
    holder.dragHandle.setOnTouchListener(new View.OnTouchListener() {
        @Override
        public boolean onTouch(View v, MotionEvent event) {
            if (MotionEventCompat.getActionMasked(event) == MotionEvent.ACTION_DOWN)
                startDragging(holder);
            return false;
        }
    });
}

@Override
public boolean onMove(RecyclerView.ViewHolder dragging, RecyclerView.ViewHolder target) {
    // Get positions of items in adapter.
    int draggingPos = dragging.getAdapterPosition();
    int targetPos = target.getAdapterPosition();

    // Get the unique IDs of the items from the tag that we set in onBindViewHolder().
    long draggingId = (long) ((ItemVH) dragging).content.getTag();
    long targetId = (long) ((ItemVH) target).content.getTag();

    // Move the item up or down. The methods in ItemDragHelper will calculate and
    // assign a new position value for the item whose uniqueId == draggingId.
    if (draggingPos > targetPos) ItemDragHelper.moveItemToBefore(draggingId, targetId);
    else ItemDragHelper.moveItemToAfter(draggingId, targetId);

    return true;
}
```
Note how we've ensured that the `onMove` method will have access to the value in each `Item`'s `uniqueId` field by storing that value as the tag of the `content` view in the view holder.

Please take a moment to look at some of the sample application's classes if you haven't done so yet. Especially relevant here are the [`Item` model class][Item Class] and the [`ItemDragHelper` class][ItemDragHelper Class].

It's vitally important that you understand at least the *concept* of how the methods in `ItemDragHelper` work because you will need to implement something similar. If you're lucky then you may be able to adapt mine for your use. I won't detail it here, but if you want to read a bit more about my ordering scheme, [you can read this][Ordering Notes].

You'll notice that we **must** handle two cases in the `onMove` method:
1. An item has been moved up
2. An item has been moved down

You should also notice that nowhere in this code, be it the `onMove` method above or the methods in `ItemDragHelper`, do we call *any* of the `notify*Changed` methods. This is intended, because `RealmBasedRecyclerViewAdapter` handles making the correct calls for you when it detects the changes you've made to your data (it relies on a `RealmChangeListener` to get these notifications, and if you wish to see how it decides which of the `notify*Changed` methods to call, take a look at the [`RealmBasedRecyclerViewAdapter` class][RealmBasedRecyclerViewAdapter Class]).

<a name="long-click-drag-trigger"/>
### Long Click as the Drag Trigger
In the example above I showed you how you could set up your `onBindViewHolder` method so that grabbing a specific view on your item (the drag handle) would initiate a drag.

While that's great, we also have the option automatically initiating a drag when an item is long clicked; and for some use cases that's really all we need.  
To enable this functionality, we can either:
* Add the `longClickTriggersDrag` attribute to our layout:  
```xml
<com.bkromhout.rrvl.RealmRecyclerView
        android:id="@+id/recycler"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        app:dragAndDrop="true"
        app:longClickTriggersDrag="true"/>
```
* Or call the `setLongClickTriggersDrag` method on our `RealmRecyclerView`:  
```java
recyclerView.setLongClickTriggersDrag(true);
```

Once enabled, long clicking an item will automatically initiate a drag without you needing to call `startDragging`. You can still call it yourself in response to some other interaction if you'd like though.

Note that you *do* still have to implement the `onMove` method.

<a name="multi-select"/>
## Multi-Select
Multi-select support is achieved through these methods on `RealmBasedRecyclerViewAdapter`:
* `boolean isSelected(int position)`
* `int getSelectedItemCount()`
* `List<T> getSelectedRealmObjects()`
* `void setSelected(boolean selected, int position)`
* `void toggleSelected(int position)`
* `void extendSelectionTo(int position)`
* `void selectAll()`
* `void clearSelections()`
* `void notifySelectedItemsChanged()`

Additionally, there are `void saveInstanceState(Bundle out)` and `void restoreInstanceState(Bundle in)` methods which will save and restore the currently selected positions.

You may make use of these how you wish. Here are some things to note, tips, etc:
* The methods which change the set of selected items all call the appropriate `notify*Changed()` methods for you, *but they do not actually modify the state of your views*. In your overridden `onBindViewHolder` method, you should make a call like `selectedPositions.contains(position);` to check and see if the item at that position is currently selected.
* `clearSelections` is automatically called if the adapter is notified by Realm that the data has changed. So you cannot, for example, maintain a set of selected items and do drag-and-drop (At some point I hope to re-work multi-select to remove this limitation)
* The `List` returned by `getSelectedRealmObjects` *is not* managed by Realm
* All of these methods are well-documented, any questions which remain should be answered by referring to their JavaDoc.

<a name="fast-scrolling"/>
## Fast Scrolling
Having a fast scroller is extremely useful in some situations, and I've tried to make the implementation of it as easy as possible while leaving room for flexibility.

There are a few different attributes which are associated with the fast scrolling feature:
```xml
<com.bkromhout.rrvl.RealmRecyclerView
        android:id="@+id/recycler"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        app:fastScroll="true"
        app:autoHideFastScrollHandle="true"
        app:handleAutoHideDelay="1000"
        app:useFastScrollBubble="true"/>
```
Here's what the attributes (and their associated methods on `RealmRecyclerView`) do

| Attribute/Method | Method | Description |
|:---:|:---:|:---:|
| `fastScroll` | `setFastScroll` | Turns the fast scroller on. The default vertical scrollbar will be used if it's off.|
| `autoHideFastScrollHandle` | `setAutoHideFastScrollHandle` | For if you want the fast scroller's handle to auto-hide after a delay instead of staying visible. False by default. |
| `handleAutoHideDelay` | `setHandleAutoHideDelay` | For if you want the handle's auto-hide delay to be something other than the default 2000 milliseconds. |
| `useFastScrollBubble` | `setUseFastScrollBubble` | For if you want the fast scroller to display a bubble next to the handle while using it to scroll. False by default. |

Other than the last one, these attributes are all you need to set if you want to have fast scrolling functionality.

To have the fast scroller show a bubble (akin to the stock Android Contacts app), you need to both set that last one to `true` as well as have some class implement the [`BubbleTextProvider` interface][BubbleTextProvider Class], which defines one method, `getFastScrollBubbleText`. That method provides the position of the item in the adapter and expects the text which should be shown in the bubble in return.

Our sample application's [`ItemAdapter` class][ItemAdapter Class] implements this method like so:
```java
@Override
public String getFastScrollBubbleText(int position) {
    return String.valueOf(realmResults.get(position).name.charAt(0));
}
```

And then in the [`MainActivity` class][MainActivity Class], we pass the adapter to the `RealmRecyclerView` both as the adapter (of course) and as the bubble text provider at the end of    `onCreate`, like this:
```java
recyclerView.setAdapter(adapter);
recyclerView.setBubbleTextProvider((ItemAdapter) adapter);
```

That's all there is to it! Note that while I chose to have the adapter implement the `getFastScrollBubbleText` method in my example, you could have some other object implement it if you so choose. Just remember that all you're given to work with is a position, so that object would need to have a copy of the same `RealmResults` that your adapter has in the first place.

<a name="handle-state-notifications"/>
### Handle State Notifications
Having a fast scroller is great, but sadly Android's built-in classes, like `CoordinatorLayout`, don't really know about it, so in some cases you might need to do a bit of work yourself to make your views play nice.

A good example of this is when you have a `FloatingActionButton` on the screen with your `RealmRecyclerView`. While you *can* create a "behavior" which will cause the `FloatingActionButton` to automatically show/hide itself as you scroll the `RealmRecyclerView` up/down, that behavior class won't pick up on the scrolling done with the fast scroller, only normal scrolling 😞.

For my use case, I wanted the `FloatingActionButton` to hide itself when I grabbed the fast scroll handle, so I created the [`FastScrollHandleStateListener` interface][FastScrollHandleStateListener Class]. Here's how the [`MainActivity` class][MainActivity Class] implements it:
```java
@Override
public void onHandleStateChanged(FastScrollerHandleState newState) {
    switch (newState) {
        case VISIBLE:
            Log.d("MainActivity", "Handle visible.");
            break;
        case HIDDEN:
            Log.d("MainActivity", "Handle hidden.");
            break;
        case PRESSED:
            // Hide the FloatingActionButton.
            fab.hide();
            Log.d("MainActivity", "Handle pressed.");
            break;
        case RELEASED:
            Log.d("MainActivity", "Handle released.");
            break;
    }
}
```

While I've only used the `PRESSED` state to do something here, you can see that there are a total of four states which you'll be notified of.  
Note that the `VISIBLE` and `HIDDEN` states are only triggered if you have auto-hide on, and they're triggered *after the show/hide animation completes*.

<a name="fast-scroller-customization"/>
### Fast Scroller Customization
It's a pretty sure bet that the default colors for the fast scroller isn't the one you want (it's that hot pink accent color that you get when you create a new app project in Android Studio 😉).  
That, along with a number of other things, can be changed by overriding the following resources in the appropriate files in your project (I've included the defaults here):

* In `colors.xml`:  
```xml
<color name="rrvl_handle_inactive_color">#757575</color>
<color name="rrvl_handle_active_color">#FF4081</color>
<color name="rrvl_bubble_color">#FF4081</color>
<color name="rrvl_bubble_text_color">@android:color/white</color>
```
* In `dimens.xml`:  
```xml
<dimen name="rrvl_bubble_text_size">48sp</dimen>
<dimen name="rrvl_handle_margin_end">8dp</dimen>
<dimen name="rrvl_handle_padding_start">8dp</dimen>
```

[Minerva]: https://github.com/bkromhout/Minerva
[CHANGELOG]: CHANGELOG.md
[RRV]: https://github.com/thorbenprimke/realm-recyclerview
[MainActivity Class]: sample/src/main/java/com/bkromhout/rrvl/sample/MainActivity.java
[Item Class]: sample/src/main/java/com/bkromhout/rrvl/sample/Item.java
[ItemAdapter Class]: sample/src/main/java/com/bkromhout/rrvl/sample/ItemAdapter.java
[ItemDragHelper Class]: sample/src/main/java/com/bkromhout/rrvl/sample/ItemDragHelper.java
[BubbleTextProvider Class]: library/src/main/java/com/bkromhout/rrvl/BubbleTextProvider.java
[FastScrollHandleStateListener Class]: library/src/main/java/com/bkromhout/rrvl/FastScrollHandleStateListener.java
[RealmBasedRecyclerViewAdapter Class]: library/src/main/java/io/realm/RealmBasedRecyclerViewAdapter.java
[Ordering Notes]: md-files/ordering-scheme-notes.md
[Origin]: md-files/origin.md

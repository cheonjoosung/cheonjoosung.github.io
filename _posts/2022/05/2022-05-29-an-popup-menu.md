---
title: Android Popup Menu (팝업 메뉴 만들기)
tags: [Android]
style: fill
color: dark
description: Android Popup Menu (팝업 메뉴 만들기)
---

## 개요
팝업 메뉴를 보여줄 때 안드로이드에서 제공하는 menu 를 활용하여 개발이 가능하다. 하지만, 아이콘을 넣고 추가적인 액션이 필요한 팝업 메뉴는 안드로이드 기본 메뉴로만은 불가능 하다.


## 1. 안드로이드 기본 제공 Popup Menu
```kotlin
btnPopup.setOnClickListener {
	createPopupMenu(it)
}

private fun createPopupMenu(view: View) {
    PopupMenu(applicationContext, view).apply {

        menuInflater.inflate(R.menu.item_popup, this.menu)

        setOnMenuItemClickListener {
            when (it.itemId) {
                R.id.action_hi -> {
                    Toast.makeText(this@MainActivity, "HI", Toast.LENGTH_SHORT).show()
                }
                R.id.action_bye -> {
                    Toast.makeText(this@MainActivity, "Bye", Toast.LENGTH_SHORT).show()
                }
                R.id.action_good -> {
                    Toast.makeText(this@MainActivity, "Good", Toast.LENGTH_SHORT).show()
                }
            }

            false
        }

    }.show()

}
```

- res 우클릭 > new > Android Resource Directory > menu
- menu 우클릭 > new > Android Resource Menu > R.menu.item_popup.xml 생성

```kotlin

<?xml version="1.0" encoding="utf-8"?>
<menu xmlns:android="http://schemas.android.com/apk/res/android">
    <item
        android:id="@+id/action_hi"
        android:title="HI" />

    <item
        android:id="@+id/action_bye"
        android:title="bye" />

    <item
        android:id="@+id/action_good"
        android:title="Good" />
</menu>

```



## 2. Custom PopUp Menu
![preview](https://github.com/cheonjoosung/Android-Simple-Popup/raw/master/image/sample.jpeg)

- 메뉴를 담을 팝업 item_simple_popup.xml

```kotlin
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="250dp"
    android:layout_height="wrap_content"
    android:background="@color/cardview_light_background"
    android:orientation="vertical">

    <LinearLayout
        android:id="@+id/ll_popup"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="vertical" />

    <View
        android:layout_width="match_parent"
        android:layout_height="0.5dp"
        android:background="@color/material_dynamic_neutral0" />

</LinearLayout>
```

- 메뉴 아이템 레이아웃 추가 item_simple_popup.xml

```kotlin
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="50dp">

    <ImageView
        android:id="@+id/iv_menu_icon"
        android:layout_width="24dp"
        android:layout_height="24dp"
        android:layout_marginEnd="16dp"
        android:contentDescription="@string/icon"
        android:src="@drawable/ic_baseline_menu_24"
        app:layout_constraintEnd_toStartOf="@id/tv_menu_title"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintBottom_toBottomOf="@id/v_line"
        app:layout_constraintTop_toTopOf="parent" />

    <TextView
        android:id="@+id/tv_menu_title"
        android:layout_width="0dp"
        android:layout_height="wrap_content"
        android:ellipsize="end"
        android:maxLines="1"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintStart_toEndOf="@id/iv_menu_icon"
        app:layout_constraintTop_toTopOf="parent"
        app:layout_constraintBottom_toBottomOf="@id/v_line"
        tools:text="menu_title" />

    <View
        android:id="@+id/v_line"
        android:layout_width="match_parent"
        android:layout_height="1dp"
        android:layout_alignParentBottom="true"
        android:layout_marginTop="2dp"
        android:background="@color/material_dynamic_neutral10"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintTop_toBottomOf="@id/iv_menu_icon" />

</androidx.constraintlayout.widget.ConstraintLayout>
```

메뉴 아이템을 변경하고 싶다면 위의 레이아웃을 수정


- 커스텀 팝업 윈도우

```kotlin

import android.app.ActionBar
import android.content.Context
import android.util.TypedValue
import android.view.LayoutInflater
import android.view.View
import android.view.ViewGroup
import android.widget.ImageView
import android.widget.LinearLayout
import android.widget.PopupWindow
import android.widget.TextView
import androidx.core.content.ContextCompat
import com.example.simple_popup.databinding.LayoutSimplePopupBinding

class SimplePopup(
    context: Context,
    popupList: MutableList<SimplePopupValue>,
    menuItemListener: (View?, SimplePopupValue, Int) -> Unit,
) : PopupWindow(context) {

    init {
        val inflater = LayoutInflater.from(context)

        val binding = LayoutSimplePopupBinding.inflate(inflater, null, false)

        // PopupWindow Value Setting
        contentView = binding.root
        width = getDp(context, 250.0f)

        val layoutParam = LinearLayout.LayoutParams(LinearLayout.LayoutParams.MATCH_PARENT,
            ActionBar.LayoutParams.WRAP_CONTENT)

        for (i in 0 until popupList.size) {
            val view = inflater.inflate(R.layout.item_simple_popup, null, false)

            val tvMenuTitle = view.findViewById(R.id.tv_menu_title) as TextView
            val ivMenuIcon = view.findViewById(R.id.iv_menu_icon) as ImageView
            val vLine = view.findViewById(R.id.v_line) as View

            tvMenuTitle.text = popupList[i].title
            ivMenuIcon.setImageResource(popupList[i].resId)

            vLine.visibility = if (i < popupList.size - 1) View.VISIBLE else View.INVISIBLE

            view.setOnClickListener {
                menuItemListener.invoke(it, popupList[i], i)
                dismiss()
            }

            binding.llPopup.addView(view, layoutParam)
        }

        val layout = contentView
        layout.measure(ViewGroup.LayoutParams.WRAP_CONTENT, ViewGroup.LayoutParams.WRAP_CONTENT)
        height = layout.measuredHeight

        setBackgroundDrawable(ContextCompat.getDrawable(context, R.drawable.bg_transparent))
    }

    private fun getDp(context: Context, value: Float): Int {
        val dm = context.resources.displayMetrics
        return TypedValue.applyDimension(TypedValue.COMPLEX_UNIT_DIP, value, dm).toInt()
    }

}

/**
 * data class
 * title: 목록 타이틀
 * resId: 이미지
 */
data class SimplePopupValue(
    val title: String = "",
    val resId: Int = R.drawable.ic_baseline_menu_24
)
```

SimplePopupValue 를 통해 아이콘과 타이틀을 사용

- 샘플 코드

```kotlin
val tv = findViewById<TextView>(R.id.tv_title)
tv.setOnClickListener {
    val list = mutableListOf<SimplePopupValue>().apply {
        add(SimplePopupValue("menu_title_01",R.drawable.ic_baseline_looks_one_24))
        add(SimplePopupValue("menu_title_02",R.drawable.ic_baseline_looks_two_24))
        add(SimplePopupValue("menu_title_03",R.drawable.ic_baseline_looks_3_24))
    }

    SimplePopup(context = applicationContext, popupList = list) {  _, popupValue, position ->
        when (position) {
            0 -> {
                Toast.makeText(applicationContext, "Clicked $position ${popupValue.title}", Toast.LENGTH_SHORT).show()
            }

            1 -> {
                Toast.makeText(applicationContext, "Clicked $position ${popupValue.title}", Toast.LENGTH_SHORT).show()
            }

            2 -> {
                Toast.makeText(applicationContext, "Clicked $position ${popupValue.title}", Toast.LENGTH_SHORT).show()
            }
        }
    }.apply {
        isOutsideTouchable = true
        isTouchable = true
        showAsDropDown(it, 60, 10)
    }

}
```

---
title: Android RecyclerView ViewType 사용하여 여러 ViewHolder 사용하기
tags: [Android]
style: fill
color: dark
description: Android RecyclerView ViewType 으로 ViewHolder 나누기
---

## 설명
RecyclerView는 리스트를 보여줄 때 주로 사용한다.

RecyclerView.Adapter<>를 implement 하면 3가지 메소드(onCreateViewHolder, onBindViewHolder, getItemCount())를 반드시 구현해야 한다. 추가적으로 `getItemViewType`()라는 오버라이딩 가능한 메소드가 하나 더 있다.



**getItemViewType()**
이 메소드는 여러개의 ViewHodler를 사용하는 경우에 본인 타입에 맞는 레이아웃과 데이터를 bind() 하기 위해서 이다.

복잡한 화면구성이 아닌 이상 ViewHolder 하나만 사용해도 되지만 카카오톡의 연락처의 레이아웃의 경우 필요하다.

**왜냐하면 결과를 먼저 말하면 nestedScrollView를 사용하면서 버벅이거나 멈추는 현상이 발생할 수 있다.**

카카카 톡의 친구 탭의 레이아웃은 “나", “멀티프로필", “친구", “업데이트 친구", “즐겨찾기", “친구목록”, “채널" 등으로 나누어졌있다.

nestedScrollView - Linear - recyclerContact 로 구성할 수 있고 Linear - rvMe, rvFriend, rvUpdate….. etc 의 여러개 recyclerView로 구성될 수 있다.

후자인 여러 개의 recyclerView를 사용할 경우 연락처 목록이 적다면 레이아웃은 그리는데 크게 문제가 발생하지 않는다. 연락처가 1000개 이상인 경우 여러 rv를 그리다가 화면이 멈추거나 다른뷰가 그려지는 현상이 발생합니다. **정확한 키워드는 모르겠지만 메모리 및 뷰 재사용과 관련한 이슈라고 생각을 했습니다.**

**결론은 nestedScrollView안에 여러개의 recyclerView를 사용하지말고 하나의 recyclerView안에 ViewType을 나누어서 화면을 그려야 한다는 점입니다.**




**ViewBinding**

```javascript
android {
	... 생략
	viewBinding {
		enabled = true
	}
}
```
Gradle 에 추가

**MainActivity**
```javascript
import android.os.Bundle
import androidx.appcompat.app.AppCompatActivity
import com.example.sample_adapter.databinding.ActivityMainBinding

class MainActivity : AppCompatActivity() {

    private lateinit var binding: ActivityMainBinding

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        binding = ActivityMainBinding.inflate(layoutInflater)
        setContentView(binding.root)

        binding.btnEmpty.setOnClickListener {
            binding.rvList.adapter = MyListAdapter(mutableListOf())
        }

        binding.btnFull.setOnClickListener {
            val list = mutableListOf<Contact>()
            (1..20).forEach { list.add(Contact("name $it", "010-1234-5678")) }
            binding.rvList.adapter = MyListAdapter(list)
        }
    }
}
```

```javascript
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".MainActivity">

    <Button
        android:id="@+id/btn_full"
        android:layout_width="0dp"
        android:layout_height="wrap_content"
        android:layout_margin="4dp"
        android:text="full list\nadapter"
        app:layout_constraintEnd_toStartOf="@id/btn_empty"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toTopOf="parent" />

    <Button
        android:id="@+id/btn_empty"
        android:layout_width="0dp"
        android:layout_height="wrap_content"
        android:layout_margin="4dp"
        android:text="empty list\nadapter"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintStart_toEndOf="@id/btn_full"
        app:layout_constraintTop_toTopOf="parent" />

    <androidx.recyclerview.widget.RecyclerView
        android:id="@+id/rv_list"
        android:layout_width="0dp"
        android:layout_height="0dp"
        android:layout_margin="16dp"
        app:layoutManager="androidx.recyclerview.widget.LinearLayoutManager"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toBottomOf="@id/btn_full" />

</androidx.constraintlayout.widget.ConstraintLayout>

```


**MyListAdapter**
```javascript
import com.example.sample_adapter.databinding.ItemEmptyBinding
import com.example.sample_adapter.databinding.ItemMeBinding
import com.example.sample_adapter.databinding.ItemPeopleBinding
import android.view.LayoutInflater
import android.view.ViewGroup
import androidx.recyclerview.widget.RecyclerView

class MyListAdapter(
    private val peoples: MutableList<Contact>
) : RecyclerView.Adapter<RecyclerView.ViewHolder>() {

    enum class ViewType {
        ME, PEOPLE, EMPTY
    }

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): RecyclerView.ViewHolder {
        when (viewType) {
            ViewType.ME.ordinal -> {
                val binding =
                    ItemMeBinding.inflate(LayoutInflater.from(parent.context), parent, false)
                return MeViewHolder(binding)
            }

            ViewType.PEOPLE.ordinal -> {
                val binding =
                    ItemPeopleBinding.inflate(LayoutInflater.from(parent.context), parent, false)
                return PeopleViewHolder(binding)
            }

            ViewType.EMPTY.ordinal -> {
                val binding =
                    ItemEmptyBinding.inflate(LayoutInflater.from(parent.context), parent, false)
                return EmptyViewHolder(binding)
            }

            else -> {
                val binding =
                    ItemEmptyBinding.inflate(LayoutInflater.from(parent.context), parent, false)
                return EmptyViewHolder(binding)
            }
        }
    }

    override fun onBindViewHolder(holder: RecyclerView.ViewHolder, position: Int) {
        when (holder) {
            is MeViewHolder -> holder.bind()
            is PeopleViewHolder -> holder.bind(peoples[position - 1])
            is EmptyViewHolder -> holder.bind()
        }
    }

    override fun getItemViewType(position: Int): Int {
        return if (peoples.isEmpty()) {
            ViewType.EMPTY.ordinal
        } else {
            if (position == 0) {
                ViewType.ME.ordinal
            } else {
                ViewType.PEOPLE.ordinal
            }
        }
    }

    override fun getItemCount(): Int = peoples.size + 1

    class PeopleViewHolder(private val binding: ItemPeopleBinding) :
        RecyclerView.ViewHolder(binding.root) {

        fun bind(contact: Contact) {
            binding.tvName.text = contact.name
            binding.tvPhone.text = contact.phoneNumber
        }
    }

    class MeViewHolder(private val binding: ItemMeBinding) :
        RecyclerView.ViewHolder(binding.root) {

        fun bind() {
            binding.tvName.text = "나"
        }
    }

    class EmptyViewHolder(private val binding: ItemEmptyBinding) :
        RecyclerView.ViewHolder(binding.root) {

        fun bind() {
        }
    }

}
```

**item_empty.xml**
```javascript
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:layout_gravity="center"
    android:layout_margin="8dp"
    android:orientation="vertical">

    <TextView
        android:id="@+id/tv_empty"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:textSize="36sp"
        android:textStyle="bold"
        android:text="Contact List Empty" />

</LinearLayout>

```

**item_me.xml**
```javascript
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:layout_margin="8dp"
    android:orientation="vertical">

    <TextView
        android:id="@+id/tv_name"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="Me"
        android:textSize="16sp"
        android:textStyle="bold"
        app:drawableLeftCompat="@drawable/ic_baseline_me_24" />

</LinearLayout>

```

**item_people.xml**
```javascript
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:layout_margin="8dp"
    android:orientation="vertical">

    <TextView
        android:id="@+id/tv_name"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:textSize="16sp"
        android:textStyle="bold"
        tools:text="이것은 이름입니다." />

    <TextView
        android:id="@+id/tv_phone"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_marginTop="4dp"
        android:textSize="16sp"
        android:textStyle="bold"
        tools:text="이것은 전화번호입니다." />

</LinearLayout>
```

---
title: Android RecyclerView 클릭 이벤트 처리 방법 (Adapter 내에서 안전하게 구현)
tags: [Android]
style: fill
color: dark
description: Android RecyclerView 아이템 클릭 이벤트를 처리하는 안전하고 실용적인 방법을 소개합니다.
---

## ✨ 개요

`RecyclerView`는 리스트 형태의 UI를 구성할 때 가장 많이 쓰이는 컴포넌트입니다.  
하지만 단순한 리스트뿐 아니라 **아이템 클릭** 등의 이벤트 처리가 필요한 경우, Adapter에서 어떻게 처리할지 고민이 생기기 마련이죠.

이 포스트에서는 **RecyclerView의 아이템 클릭을 안전하고 효율적으로 처리하는 기본적인 방법**을 소개합니다.

---

## 1. ✅ 인터페이스 방식

### 1.1 Adapter

```kotlin
import android.view.LayoutInflater
import android.view.ViewGroup
import androidx.recyclerview.widget.RecyclerView
import com.js.sample.databinding.ItemMyBinding

class MyAdapter(
    private val items: List<String>,
    val listener: OnItemClickListener,
) : RecyclerView.Adapter<MyAdapter.MyViewHolder>() {

    inner class MyViewHolder(private val binding: ItemMyBinding) :
        RecyclerView.ViewHolder(binding.root) {

        fun bind(text: String) {
            binding.textView.text = text

            binding.root.setOnClickListener {
                listener.onItemClick(adapterPosition, text)
            }

            binding.root.setOnLongClickListener {
                listener.onItemLongCLick(adapterPosition, text)
                true
            }
        }
    }

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): MyViewHolder {
        val binding = ItemMyBinding.inflate(LayoutInflater.from(parent.context), parent, false)
        return MyViewHolder(binding)
    }

    override fun onBindViewHolder(holder: MyViewHolder, position: Int) {
        holder.bind(items[position])
    }

    override fun getItemCount(): Int = items.size
}
```

### 1.2 인터페이스 및 호출코드

```kotlin
import android.os.Bundle
import android.util.Log
import androidx.appcompat.app.AppCompatActivity
import com.js.sample.databinding.ActivityMainBinding

class MainActivity : AppCompatActivity(), OnItemClickListener {

    private lateinit var binding: ActivityMainBinding

    private val TAG: String = "MainActivity"

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = ActivityMainBinding.inflate(layoutInflater)
        setContentView(binding.root)

        val list = mutableListOf<String>()
        repeat(100) { list.add("String Add $it") }
        val adapter = MyAdapter(list, this)
        binding.recyclerView.adapter = adapter
    }

    override fun onItemClick(position: Int, item: String) {
        Log.e(TAG, "onItemClick $position $item")
    }

    override fun onItemLongCLick(position: Int, item: String) {
        Log.e(TAG, "onItemLongCLick $position $item")
    }
}

interface OnItemClickListener {
    fun onItemClick(position: Int, item: String)
    fun onItemLongCLick(position: Int, item: String)
}
```

---

## 2. ✅ 람다 방식

### 2.1 Adapter 코드

```kotlin
import android.view.LayoutInflater
import android.view.ViewGroup
import androidx.recyclerview.widget.RecyclerView
import com.js.sample.databinding.ItemMyBinding

class MyAdapter(
    private val items: List<String>,
    val itemClick: ((Int, String) -> Unit),
    val itemLongClick: ((Int, String) -> Unit),
) : RecyclerView.Adapter<MyAdapter.MyViewHolder>() {

    inner class MyViewHolder(private val binding: ItemMyBinding) :
        RecyclerView.ViewHolder(binding.root) {

        fun bind(text: String) {
            binding.textView.text = text

            binding.root.setOnClickListener {
                itemClick.invoke(adapterPosition, text)
            }

            binding.root.setOnLongClickListener {
                itemLongClick.invoke(adapterPosition, text)
                true
            }
        }
    }

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): MyViewHolder {
        val binding = ItemMyBinding.inflate(LayoutInflater.from(parent.context), parent, false)
        return MyViewHolder(binding)
    }

    override fun onBindViewHolder(holder: MyViewHolder, position: Int) {
        holder.bind(items[position])
    }

    override fun getItemCount(): Int = items.size
}
```

### 2.2 메인 코드

```kotlin
import android.os.Bundle
import android.util.Log
import androidx.appcompat.app.AppCompatActivity
import com.js.sample.databinding.ActivityMainBinding

class MainActivity : AppCompatActivity() {

    private lateinit var binding: ActivityMainBinding

    private val TAG: String = "MainActivity"

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = ActivityMainBinding.inflate(layoutInflater)
        setContentView(binding.root)

        val list = mutableListOf<String>()
        repeat(100) { list.add("String Add $it") }
        val adapter = MyAdapter(list,
            itemClick = { position, text ->
                Log.e(TAG, "itemClick $position $text")
            },
            itemLongClick = { position, text ->
                Log.e(TAG, "itemLongClick $position $text")
            }
        )
        binding.recyclerView.adapter = adapter
    }
}
```

---

## 3.🧠 **결론 & 보안 팁**

- ViewHolder에서 adapterPosition 사용하면 안전하게 클릭 위치를 확인할 수 있습니다.
- 이벤트 처리 방식은 인터페이스 방식과 람다 방식 모두 실무에서 널리 사용됩니다.
- 상황에 맞게 코드 재사용성과 가독성을 고려해 선택하세요!
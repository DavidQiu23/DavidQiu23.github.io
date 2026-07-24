---
title:  "Android UI 元件小抄：水平線、Spinner、Dialog"
tags:
    - Android
---

一些常用的 Android UI 元件小片段整理。

## 繪製水平線 (Horizontal Line)

在 XML 佈局中繪製簡單分隔線的方式：

```xml
<View
    android:id="@+id/line"
    android:layout_width="match_parent"
    android:layout_height="1dip"
    android:background="#686868" />
```

## Spinner 下拉選單

實作簡單的 Spinner 選單與點選事件處理：

```kotlin
val lunch = arrayListOf("雞腿飯", "魯肉飯", "排骨飯", "水餃", "陽春麵")  
val adapter = ArrayAdapter(this, android.R.layout.simple_spinner_dropdown_item, lunch)  
spinner.adapter = adapter  
spinner.onItemSelectedListener = object: AdapterView.OnItemSelectedListener {  
    override fun onItemSelected(parent: AdapterView<*>, view: View, pos: Int, id: Long) =  
        Toast.makeText(this@MainActivity, "你選的是：" + lunch[pos], Toast.LENGTH_SHORT).show()  
  
    override fun onNothingSelected(parent: AdapterView<*>) {}  
}
```

## Dialog 對話框

使用 `AlertDialog.Builder` 載入自定義佈局：

```kotlin
val builder = AlertDialog.Builder(this)
val v = LayoutInflater.from(this).inflate(R.layout.dialog_tran_s2_detail_btn1, null)
builder.setView(v)
val dialog = builder.create()

// 提醒：取得 View 中的元件需在 builder.create() 之後進行
val date = v.findViewById<Button>(R.id.btn_date) 
dialog.show()
```

---
title:  "Android RecyclerView 列表實作"
tags:
    - Android
---

## 分隔線顯示優化

解決重複新增分隔線導致的顯示問題：

```kotlin
fun RecyclerView.addItemDecoration(context: Context){
    if(this.itemDecorationCount == 0){
        this.addItemDecoration(
            DividerItemDecoration(context, DividerItemDecoration.VERTICAL)
        )
    }
}

binding.recyclerView.addItemDecoration(DividerItemDecoration(context, DividerItemDecoration.VERTICAL))
binding.recyclerView.layoutManager = LinearLayoutManager(context)
binding.recyclerView.adapter = RecycleAdapter(json)
```

## Adapter 實作範例

```kotlin
inner class RecycleAdapter(private var dataSet: JSONArray) : RecyclerView.Adapter<RecycleAdapter.ViewHolder>() {
    
    // 定義 ViewHolder
    inner class ViewHolder(view: View) : RecyclerView.ViewHolder(view) {
        val size: TextView = view.findViewById(R.id.txt_size)
        val freeze: TextView = view.findViewById(R.id.txt_freeze)
        val total: TextView = view.findViewById(R.id.txt_total)
        val read: TextView = view.findViewById(R.id.txt_read)
        val unread: TextView = view.findViewById(R.id.txt_unread)
    }

    override fun onCreateViewHolder(viewGroup: ViewGroup, viewType: Int): ViewHolder {
        val view = LayoutInflater.from(viewGroup.context).inflate(R.layout.item_fragment_tran_s3_stat, viewGroup, false)
        return ViewHolder(view)
    }

    override fun getItemCount(): Int = dataSet.length()

    // 提醒：複寫以下方法可避免 RecyclerView 回收機制導致的顯示錯誤
    override fun getItemId(position: Int): Long = position.toLong()

    override fun getItemViewType(position: Int): Int = position

    // 資料綁定
    override fun onBindViewHolder(viewHolder: ViewHolder, position: Int) {
        val json: JSONObject = dataSet.getJSONObject(position)
        viewHolder.size.text = json.getString("PACK_SIZE")
        viewHolder.freeze.text = json.getString("FREEZE_FG")
        viewHolder.total.text = json.getString("TOTAL")
        viewHolder.read.text = json.getString("isREAD")
        viewHolder.unread.text = json.getString("unREAD")
    }
}
```

---
title:  "Android OkHttp3 網路請求"
tags:
    - Android
---

## 同步/非同步呼叫範例

```kotlin
fun clickLogin(view: View){
    val url = App.IP.decrypt(App.DEFAULT_IP) + "/api/CustUser"
    
    val formBody = FormBody.Builder()
        .add("custcode", binding.editComp.text.toString())
        .build()

    val request = Request.Builder()
        .url(url)
        .post(formBody) // 預設為 GET，加上 post() 則改為 POST 請求
        .build()

    App.client.newCall(request).enqueue(object : Callback {
        override fun onFailure(call: Call, e: IOException) {
            e.printStackTrace()
        }

        override fun onResponse(call: Call, response: Response) {
            val json = JSONObject(response.body!!.string())
        }
    })
}
```

## 請求事件生命週期

![事件流程](/assets/images/okhttp3RequestLifeCycle.jpg)

---
title: "Activity Result API 闪退问题解决"
weight: 1
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

# Activity Result API 闪退问题解决

## 问题复现

在调用 Activity Result API 的 `TakePhotoPreview` 方法时会出现闪退问题。

例如调用摄像头拍摄一张照片并将其加载到 imageView 中：

```kotlin
package com.example.cameraalbumtest

import android.os.Bundle
import androidx.activity.result.contract.ActivityResultContracts
import androidx.appcompat.app.AppCompatActivity
import com.example.cameraalbumtest.databinding.ActivityMainBinding

class MainActivity : AppCompatActivity() {
    private lateinit var binding: ActivityMainBinding

    private val launcherActivity =
        registerForActivityResult(ActivityResultContracts.TakePicturePreview()) {
            if (it != null) {
                binding.imageView.setImageBitmap(it)
            }
        }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = ActivityMainBinding.inflate(layoutInflater)
        val view = binding.root
        setContentView(view)

        binding.takePhotoButton.setOnClickListener {
            launcherActivity.launch(null)
        }
    }
}
```

## 闪退原因

查看源文件发现 `TakePicturePreview` 类中含有如下的方法：

```kotlin
override fun createIntent(context: Context, input: Void?): Intent {
    return Intent(MediaStore.ACTION_IMAGE_CAPTURE)
}
```

分析发现调用了 `MediaStore.ACTION_IMAGE_CAPTURE`，而调用此类 Intent 需要申请 `android.permission.WRITE_EXTERNAL_STORAGE` 权限，否则会出现闪退的问题。

## 解决方法

在 AndroidManifest.xml 中加入下面一行：

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools">

    <!-- 加入以下权限申请 -->
    <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE"/>
    
    <application>
        ...
    </application>
</manifest>
```
---
title: Android 설치된 앱의 서명값 (Signature) 확인 방법
tags: [Android]
style: fill
color: dark
description: Android 설치된 앱의 서명값 (Signature) 확인 방법
---

## 개요

Android 앱 개발에서 **서명(Signature)**은 앱의 신뢰성과 무결성을 검증하는 중요한 요소입니다.
특히 앱 간의 통신이나 보안 민감 기능을 구현할 때, 다른 앱이 신뢰된 앱인지 확인하기 위해 서명 값을 비교하는 방식이 자주 사용됩니다.

- 설치된 앱 목록 가져오기
- Android 11(API 30) 이상의 Package Visibility 제한 대응 방법
- SHA-256 기준 서명 해시값 추출 방법

---

## 1. Android 11+ (API 30 이상): 패키지 정보 보기 위함
```xml
<!-- Android 11 이상에서 모든 앱 정보를 보려면 필요 -->
<uses-permission android:name="android.permission.QUERY_ALL_PACKAGES"
    tools:ignore="QueryAllPackagesPermission" />
```

---

## 2. 설치된 앱 목록 가져오기
- 시스템 앱 제외하고 싶다면 if 주석문을 풀면 됨
- 주석 상태에서는 설치된 앱 목록 전부를 가져옴

```kotlin
private fun getInstalledApps(context: Context): List<AppInfo> {
    val installedApps = context.packageManager.getInstalledApplications(PackageManager.GET_META_DATA)

    return installedApps.mapNotNull { app ->
        val appName = context.packageManager.getApplicationLabel(app).toString()
        val packageName = app.packageName

        // 시스템 앱 제외하고 싶다면 조건 추가
        //if ((app.flags and ApplicationInfo.FLAG_SYSTEM) == 0) {
        AppInfo(appName, packageName)
        //} else null
    }
}

data class AppInfo(
    val name: String,
    val packageName: String
)
```

---

## 3. 서명값 가져오기 
- 확장함수를 이용해서 xx:xx:xx:xx 형태로 사용합니다

```kotlin
private fun getSignatureDigest(context: Context, targetPackage: String): String? {
    return try {
        val packageInfo = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.P) {
            val signingInfo = context.packageManager.getPackageInfo(
                targetPackage,
                PackageManager.GET_SIGNING_CERTIFICATES
            ).signingInfo!!

            if (signingInfo.hasMultipleSigners()) {
                signingInfo.apkContentsSigners
            } else {
                signingInfo.signingCertificateHistory
            }
        } else {
            context.packageManager.getPackageInfo(
                targetPackage,
                PackageManager.GET_SIGNATURES
            ).signatures
        }

        val cert = packageInfo?.get(0)?.toByteArray() ?: byteArrayOf()
        val md = MessageDigest.getInstance("SHA-256")
        val digest = md.digest(cert)

        digest.toHex()
    } catch (e: Exception) {
        e.printStackTrace()
        null
    }
}

// 서명값 Sha256 형태로 표시하기 위한 확장함수
fun ByteArray.toHex(): String = joinToString(separator = ":") { "%02X".format(it) }
```

---

## 4. 호출 예시
- 설치된 목록을 가져와서 서명값을 호출하면 됨
```kotlin
val list = getInstalledApps(applicationContext)
list.forEach {
    val result = getSignatureDigest(applicationContext, it.packageName)
    Log.e("MainActivity", "${it.packageName} - signature=$result")
}
```

---

## 5. 결론
- 주의사항
  - Google Play Signing을 사용하는 앱은 로컬에서 빌드한 APK와 서명이 다를 수 있습니다.
  - 타 앱의 인증 목적이라면 SHA-256 기준 비교를 권장합니다. 
  - 서명 값은 보안 민감 정보이므로 비교 로직을 외부에서 노출하지 않는 것이 좋습니다. 
  - 패키지 가시성 제한이 있기 때문에 Android 11 이상에서는 반드시 queries 설정이 필요
    ```xml
    <manifest>
        <queries>
            <package android:name="com.example.targetapp" />
        </queries>
    </manifest>
    ```
  
- 결론
  + Android에서는 다른 앱의 서명 정보를 통해 앱의 진위 여부를 판단 가능
  + 이를 통해 앱 간 인증, 보안 기능 강화, 위조 앱 방지 등의 다양한 보안 기능을 구현 가능
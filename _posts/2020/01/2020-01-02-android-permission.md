---
title: 안드로이드 권한 처리/권한 요청/퍼미션 체크 6.0(마쉬멜로우 버전) 이상
tags: [Android]
style: fill
color: dark
description: 안드로이드 6.0(마쉬멜로우 버전) 이상부터 스마트폰 내에 저장되어 있는 정보 및 설치된 기능의 접근하기 위해서 퍼미션 체크가 필요합니다.
---

## 안드로이드 퍼미션 체크의 필요성?
 방통위(방송통신위원회 & 행정자치부)에서 스마트폰 내에 저장되어 있는 정보 및 설치된 기능에 무분별하게 접근하는 것을 방지하기 위해서 ['스마트폰 앱 접근권한 개인정보보호 안내서'](https://kcc.go.kr/user.do?mode=view&page=A02030700&dc=&boardId=1099&cp=1&boardSeq=44582)를 발표합니다. 즉, 안드로이드 6.0 이상의 버전부터 사용자가 인지할 수 있도록 권한 처리/요청/퍼미션 체크에 대한 로직을 추가해야 합니다. 


## 안드로이드 접근 권한 처리 방법
스마트폰에서 사용하는 기능에 대해 필수적/선택접 접근 권한을 구분하여 이용자에게 알리고 동의를 받아야 합니다.

안드로이드 버전이 6.0(M: 마쉬멜로우) 인지 체크하고 6.0 미만의 버전일 경우 권한 사용에 대한 내용을 스토어 또는 설정으로 고지해야 합니다.

안드로이드 앱을 실행할 때 퍼미션 요구를 해도 되지만 그것보다 특정 권한/기능을 사용할 때 퍼미션 요구를 하는 것이 사용자가 인지하기 편합니다

```javascript
findViewById(R.id.btn_test).setOnClickListener(new View.OnClickListener() {
  @Override
  public void onClick(View v) {
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
      if (checkSelfPermission(Manifest.permission.READ_PHONE_STATE) == PackageManager.PERMISSION_GRANTED {
        Toast.makeText(getApplicationContext(), "Permission Granted", Toast.LENGTH_SHORT).show();
      } else {
        Toast.makeText(getApplicationContext(), "Permission Not Granted", Toast.LENGTH_SHORT).show(); requestPermissions(new String[]{Manifest.permission.READ_PHONE_STATE}, 1);
      }
    }
  }
});

@Override
  public void onRequestPermissionsResult(int requestCode, @NonNull String[] permissions, @NonNull int[] grantResults) {
      super.onRequestPermissionsResult(requestCode, permissions, grantResults);
 
       switch (requestCode) {
          case 1:
              if(grantResults[0] == PackageManager.PERMISSION_GRANTED) {
                  Toast.makeText(getApplicationContext(), "Pass Permission", Toast.LENGTH_SHORT).show();
              }
              break;
          default:
              break;
        }
    }

```
READ_PHONE_STATE 권한 사용을 위한 체크 예시
1. AndroidManifest.xml 에 <uses-permission android:name="android.permission.READ_PHONE_STATE"> 추가
2. 해당 앱 이용이 6.0(M 마쉬멜로우) 버전 이상인지 체크
3. 권한 승인이 되어있지 않다면 requestPermission을 통해 권한 요청 전달 new String[]{ 원하는 퍼미션 목록 1,2,3, .... }
4. requestCode를 통해 응답받을 코드 작성
5. 결과값을 바탕으로 권한이 제대로 처리가 되었는지 확인 getResults[1, 2, 3]... 순서대로 확인 가능

*EasyCanvas와 EasyCanvas Pro 프로젝트는 앱에서 결제방식을 제외하고는 같은 기능을 사용하므로,*

*한 프로젝트에서 관리하는 것이 더 편하다고 생각되어 flavorDimensions 기능을 적용하는 과정을 정리합니다.*

# flavorDimensions 적용

## flavorDimension이란?

+ Gradle에서 제공하는 빌드 변형 구성 옵션으로 **한 프로젝트에서 여러개의 버전을 생성 가능**

사용 예로는 아래와 같은 방법들이 있습니다.

+ 광고제거 버전, 광고가 보여지는 버전
+ one store 버전, play store 버전
+ 기업 맞춤형 서비스 (한 앱에서 A기업에는 A기능, B기업에는 B기능 추가 요청이 오면 각각 프로젝트를 생성하지 않고 한 베이스 코드에서 분기시킬 수 있다.)


## flavorDimension 적용하기

build.gradle(:app)
```java

    // flavor 카테고리 설정. flavorDimensions를 설정하지 않으면 빌드오류 발생
    // 하나의 flavorDimensions를 설정하면 productFlavors에 dimensions 설정을 하지 않아도 된다. (알아서 선택되는 것 같다.)
    // 여러개의 flavorDimensions를 설정 시 각 productFlavors마다 flavorDimensions를 명시해줘야 합니다.
flavorDimensions "version"

productFlavors {
    bypd { 
        applicationIdSuffix ".bypd" 
        buildConfigField 'boolean', 'INAPP', "true"
        manifestPlaceholders = [ appLabel: "Easycanvas" ]
        signingConfig signingConfigs.bypd
    }
    bypd_pro {
        applicationIdSuffix ".bypdpro"
        buildConfigField 'boolean', 'INAPP', "false"
        manifestPlaceholders = [ appLabel: "Easycanvas Sub" ]
        signingConfig signingConfigs.bypd_pro
    }
}
```

여기까지 설정 후 gradle Sync 하면 Build Variants에 네가지 옵션이 생성됩니다.
+ bypdDebug 
+ bypdRelease 
+ bypd_proDebug 
+ bypd_proRelease

## flavor에 따라 코드 분기

appCenter Key 값, LicenseCheck을 위한 키값 등 EasyCanvas와 EasyCanvasPro 에서 각각 다르게 적용되어야 할 코드들이 있습니다.   

이때 위에서 설정한 buildConfigField를 통해 EasyCanvas에서만 실행할 지, EasyCanvas Pro 에서만 실행할 지 여부를 선택할 수 있습니다.   

만약 EasyCanvas에서 앱을 빌드했을 경우 BuildConfig.INAPP의 값이 true로 설정되어 이지캔버스에서만 동작하게끔 설정할 수 있습니다.   

또한 프로젝트를 Android -> Project 구조로 변경하여 사용 시 각 flavor마다 실행될 클래스를 변경하여도 사용할 수 있습니다.   

그러나 저희 프로젝트의 경우 Preference, SharedContext 객체에 영향을 받는 객체가 상당히 많고,

flavor에 따라 package명이 달라지면서 참조할 클래스를 찾지못하는 오류들이 많이 발생했습니다.   

분리되어야 할 클래스를 제외하고도 의존성때문에 다른 클래스들도 분리하게 되면서 클래스를 분리하여 사용하는 방법은 적절하지 않다고 판단했습니다.   

그러므로, 위에서 설명한 BuildConfig의 변수값에 따라 클래스 내부에서 분기되도록 구성했습니다.

## 리소스 파일 분리하기
    이지캔버스와 이지캔버스 프로에서 앱 아이콘, 구매팝업창의 경우 각각 다른 이미지와 화면을 표시해야 합니다.
    안드로이드 프로젝트를 Android -> Project 구조로 변경한 후 
    bypd/app/src 디렉토리로 이동합니다.

    아래와 같은 구조로 bypd_pro 디렉토리 생성 후 bypd_pro 프로젝트에서 사용할 리소스파일들을 적용할 수 있습니다.
    src 
    ├─bypd_pro
    │  ├─java
    │  └─res
    └─main
      ├─java
      └─res

## 주의사항

종종 Build Variant에서 Active Build Variant를 설정하고 실행 시켰을 때 이전에 선택했었던 Build Variant로 실행되는 경우가 있는데,

이때는 Clean Project -> Rebuild Project 후 다시실행 하시면 정상적으로 실행됩니다.


# 빌드 스크립트 수정

추가로 EasyCanvas 안드로이드 앱의 경우 [2021년 새로운 Android App Bundle 및 타겟 API 레벨 요구사항](https://developers-kr.googleblog.com/2020/12/new-android-app-bundle-and-target-api.html)으로 인해 기존의 APK 빌드 결과물이 아닌, AAB 형식의 빌드 결과물을 업로드 해야 했습니다.

EasyCanvas와 EasyCavnas Pro 통합 프로젝트의 빌드 스크립트를 수정하면서, aab 형식의 빌드 결과물을 뽑아내는 빌드스크립트도 함께 작성했습니다.

빌드 결과물을 aab형식으로 생성할 경우 기존 gradle 파일에서 apk 생성시 결과물 포맷을 지정하는 코드가 적용되지 않아서, 
aab 파일의 경우 batch 빌드 스크립트에서 이름을 변경하도록 했습니다.

## aab 파일에서 apk 추출하기

+ 준비물
1. java (저는 OpenJDK 1.8.0_291 버전 사용했습니다.) 
2. bundletool (1.8.2 버전 사용했습니다.) [Download bundletool](https://github.com/google/bundletool)

+ 현재 연결된 디바이스에 apk 설치
  
1. java -jar "bundletool_path.jar" build-apks --bundle="bundle_path.aab" --output="output_apk_path.apks" 

2. java -jar "bundletool_path_.jar" install-apks --apks=./app-debug.apks


+ apk 파일 추출

1. java -jar "bundletool-all-1.8.2.jar" build-apks 
   --bundle="bypd-Debug-0.0.1.aab" 
   --output="output.apks" --mode=universal 

2. apks 파일의 확장자를 zip으로 변경

3. zip 파일 내부 universal.apk 파일을 사용 (위의 keystore signing 제외하고 설치 시 Play 프로텍트에 의해 차단됨 팝업 발생)
https://stackoverflow.com/questions/53040047/generate-apk-file-from-aab-file-android-app-bundle/57149405#57149405

## aab signing

[앱 서명](https://developer.android.com/studio/publish/app-signing?hl=ko#sign-apk)

안드로이드 공식 문서에서는 구글 플레이에 앱을 등록하기 위해 두가지 서명을 진행해야 한다고 함. 

1. 업로드 키 서명

build.gradle에 이미 등록되어 있으므로, 작업 필요 X

2. 앱 서명

*구글에서 다 해준다고 함.*

Google Play에서 대신 앱 서명 키를 생성하여 앱에 서명하는 데 사용하려면 어떤 작업도 실행하지 않아도 됩니다. 

첫 번째 버전에 서명할 때 사용한 키가 업로드 키가 되며 차후 버전에 서명할 때도 이 키를 사용해야 합니다.

## 주의사항

aab에서 추출한 apk 파일은 디바이스에 설치 시에 아래와 같은 팝업메세지가 발생합니다.

"Play 프로텍트에 의해 차단됨"

Play 프로텍트에서 앱 개발자를 인식할 수 없습니다. 알 수 없는 개발자의 앱은 안전하지 않을 수 있습니다.

이는 추출한 apk 파일의 서명이 되지 않아서 발생하는 문제로 보이며,

무시하고 설치하면 정상적으로 설치 가능합니다.

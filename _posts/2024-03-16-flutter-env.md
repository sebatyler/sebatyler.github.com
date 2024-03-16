---
layout: post
title: Flutter 소셜 로그인 with .env
---

최근에 개발하기 시작한 Flutter 앱에 구글, 애플, 네이버, 카카오 로그인을 적용했다.
구글, 네이버, 카카오의 경우 몇 가지 설정값을 요구하는데, 이것을 코드에 직접 노출하고 싶지 않았고 특히나 private repository에서도 보안을 간과할 수 없기에 GitHub에 푸시되는 것을 피하고 싶었다.
보안과 유지보수의 용이성을 동시에 만족시키기 위해 .env 파일을 활용하기로 했다. 그 과정에서 삽질도 좀 했지만 결국에는 해결했고 내가 적용한 방법을 간략하게 정리해보려고 한다.

소셜 로그인을 위해 추가한 내용을 파일별로 나열하고 설명을 아래에 달려고 한다. 파일의 경로는 Flutter 프로젝트 루트 디렉토리 기준이다.

### .env

```bash
GOOGLE_IOS_URL=ios_url_scheme_from_google_cloud
KAKAO_NATIVE_APP_KEY=kakao_native_app_key
NAVER_CLIENT_ID=naver_client_id
NAVER_CLIENT_SECRET=naver_client_secret
NAVER_IOS_URL=naver_ios_url_scheme
```

> 우선 채워넣어야 하는 설정값을 위와 같이 정의해주었다. Android, iOS 공통으로 사용하기 위해 프로젝트 루트 디렉토리에 파일을 생성했다.

## Android

### android/src/main/AndroidManifest.xml

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
    <application ...>
        <activity ...>
            ...
            <!-- 카카오톡 공유, 카카오톡 메시지 커스텀 URL 스킴 설정 -->
            <intent-filter>
                <action android:name="android.intent.action.VIEW" />
                <category android:name="android.intent.category.DEFAULT" />
                <category android:name="android.intent.category.BROWSABLE" />

                <!-- 카카오톡 공유, 카카오톡 메시지 -->
                <data android:host="kakaolink" android:scheme="kakao${kakaoNativeAppKey}" />
            </intent-filter>
        </activity>
        <!-- 카카오 로그인 커스텀 URL 스킴 설정 -->
        <activity
            android:name="com.kakao.sdk.flutter.AuthCodeCustomTabsActivity"
            android:exported="true">
            <intent-filter android:label="flutter_web_auth">
                <action android:name="android.intent.action.VIEW" />
                <category android:name="android.intent.category.DEFAULT" />
                <category android:name="android.intent.category.BROWSABLE" />

                <!-- 카카오 로그인 Redirect URI -->
                <data android:scheme="kakao${kakaoNativeAppKey}" android:host="oauth"/>
            </intent-filter>
        </activity>
        <!-- 네이버 로그인 설정 -->
        <meta-data android:name="com.naver.sdk.clientId" android:value="${naverClientId}" />
        <meta-data android:name="com.naver.sdk.clientSecret" android:value="${naverClientSecret}" />
        <meta-data android:name="com.naver.sdk.clientName" android:value="App Name" />
    </application>
</manifest>
```

> Android는 이 파일에 카카오, 네이버 로그인 설정값을 채워넣어야 한다. 설정값을 채우기 위한 이름으로 **kakaoNativeAppKey, naverClientId, naverClientSecret**을 지정해주었다.

### android/build.gradle

```gradle
buildscript {
    dependencies {
        classpath "io.github.cdimascio:java-dotenv:5.2.2"
    }
}
```

> **.env** 적용을 위해 여러가지 gradle plugin을 시도했는데 프로젝트 루트 디렉토리의 .env 파일을 가져오는 것이 용이해서 최종적으로는 **java-dotenv**를 사용했다.

### android/app/build.gradle

```gradle
import io.github.cdimascio.dotenv.Dotenv

// 프로젝트 루트 디렉토리의 .env 파일 읽기
def dotenv = Dotenv.configure().directory("..").filename(".env").load()

android {
    defaultConfig {
        // .env 에서 읽은 값을 AndroidManifest.xml 에서 지정한 이름에 대한 값으로 설정
        manifestPlaceholders['kakaoNativeAppKey'] = dotenv['KAKAO_NATIVE_APP_KEY']
        manifestPlaceholders['naverClientId'] = dotenv['NAVER_CLIENT_ID']
        manifestPlaceholders['naverClientSecret'] = dotenv['NAVER_CLIENT_SECRET']
    }
}
```

> **.env** 파일을 읽어서 **AndroidManifest.xml** 파일에서 지정한 이름에 대한 값을 설정해주었다.

Android 완료!

## iOS

### ios/Runner/Info.plist

```xml
<array>
    <dict>
        <key>CFBundleTypeRole</key>
        <string>Editor</string>
        <key>CFBundleURLName</key>
        <string>Google</string>
        <key>CFBundleURLSchemes</key>
        <array>
            <string>$(GOOGLE_IOS_URL)</string>
        </array>
    </dict>
    <dict>
        <key>CFBundleTypeRole</key>
        <string>Editor</string>
        <key>CFBundleURLName</key>
        <string>Kakao</string>
        <key>CFBundleURLSchemes</key>
        <array>
            <string>kakao$(KAKAO_NATIVE_APP_KEY)</string>
        </array>
    </dict>
    <dict>
        <key>CFBundleTypeRole</key>
        <string>Editor</string>
        <key>CFBundleURLName</key>
        <string>Naver</string>
        <key>CFBundleURLSchemes</key>
        <array>
            <string>$(NAVER_IOS_URL)</string>
        </array>
    </dict>
</array>

<array>
    <string>kakaokompassauth</string>
    <string>kakaolink</string>
    <string>kakaoplus</string>
    <string>naversearchapp</string>
    <string>naversearchthirdlogin</string>
</array>
<key>NSAppTransportSecurity</key>
<dict>
    <key>NSAllowsArbitraryLoads</key>
    <true/>
    <key>NSExceptionDomains</key>
    <dict>
        <key>naver.com</key>
        <dict>
            <key>NSExceptionRequiresForwardSecrecy</key>
            <false/>
            <key>NSIncludesSubdomains</key>
            <true/>
        </dict>
        <key>naver.net</key>
        <dict>
            <key>NSExceptionRequiresForwardSecrecy</key>
            <false/>
            <key>NSIncludesSubdomains</key>
            <true/>
        </dict>
    </dict>
</dict>

<key>naverConsumerKey</key>
<string>$(NAVER_CLIENT_ID)</string>
<key>naverConsumerSecret</key>
<string>$(NAVER_CLIENT_SECRET)</string>
<key>naverServiceAppName</key>
<string>App Name</string>
<key>naverServiceAppUrlScheme</key>
<string>$(NAVER_IOS_URL)</string>
```

> iOS는 이 파일에 구글, 카카오, 네이버 로그인 설정값을 채워넣어야 한다. **.env** 에서 정의한 것과 동일한 이름으로 값이 들어갈 자리에 **GOOGLE_IOS_URL, KAKAO_NATIVE_APP_KEY, NAVER_CLIENT_ID, NAVER_CLIENT_SECRET, NAVER_IOS_URL**를 채워넣었다.

### ios/Flutter/Debug.xcconfig

```text
// Generated from .env in Podfile
#include "env.xcconfig"
```

> **.env** 파일을 읽어서 **env.xcconfig**를 생성할 예정이라 이 파일을 불러들이는 코드를 정의했다.

### ios/Flutter/Release.xcconfig

```text
// Generated from .env in Podfile
#include "env.xcconfig"
```

> 여기도 Debug.xcconfig와 동일하다.

### ios/fastlane/Fastfile

```ruby
platform :ios do
  lane :update_xcconfig do
    env_path = File.join("../..", '.env')
    env_xcconfig_path = File.join('..', 'Flutter', 'env.xcconfig')

    # Initialize the content for env.xcconfig
    env_xcconfig_content = "// This file is generated from .env. Do not edit directly.\n"

    # Check if .env exists
    if File.exist?(env_path)
      pattern = /^(GOOGLE|KAKAO|NAVER)_/

      # Read each line of the .env file
      File.readlines(env_path).each { |line| env_xcconfig_content += line if pattern.match(line) }

      # Write the new content to env.xcconfig
      File.write(env_xcconfig_path, env_xcconfig_content)
    else
      puts "Warning: .env file not found at path #{env_path}"
    end
  end
end
```

> 테스트 버전 배포를 위해 Fastlane을 이미 사용하고 있어서 **.env** 파일에서 필요한 부분만 읽어서 **env.xcconfig** 파일을 생성하는 것도 Fastlane의 힘을 빌리기로 했다.

**fastlane update_xcconfig**를 iOS 앱 빌드 전에 매번 실행하기 위해서 xcode 에서 프로젝트를 열고 아래의 2개 스크린샷과 같이 Pre-action을 추가해주었다.

### Product 메뉴 > Scheme > Edit Scheme 클릭

![Edit Scheme](/images/20240316-edit_scheme.png)

### Build > Pre-actions > + 버튼 클릭 > New Run Script Action 클릭

![Pre-action Script](/images/20240316-pre_action.png)

코드는 아래와 같다.

```bash
# ios 디렉토리로 이동
cd $(dirname $WORKSPACE_PATH)

if which fastlane >/dev/null; then
  fastlane update_xcconfig
else
  echo "error: Fastlane not installed!" 1>&2
  exit 1
fi
```

> ios 디렉토리로 이동해야 **fastlane update_xcconfig**를 실행할 수 있는데, 이 경로를 알아내는데 좀 헤맸다. 구글링을 해보면 PROJECT_DIR 환경변수를 쓸 수 있다고 하는데 아니었다. 빌드 에러 로그와 env 명령을 실행해서 확인해보니 **WORKSPACE_PATH**가 유일한 활용 가능한 값이어서 이걸 사용했다.

iOS도 완료!

## 맺음말

Android에서는 gradle plugin과 씨름하다가 java-dotenv를 사용해서 문제를 해결했다.
iOS에서는 사실 Build phase에 Run script를 추가해서 env.xcconfig 파일을 생성하려고 했는데 Build phase 시작 시점에는 이미 Debug.xcconfig 혹은 Release.xcconfig가 로드된 상태여서 env.xcconfig를 찾지 못한다는 에러와 함께 실패했다. 그래서 Pre-action을 시도했고 cd 경로를 찾느라 좀 헤맸지만 결국에는 해결을 했다.
별 것 아닌것 같은 일에 시간과 노력이 꽤 들어갔지만 Android, iOS 앱 빌드 과정에 대해서 조금은 이해하게 된 것 같다.

아직 시도해보지는 않았지만 위의 과정을 거쳐서 해결한 이후에 **[flutter_config](https://pub.dev/packages/flutter_config)** 라는 Flutter 패키지를 발견했다. 자세히 보지는 않았지만 이걸 사용하는게 쉬운 방법일 수 있어 보인다. 누군가는 나와 비슷한 고민을 한게 아닌가 싶다. 진작 알았으면 좋았겠지만 아직 해본 건 아니기에 단언할 수는 없다. 다음에 시간나면 한번 시도해봐야 겠다.
적어도 내가 삽질했던 이야기와 잘 동작하는 코드를 공유한 것이 누군가에게는 가이드가 될 수 있지 않을까 싶고, 아니면 적어도 같은 실수를 반복하지 않게 해줄 수 있을 것 같다.

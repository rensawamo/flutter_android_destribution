# android destribution(2024/2)

### fvm 設定で行う場合の追加
``` sh
fvm list
fvm use (上記ver)
or
fvm use stable
```

### gitigonre の更新

以下で firebaseの設定やAPIのファイルをgitで公開しないようにする
```sh
**/ios/Runner/GoogleService-Info.plist
**/android/app/google-services.json
*.jks
*.pem
**/.fvm/flutter_sdk
**/lib/firebase_options.dart
```

### 使用する エンコードコマンド
```sh
$ base64 -i input.xxx  -o output.pem 
```


###  必要な環境変数
GOOGLE_SERVICES_JSON_BASE64 :  firebaseのセットアップでダウンロードした google-services.json のエンコード


- KEYSTORE_BASE64  :  release.jks(下で説明) のエンコード


- FIREBASE_OPTION_BASE64  :  lib/の firebase_options.dart のエンコード


- KEY_ALIAS, KEY_PASSWORD , STORE_PASSWORD   :  release.jksのパスワードやエイリアス名


- ANDROID_APP_ID  :  firebase のandoroid アプリ名


- CREDENTIAL_FILE_CONTENT  :  GCP トークン


## アップロードkeyの準備

### プロジェクトファイルの andoroi/app で以下のコマンドを実行し証明書を作成する
alias_nameは覚えやすい名前にする

```sh
keytool -genkey -v -keystore release.jks -alias alias_name -keyalg RSA -keysize 2048 -validity 10000
```

### github actionの環境変数で使うためエンコードしておく
```sh
 openssl base64 -in release.jks  -out release.pem 
```

### android/keystore.propertiesを作成（local用）
```sh
storePassword=パスワード
keyPassword=パスワード
keyAlias=alias_name
storeFile=release.jks
```

### android/app/build.gradeの編集
```sh
// android 設定の上
def keystorePropertiesFile = rootProject.file("keystore.properties")
android {
....
// defalutConfigの下
signingConfigs {
        release {
            if (System.getenv("GITHUB_ACTIONS")) { // gitaction用
                storeFile file("release.jks")
                storePassword System.getenv()["STORE_PASSWORD"]
                keyAlias System.getenv()["KEY_ALIAS"]
                keyPassword System.getenv()["KEY_PASSWORD"]
            } else if (keystorePropertiesFile.exists()) { // local用
                def keystoreProperties = new Properties()
                keystoreProperties.load(new FileInputStream(keystorePropertiesFile))
                keyAlias keystoreProperties['keyAlias']
                keyPassword keystoreProperties['keyPassword']
                storeFile file(keystoreProperties['storeFile'])
                storePassword keystoreProperties['storePassword']
            }
        }
    }

....
// この中に releaseの追加
buildTypes {
        release {
            signingConfig signingConfigs.release

```

### 以下コマンドを実行(local apkファイルの出力の確認)
```sh
flutter build apk
```


## firebase 導入


### 上記で作成した 証明書の フィンガプリントを取得
以下のコマンドを実行し SHA1を取得。
```sh
keytool -v -list -keystore release.jks
```
![image](https://github.com/rensawamo/firebase_destribution/assets/106803080/44cc26e4-e5ed-4bf8-8b68-8bc4d833ed4c)

firebase のプロジェクト → andoroid app  を追加のページの 



デバッグ用の署名証明書 SHA-1に上記を張り付け、google-services.jsonをダウンロード


![image](https://github.com/rensawamo/firebase_destribution/assets/106803080/f781ba07-a01c-474d-bf35-05eda06e7b32)



android/build.grade に以下の設定を追加する
```sh
dependencies {
        classpath 'com.google.gms:google-services:4.3.15'
        classpath "org.jetbrains.kotlin:kotlin-gradle-plugin:$kotlin_version"
        classpath 'com.android.tools.build:gradle:7.3.0'
    }
```


android/app/build.gradeの設定を変更
```sh
plugins {
    id "com.android.application"
    id "kotlin-android"
    id "dev.flutter.flutter-gradle-plugin"
    id 'com.google.gms.google-services'
}
....

dependencies {
    implementation platform('com.google.firebase:firebase-bom:32.7.2')
    implementation 'com.google.firebase:firebase-analytics'
}
```

上記の後、firebaseの画面に移動 → アプリの追加 → flutter で
firebase login コマンドなどを順番に実行し、プロジェクトに firebase_option.dartが追加されていることを確認する

![image](https://github.com/rensawamo/firebase_destribution/assets/106803080/3ed7ca87-d419-4bb5-9bc8-2b831be7b779)


### firebase core の導入
[pubget](https://pub.dev/)
から firebase_core のバージョンを指定して

```sh
flutter pub get
```

###  firebase  distribution との接続トークンの発行

https://github.com/wzieba/Firebase-Distribution-Github-Action/wiki/FIREBASE_TOKEN-migration#guide-2---the-same-but-with-screenshots

以下のコマンドは、現在では非推奨。
```sh
firebase login:ci
```




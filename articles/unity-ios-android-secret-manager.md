---
dev_article_id: 792603
title: "Unity で iOS/Android アプリの設定値をセキュアに扱う方法"
emoji: "🔑"
type: "tech"
topics: ["unity", "ios", "android"]
published: true
---

# はじめに

iOS/Android でユーザーの情報をセキュアに扱う必要があったので、調査したところ Android には [EncryptedSharedPreferences](https://developer.android.com/reference/androidx/security/crypto/EncryptedSharedPreferences) が存在することを知りました。iOS には [Keychain Services](https://developer.apple.com/documentation/security/keychain_services) が存在します。

今回は Unity の iOS/Android プラットフォーム上で設定値を保存するための実装を行う必要があったので、Unity から扱えるようネイティブプラグインを作成しました。今後もこういった要望はありそうでしたので、記事として手順や内容を書き記しておくことにしました。

本記事内で紹介しているコードは下記にアップ済みです。

https://github.com/nikaera/Unity-iOS-Android-SecretManager-Sample

# 動作環境

- MacBook Air (M1, 2020)
- Unity 2020.3.15f2
- Android 6.0 以上
  - [EncryptedSharedPreferences](https://developer.android.com/reference/androidx/security/crypto/EncryptedSharedPreferences) が使用可能なバージョン

# Android のネイティブプラグインを作成する

Android 環境ではまず [External Dependency Manager for Unity](https://github.com/googlesamples/unity-jar-resolver) を利用して、Unity の Android ネイティブプラグインで `EncryptedSharedPreferences` 利用可能にします。

## External Dependency Manager for Unity で必要なパッケージをインストールする

`External Dependency Manager for Unity` をインポートするため [unitypackage](https://github.com/googlesamples/unity-jar-resolver/blob/master/external-dependency-manager-latest.unitypackage) をダウンロードして、**`EncryptedSharedPreferences` を導入したい Unity プロジェクトを開いてから `unitypackage` をクリックすることで、`External Dependency Manager for Unity` を Unity プロジェクトにインポートします。**

![ダウンロードした `unitypackage` をクリックして Unity プロジェクトに External Dependency Manager for Unity をインポートする](https://i.gyazo.com/1af7cdf4d7d5749e59e151eef1ca5493.png)

Unity プロジェクトの `Build Settings` からプラットフォームは Android に切り替えておきます。`Enable Android Auto-resolution?` というダイアログの選択肢はどちらを選んでも構いません。[^1]

[^1]: パッケージの依存関係を自動で解決するかどうかという選択肢になります。本記事では明示的に Resolve を実行するため `Disable` でも `Enable` でも進行上の問題はありません。

External Dependency Manager for Unity で各種パッケージを管理する方法は [README](https://github.com/googlesamples/unity-jar-resolver#android-resolver-usage) に記載がある通り、**`*Dependencies.xml` というファイルを `Editor` フォルダに配置することで可能になります。**

今回は `EncryptedSharedPreferences` を導入するため、下記の xml ファイルを `Editor` フォルダ内に配置します。

```xml:Assets/Editor/AndroidPluginDependencies.xml
<?xml version="1.0" encoding="utf-8"?>
<dependencies>
    <androidPackages>
        <!--
            本記事ではバージョン 1.1.0-alpha03 を利用している
        -->
        <androidPackage spec="androidx.security:security-crypto:1.1.0-alpha03">
            <androidSdkPackageIds>
                <!--
                    Google の Maven リポジトリからインストールするため、
                    extra-google-m2repository を指定する
                -->
                <androidSdkPackageId>extra-google-m2repository</androidSdkPackageId>
            </androidSdkPackageIds>
        </androidPackage>
    </androidPackages>
</dependencies>

```

その後、**Unity メニューから `Assets -> External Dependency Manager -> Android Resolver -> Force Resolve` を選択して、`Assets/Editor/AndroidPluginDependencies.xml` の内容を元に `EncryptedSharedPreferences` を利用するのに必要なパッケージを自動で `Assets/Plugins/Android` フォルダにダウンロードします。**

![1. Unity メニューから `Assets -> External Dependency Manager -> Android Resolver -> Force Resolve` を選択する](https://i.gyazo.com/df394e15149e54dae3e9a81848512ee9.png)
**1. Unity メニューから `Assets -> External Dependency Manager -> Android Resolver -> Force Resolve` を選択する**

![2. 実行に成功すると EncryptedSharedPreferences を利用するのに必要なライブラリ群が `Assets/Plugins/Android` フォルダに配置される](https://i.gyazo.com/f6d2ec95ef9c2afdc857fecef2b165e5.png)
**2. 実行に成功すると EncryptedSharedPreferences を利用するのに必要なライブラリ群が `Assets/Plugins/Android` フォルダに配置される**

ここまで来ればあとは Android ネイティブコードを `Assets/Plugins/Android` フォルダ内に配置して Unity 側から叩けるようにするだけです。

## EncryptedSharedPreferences を利用するためのネイティブコードを追加する

早速下記の Android ネイティブコードを `Assets/Plugins/Android/SecretManager.java` に配置します。

```java:Assets/Plugins/Android/SecretManager.java
package com.nikaera;

import com.unity3d.player.UnityPlayerActivity;

import java.lang.Exception;

// External Dependency Manager for Unity によって、
// 必要な jar が含まれているため EncryptedSharedPreferences の利用が可能になる
import androidx.security.crypto.EncryptedSharedPreferences;
import androidx.security.crypto.MasterKey;

import android.content.Context;
import android.content.SharedPreferences;
import android.content.SharedPreferences.Editor;

import android.os.Bundle;
import android.util.Log;

public class SecretManager {
  private SharedPreferences sharedPreferences;
  
  public SecretManager(Context context) {
    try {
        // EncryptedSharedPreferences で設定値を保存する際に用いる、
        // 暗号鍵を扱うためのラッパークラスをデフォルト設定で作成する
        MasterKey masterKey = new MasterKey.Builder(context)
                .setKeyScheme(MasterKey.KeyScheme.AES256_GCM)
                .build();
                
        // EncryptedSharedPreferences のインスタンスを生成する
        // コンストラクタで作成した masterKey を指定している
        this.sharedPreferences = EncryptedSharedPreferences.create(
          context, context.getPackageName(), masterKey,
          EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
          EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM
        );
    } catch (Exception e) {
        e.printStackTrace();
    }
  }
  
  /**
   * 指定したキーで値を保存する関数
   * @param key 値を保存する際に用いるキー
   * @param value 保存したい値
   * @return boolean 値の保存に成功したかどうか
   */
  public boolean put(String key, String value) {
    SharedPreferences.Editor editor = sharedPreferences.edit();
    editor.putString(key, value);

    return editor.commit();
  }
  
  /**
   * 指定したキーで保存した値を取得する関数
   * `put` 関数で保存した値を取得するのに利用する
   * @param key 取得したい値のキー
   * @return string キーに紐づく値、存在しなければ空文字が返却される
   */
  public String get(String key) {
    return sharedPreferences.getString(key, null);
  }
  
  /**
   * 指定したキーで値を削除する関数
   * @param key 削除したい値のキー
   * @return boolean 値の削除に成功したかどうか
   */
  public boolean delete(String key) {
    SharedPreferences.Editor editor = sharedPreferences.edit();
    editor.remove(key);

    return editor.commit();
  }
}

```

その後、上記を Unity スクリプトから実行可能にするための C# クラスを作成します。本記事ではファイルを `Assets/Scripts/EncryptedSharedPreferences.cs` に配置します。

```csharp:Assets/Scripts/EncryptedSharedPreferences.cs
using UnityEngine;

/// <summary>
///     利用するネイティブコードは <c>Assets/Plugins/Android/SecretManager.java</c> に記載
/// </summary>
/// <remarks>
///     <a href="https://developer.android.com/reference/androidx/security/crypto/EncryptedSharedPreferences">EncryptedSharedPreferences</a>
/// </remarks>
class EncryptedSharedPreferences
{
    private readonly AndroidJavaObject _secretManager;
    public EncryptedSharedPreferences()
    {
        // コンストラクタで com.nikaera.SecretManager のインスタンス生成を行う
        var activity = new AndroidJavaClass("com.unity3d.player.UnityPlayer")
            .GetStatic<AndroidJavaObject>("currentActivity");
        var context = activity.Call<AndroidJavaObject>("getApplicationContext");
        _secretManager = new AndroidJavaObject("com.nikaera.SecretManager", context);
    }

    public bool Put(string key, string value)
    {
        return _secretManager.Call<bool>("put", key, value);
    }

    public string Get(string key)
    {
        return _secretManager.Call<string>("get", key);
    }

    public bool Delete(string key)
    {
        return _secretManager.Call<bool>("delete", key);
    }
}

```

あとは用途に応じて下記のようなコードで設定値の保存や取得などを行えます。

```csharp
// ...
var _sharedPreferences = new EncryptedSharedPreferences();

// name をキーとして値を nikaera で保存する
_sharedPreferences.Put("name", "nikaera");

// name をキーとして値を取得する
var name = _sharedPreferences.Get("name");

// "nikaera" が出力される
Debug.Log(name);

//　name をキーとして値を削除する
_sharedPreferences.Delete("name");

// ...
```

# iOS のネイティブプラグインを作成する

iOS の場合は外部ライブラリを利用しないため、`External Dependency Manager for Unity` は利用しません。**本来であれば Swift で信頼できる外部フレームワークを取り込み利用できると良さそうですが、今回は Objective-C でネイティブプラグインを書いていきます。**[^2]

[^2]: [CocoaPods](https://cocoapods.org/) もサポートされているようなので、iOS でも Android 同様、外部ライブラリを取り込むのは簡単にできそうでした。例えば [KeychainAccess](https://github.com/kishikawakatsumi/KeychainAccess) とか使いたい。

## Keychain Services を利用するためのネイティブコードを追加する

早速下記の iOS ネイティブコードを `Assets/Plugins/iOS/KeychainService.mm` に配置します。

```objc:Assets/Plugins/iOS/KeychainService.mm
// Keychain Services を利用するために Security フレームワークを利用する
#import <Security/Security.h>

extern "C"
{
    // 指定したキーで値を保存する関数
    // - param
    //   - dataType:  値を保存する際に用いるキー
    //   - value: 保存したい値
    // - return
    //   - 保存時のステータスコードを返却する (0 以外は失敗)
    int addItem(const char *dataType, const char *value)
    {
        NSMutableDictionary* attributes = nil;
        NSMutableDictionary* query = [NSMutableDictionary dictionary];
        NSData* sata = [[NSString stringWithCString:value encoding:NSUTF8StringEncoding] dataUsingEncoding:NSUTF8StringEncoding];
    
        [query setObject:(id)kSecClassGenericPassword forKey:(id)kSecClass];
        [query setObject:(id)[NSString stringWithCString:dataType encoding:NSUTF8StringEncoding] forKey:(id)kSecAttrAccount];
    
        OSStatus err = SecItemCopyMatching((CFDictionaryRef)query, NULL);
    
        if (err == noErr) {
            attributes = [NSMutableDictionary dictionary];
            [attributes setObject:sata forKey:(id)kSecValueData];
            [attributes setObject:[NSDate date] forKey:(id)kSecAttrModificationDate];
    
            err = SecItemUpdate((CFDictionaryRef)query, (CFDictionaryRef)attributes);
            return (int)err;
        } else if (err == errSecItemNotFound) {
            attributes = [NSMutableDictionary dictionary];
            [attributes setObject:(id)kSecClassGenericPassword forKey:(id)kSecClass];
            [attributes setObject:(id)[NSString stringWithCString:dataType encoding:NSUTF8StringEncoding] forKey:(id)kSecAttrAccount];
            [attributes setObject:sata forKey:(id)kSecValueData];
            [attributes setObject:[NSDate date] forKey:(id)kSecAttrCreationDate];
            [attributes setObject:[NSDate date] forKey:(id)kSecAttrModificationDate];
            err = SecItemAdd((CFDictionaryRef)attributes, NULL);
            return (int)err;
        } else {
            return (int)err;
        }
    }
    
    // 指定したキーで値を取得する関数
    // - param
    //   - dataType: 値を取得する際に用いるキー
    // - return
    //   - キーに紐づく値、存在しなければ空文字が返却される
    char* getItem(const char *dataType)
    {
        NSMutableDictionary* query = [NSMutableDictionary dictionary];
        [query setObject:(id)kSecClassGenericPassword forKey:(id)kSecClass];
        [query setObject:(id)[NSString stringWithCString:dataType encoding:NSUTF8StringEncoding] forKey:(id)kSecAttrAccount];
        [query setObject:(id)kCFBooleanTrue forKey:(id)kSecReturnData];
    
        CFDataRef cfresult = NULL;
        OSStatus err = SecItemCopyMatching((CFDictionaryRef)query, (CFTypeRef*)&cfresult);
    
        if (err == noErr) {
            NSData* passwordData = (__bridge_transfer NSData *)cfresult;
            const char* value = [[[NSString alloc] initWithData:passwordData encoding:NSUTF8StringEncoding] UTF8String];
            char *str = strdup(value);
            return str;
        } else {
            return NULL;
        }
    }
    
    // 指定したキーで値を削除する関数
    // - param
    //   - dataType:  値を削除する際に用いるキー
    // - return
    //   - 保存時のステータスコードを返却する (0 以外は失敗)
    int deleteItem(const char *dataType)
    {
        NSMutableDictionary* query = [NSMutableDictionary dictionary];
        [query setObject:(id)kSecClassGenericPassword forKey:(id)kSecClass];
        [query setObject:(id)[NSString stringWithCString:dataType encoding:NSUTF8StringEncoding] forKey:(id)kSecAttrAccount];
    
        OSStatus err = SecItemDelete((CFDictionaryRef)query);
    
        if (err == noErr) {
            return 0;
        } else {
            return (int)err;
        }
    }
}

```

`Keychain Services` は `Security` フレームワークを利用するため、**`KeychainService.mm` に対して `Security` フレームワークの依存関係を設定する必要があります。**

![`KeychainService.mm` で `Security` フレームワークの利用を可能にする](https://i.gyazo.com/ba82aaced24b83b37bf8c63e1ee7142f.png)
**`KeychainService.mm` で `Security` フレームワークの利用を可能にする**

その後、上記を Unity スクリプトから実行可能にするための C# クラスを作成します。本記事ではファイルを `Assets/Scripts/KeychainService.cs` に配置します。

```csharp:Assets/Scripts/KeychainService.cs
using System.Runtime.InteropServices;

/// <summary>
///     実装は <c>Assets/Plugins/iOS/KeychainService.mm</c> に記載
/// </summary>
/// <remarks>
///     <a href="https://developer.apple.com/documentation/security/keychain_services">Keychain Services</a>
/// </remarks>
class KeychainService
{
#if UNITY_IOS
    [DllImport("__Internal")]
    private static extern int addItem(string dataType, string value);

    [DllImport("__Internal")]
    private static extern string getItem(string dataType);

    [DllImport("__Internal")]
    private static extern int deleteItem(string dataType);
#endif
    
    public bool Put(string key, string value)
    {
#if UNITY_IOS
        // 返却されるステータスが 0 なら成功
        return addItem(key, value) == 0;
#endif
    }

    public string Get(string key)
    {
#if UNITY_IOS
        return getItem(key);
#else
        return null;
#endif
    }

    public bool Delete(string key)
    {
#if UNITY_IOS
        // 返却されるステータスが 0 なら成功
        return deleteItem(key) == 0;
#endif
    }
}

```

あとは用途に応じて下記のようなコードで設定値の保存や取得などを行えます。

```csharp
// ...
var _keychainService = new KeychainService();

// name をキーとして値を nikaera で保存する
_keychainService.Put("name", "nikaera");

// name をキーとして値を取得する
var name = _keychainService.Get("name");

// "nikaera" が出力される
Debug.Log(name);

//　name をキーとして値を削除する
_keychainService.Delete("name");

// ...
```

# (余談) インターフェースで iOS/Android のふるまいを共通化する

このままだとプラットフォームを切り替える毎にコードを書き直さないとならないので、インターフェースを利用して共通化を行います。

```csharp:Assets/Scripts/ISecretManager.cs
public abstract class ISecretManager
{
    /// <summary>
    /// 指定したキーで値を保存する関数
    /// </summary>
    /// <param name="key">キー</param>
    /// <param name="value">値</param>
    /// <returns>保存に成功したかどうか</returns>
    public abstract bool Put(string key, string value); 
    
    /// <summary>
    /// 指定したキーの値を取得する関数
    /// </summary>
    /// <param name="key">キー</param>
    /// <returns>指定したキーで設定された値、無ければ null</returns>
    public abstract string Get(string key);
    
    /// <summary>
    /// 指定したキーの値を削除する関数
    /// </summary>
    /// <param name="key">キー</param>
    /// <returns>削除に成功したかどうか</returns>
    public abstract bool Delete(string key);
}

```

その後、`Assets/Scripts/EncryptedSharedPreferences.cs` および `Assets/Scripts/KeychainService.cs` を下記の通り `ISecretManager` の実装に紐付けます。

```csharp:Assets/Scripts/EncryptedSharedPreferences.cs
using UnityEngine;

/// <summary>
///     利用するネイティブコードは <c>Assets/Plugins/Android/SecretManager.java</c> に記載
/// </summary>
/// <remarks>
///     <a href="https://developer.android.com/reference/androidx/security/crypto/EncryptedSharedPreferences">EncryptedSharedPreferences</a>
/// </remarks>
class EncryptedSharedPreferences: ISecretManager
{
    private readonly AndroidJavaObject _secretManager;
    public EncryptedSharedPreferences()
    {
        var activity = new AndroidJavaClass("com.unity3d.player.UnityPlayer")
            .GetStatic<AndroidJavaObject>("currentActivity");
        var context = activity.Call<AndroidJavaObject>("getApplicationContext");
        _secretManager = new AndroidJavaObject("com.nikaera.SecretManager", context);
    }
    
    #region ISecretManager

    public override bool Put(string key, string value)
    {
        return _secretManager.Call<bool>("put", key, value);
    }

    public override string Get(string key)
    {
        return _secretManager.Call<string>("get", key);
    }

    public override bool Delete(string key)
    {
        return _secretManager.Call<bool>("delete", key);
    }
    
    #endregion
}

```

```csharp:Assets/Scripts/KeychainService.cs
using System.Runtime.InteropServices;

/// <summary>
///     実装は <c>Assets/Plugins/iOS/KeychainService.mm</c> に記載
/// </summary>
/// <remarks>
///     <a href="https://developer.apple.com/documentation/security/keychain_services">Keychain Services</a>
/// </remarks>
class KeychainService: ISecretManager
{
#if UNITY_IOS
    [DllImport("__Internal")]
    private static extern int addItem(string dataType, string value);

    [DllImport("__Internal")]
    private static extern string getItem(string dataType);

    [DllImport("__Internal")]
    private static extern int deleteItem(string dataType);
#endif
    
    // KeychainService.mm に定義した関数を呼び出す
    #region ISecretManager
    
    public override bool Put(string key, string value)
    {
#if UNITY_IOS
        return addItem(key, value) == 0;
#else
        return false;
#endif
    }

    public override string Get(string key)
    {
#if UNITY_IOS
        return getItem(key);
#else
        return null;
#endif
    }

    public override bool Delete(string key)
    {
#if UNITY_IOS
        return deleteItem(key) == 0;
#else
        return false;
#endif
    }
    
    #endregion
}

```

あとは上記をよしなに利用可能な `SecretManager` クラスを作成します。

```csharp:Assets/Scripts/SecretManager.cs
using UnityEngine;

/// <summary>
///     <em>Editor 利用時のみ PlayerPrefs を利用する</em>
/// </summary>
/// <remarks><see cref="KeychainService" />, <see cref="EncryptedSharedPreferences" /></remarks>
public static class SecretManager
{
#if UNITY_EDITOR
#elif UNITY_ANDROID
        private static ISecretManager _instance = new EncryptedSharedPreferences();
#elif UNITY_IOS
        private static ISecretManager _instance = new KeychainService();
#endif
        
        public static bool Put(string key, string value)
        {
#if UNITY_EDITOR
                PlayerPrefs.SetString(key, value);
                PlayerPrefs.Save();
                
                return true;
#elif UNITY_IOS || UNITY_ANDROID
                return _instance.Put(key, value);
#else
                Debug.Log("Not Implemented.");
                return false;
#endif

        }

        public static string Get(string key)
        {
#if UNITY_EDITOR
                return PlayerPrefs.GetString(key);
#elif UNITY_IOS || UNITY_ANDROID
        return _instance.Get(key);
#else
            Debug.Log("Not Implemented.");
            return null;
#endif
        }

        public static bool Delete(string key)
        {
#if UNITY_EDITOR
            PlayerPrefs.DeleteKey(key);
            PlayerPrefs.Save();

            return true;
#elif UNITY_IOS || UNITY_ANDROID
            return _instance.Delete(key);
#else
            Debug.Log("Not Implemented.");
            return false;
#endif
        }
}

```

これでプラットフォーム間の実装差異を気にすることなく、下記のような記述で設定値の保存や取得などを行えます。**iOS/Android 以外のプラットフォームで追加実装したい場合は [プラットフォーム依存コンパイル](https://docs.unity3d.com/ja/2021.1/Manual/PlatformDependentCompilation.html) と `ISecretManager` の実装クラスを新たに作成することで簡単に追加できます。**

```csharp
// ...
// name をキーとして値を nikaera で保存する
SecretManager.Put("name", "nikaera");

// name をキーとして値を取得する
var name = SecretManager.Get("name");

// "nikaera" が出力される
Debug.Log(name);

//　name をキーとして値を削除する
SecretManager.Delete("name");

// ...
```

# おわりに

今回は iOS/Android で設定値をセキュアに扱うための方法についてまとめてみました。実際は `Keychain Services` 周りは実装が大変なので、`External Dependency Manager for Unity` とか使って [KeychainAccess](https://github.com/kishikawakatsumi/KeychainAccess) のような外部ライブラリを利用する構成のほうが良いと思われます。

本記事の内容に誤りがあったり、実際にはセキュアな実装ができていない等々あれば是非コメントでご指摘いただけますと幸いです。

# 参考リンク

- [Android デベロッパー  \|  Android Developers](https://developer.android.com/topic/security/data?hl=ja)
- [EncryptedSharedPreferences  \|  Android デベロッパー  \|  Android Developers](https://developer.android.com/reference/androidx/security/crypto/EncryptedSharedPreferences?hl=ja)
- [Keychain Services \| Apple Developer Documentation](https://developer.apple.com/documentation/security/keychain_services)
- [SharedPreferencesを自前で難読化するのはもう古い?これからはEncryptedSharedPrefenrecesを使おう \- Qiita](https://qiita.com/masaki_shoji/items/6c512c7ebb30a13cda1d)
- [iOSのキーチェーンについて \- Qiita](https://qiita.com/sachiko-kame/items/261d42c57207e4b7002a)
- [UnityでIOSにセキュアに値を保存するにはKeyChainを使おう \- Qiita](https://qiita.com/nyhk-oi/items/189236d0627d43e7d658)
- [googlesamples/unity\-jar\-resolver: Unity plugin which resolves Android & iOS dependencies and performs version management](https://github.com/googlesamples/unity-jar-resolver)

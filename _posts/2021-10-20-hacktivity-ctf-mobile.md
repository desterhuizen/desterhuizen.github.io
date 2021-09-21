---
layout: post
title:  "H@cktivitycon 2021 - Mobile challenge writeup"
date:   2021-10-20 20:00:00 +0100
categories: security, ctf
---


Taking part in the H@ctivityCon CTF 2021 doing the two mobile challenges, I was able to recover the flag for two of the challenges.

# ToDo *(Easy)

## Download the todo.apk file

![alt text](/assets/ToDo_challenge.png)

## Verify the file type using file

```
file todo.apk
todo.apk: Zip archive data, at least v0.0 to extract
```

## Extract the apk content using apktool.

The apktool is a used to reverse engineer Android APKs, it can disassemble APK's and rebuild it into working APK's from the resources after changes are made.

```
apktool d todo.apk -o ./todo_apk_content                                                                                                                                                                                                                     130 ⨯
I: Using Apktool 2.5.0-dirty on todo.apk
I: Loading resource table...
I: Decoding AndroidManifest.xml with resources...
I: Loading resource table from file: /home/kali/.local/share/apktool/framework/1.apk
I: Regular manifest package...
I: Decoding file-resources...
I: Decoding values */* XMLs...
I: Baksmaling classes.dex...
I: Copying assets and libs...
I: Copying unknown files...
I: Copying original files...
I: Copying META-INF/services directory
```

## Disassemble the apk into a jar using dex2jar.

```
todo.apk -o todo_jar.jar
dex2jar todo.apk -> todo_jar.jar
Detail Error Information in File ./todo-error.zip
Please report this file to one of following link if possible (any one).
    https://sourceforge.net/p/dex2jar/tickets/
    https://bitbucket.org/pxb1988/dex2jar/issues
    https://github.com/pxb1988/dex2jar/issues
    dex2jar@googlegroups.com
```

## Find out what the main activity being used by the application is
```
cat todo_apk_content/AndroidManifest.xml
<?xml version="1.0" encoding="utf-8" standalone="no"?><manifest xmlns:android="http://schemas.android.com/apk/res/android" android:compileSdkVersion="30" android:compileSdkVersionCodename="11" package="com.congon4tor.todo" platformBuildVersionCode="30" platformBuildVersionName="11">
    <application android:allowBackup="true" android:appComponentFactory="androidx.core.app.CoreComponentFactory" android:extractNativeLibs="false" android:icon="@mipmap/ic_launcher" android:label="@string/app_name" android:roundIcon="@mipmap/ic_launcher_round" android:supportsRtl="true" android:theme="@style/Theme.ToDo">
        <activity android:exported="true" android:name="com.congon4tor.todo.LoginActivity">
            <intent-filter>
                <action android:name="android.intent.action.MAIN"/>
                <category android:name="android.intent.category.LAUNCHER"/>
            </intent-filter>
        </activity>
        <activity android:exported="true" android:name="com.congon4tor.todo.MainActivity"/>
    </application>
</manifest>
```

## Lets review the Todo and MyDatabase files in jadx-gui

![alt text](/assets/todo-jadx-gui.png)

We can see in line 10 of the MyDatabase class we have todos.db, lets see if we can find it.

## Review the content directory and see if we can find the db

```
 find . | grep todos.db                                                                                                                                                                                                                                         1 ⨯
./assets/databases/todos.db
```

Lets see what type of DB this is with file.

```
file ./assets/databases/todos.db
./assets/databases/todos.db: SQLite 3.x database, last written using SQLite version 3031001
```

## Check the SQLite DB if we can log in and read the database.
```
sqlite3 ./assets/databases/todos.db                                                                                                                                                                                                                          130 ⨯
SQLite version 3.34.1 2021-01-20 14:10:07
Enter ".help" for usage hints.
sqlite> .tables
todo
sqlite> SELECT * FROM todo;
1|AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=
2|BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB==
sqlite> .quit
```

These look promissing

## Decode the base64 
```
echo "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=" | base64 -d                                                                                                                                                                                         1 ⨯
flag{XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX}
```


```
 _______ 
( PWN!!! )
 ------- 
        o   ^__^
         o  (oo)\_______
            (__)\       )\/\
                ||----w |
                ||     ||
```


#  Reactor (Medium)

## Download and confirm the file type

![alt text](/assets/reactor-chanllenge.png)

```
file reactor.apk
reactor.apk: Zip archive data, at least v0.0 to extract
```

## Extract the content
```
apktool d reactor.apk -o reactor_apk_content
I: Using Apktool 2.5.0-dirty on reactor.apk
I: Loading resource table...
I: Decoding AndroidManifest.xml with resources...
I: Loading resource table from file: /home/kali/.local/share/apktool/framework/1.apk
I: Regular manifest package...
I: Decoding file-resources...
I: Decoding values */* XMLs...
I: Baksmaling classes.dex...
I: Copying assets and libs...
I: Copying unknown files...
I: Copying original files...
```

## Disassemble the APK into a JAR

```
d2j-dex2jar reactor.apk -o reactor_jar.jar
dex2jar reactor.apk -> reactor_jar.jar
```


## Lets have a look at the AndroidManifest to find the main Activity 

```
cat reactor_apk_content/AndroidManifest.xml
<?xml version="1.0" encoding="utf-8" standalone="no"?><manifest xmlns:android="http://schemas.android.com/apk/res/android" android:compileSdkVersion="30" android:compileSdkVersionCodename="11" package="com.reactor" platformBuildVersionCode="30" platformBuildVersionName="11">
    <uses-permission android:name="android.permission.INTERNET"/>
    <application android:allowBackup="false" android:appComponentFactory="androidx.core.app.CoreComponentFactory" android:icon="@mipmap/ic_launcher" android:label="@string/app_name" android:name="com.reactor.MainApplication" android:roundIcon="@mipmap/ic_launcher_round" android:theme="@style/AppTheme">
        <activity android:configChanges="keyboard|keyboardHidden|orientation|screenSize|uiMode" android:label="@string/app_name" android:launchMode="singleTask" android:name="com.reactor.MainActivity" android:windowSoftInputMode="adjustResize">
            <intent-filter>
                <action android:name="android.intent.action.MAIN"/>
                <category android:name="android.intent.category.LAUNCHER"/>
            </intent-filter>
        </activity>
    </application>
</manifest>
```

We are mainly interested in the com.reactor.MainActivity class and looking in there we only see and include to react.ReactActivity and a reference to facebook.react, it must be a reactNative application.

![alt text](/assets/reactor_main_activity.png)

Digging a little we see the MainApplication class which seem to be loading index.

![alt text](/assets/reactor_main_applicaiton.png)

## Lets see  if we can find index in the extracted APK content.

```
find reactor_apk_content | grep index
reactor_apk_content/assets/index.android.bundle
```

We find index.android.bundle, looing at this we see the javascript code, not very easy to read through.

```
cat reactor_apk_content/assets/index.android.bundle
var __BUNDLE_START_TIME__=this.nativePerformanceNow?nativePerformanceNow():Date.now(),__DEV__=false,process=this.process||{},__METRO_GLOBAL_PREFIX__='';process.env=process.env||{};process.env.NODE_ENV=process.env.NODE_ENV||"production";
!(function(r){"use strict";r.__r=o,r[__METRO_GLOBAL_PREFIX__+"__d"]=function(r,i,n){if(null!=e[i])return;var o={dependencyMap:n,factory:r,hasError:!1,importedAll:t,importedDefault:t,isInitialized:!1,publicModule:{exports:{}}};e[i]=o},r.__c=n,r.__registerSegment=function(r,t,i){s[r]=t,i&&i.forEach(function(t){e[t]||v.has(t)||v.set(t,r)})};var e=n(),t={},i={}.hasOwnProperty;function n(){return e=Object.create(null)}function o(r){var t=r,i=e[t];return i&&i.isInitialized?i.publicModule.exports:d(t,i)}function l(r){var i=r;if(e[i]&&e[i].importedDefault!==t)return e[i].importedDefault;var n=o(i),l=n&&n.__esModule?n.default:n;return e[i].importedDefault=l}function u(r){var n=r;if(e[n]&&e[n].importedAll!==t)return e[n].importedAll;var l,u=o(n);if(u&&u.__esModule)l=u;else{if(l={},u)for(var a in u)i.call(u,a)&&(l[a]=u[a]);l.default=u}return e[n].importedAll=l}o.importDefault=l,o.importAll=u;var a=!1;function d(e,t){if(!a&&r.ErrorUtils){var i;a=!0;try{i=h(e,t)}catch(e){r.ErrorUtils.reportFatalError(e)}return a=!1,i}return h(e,t)}var f=16,c=65535;function p(r){return{segmentId:r>>>f,localId:r&c}}o.unpackModuleId=p,o.packModuleId=function(r){return(r.segmentId<<gth>0){var n,a=null!==(n=v.get(t))&&void 0!==n?n:0,d=s[a];nul
    l!=d&&(d(t),i=e[t],v.delete(t))}var f=r.nativeRequire;if(!i&&f){var c=p(t),h=c.segmentId;f(c.localId,h),i=e[t]}if(!i)throw Error('Requiring unknown module "'+t+'".');if(i.hasError)throw _(t,i.error);i.isInitialized=!0;var m=i,g=m.factory,I=m.dependencyMap;try{var M=i.publicModule;return M.id=t,g(r,o,l,u,M,M.exports,I),i.factory=void 0,i.dependencyMap=void 0,M.exports}catch(r){throw i.hasError=!0,i.error=r,i.isInitialized=!1,i.publicModule.exports=void 0,r}}function _(r,e){return Error('Requiring module "'+r+'", which threw an exception: '+e)}})('undefined'!=typeof globalThis?globalThis:'undefined'!=typeof global?global:'undefined'!=typeof window?window:this);f)+r.localId};var s=[],v=new Map;function h(t,i){if(!i&&s.len
...
...
...
__r(0);
//# sourceMappingURL=index.android.bundle.map
```

Lets load it in to the browser and check if we get anything useful.
```
echo "<script src=\"reactor_apk_content/assets/index.android.bundle\"></script>" > index.html
python3 -m http.server

Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
172.16.92.1 - - [20/Sep/2021 15:09:22] "GET / HTTP/1.1" 200 -
172.16.92.1 - - [20/Sep/2021 15:09:22] "GET /reactor_apk_content/assets/index.android.bundle HTTP/1.1" 200 -
```

![alt text](/assets/reactor_chrome_data.png)

The code looks better, but still had to work with. If it was a simpler app it could be helpful.

## Decopile the react-native code using react-native-decompiler

We firrst have to install the packages and build the typescript project before we can use it. 
```
git clone https://github.com/richardfuca/react-native-decompiler
cd react-native-decompiler
npm i
npm run build
## Even if there were errors during compilation, it does still work
```

## Run the decompiler 
```
node ./out/main.js -i ~/hacktivitycon/reactor/reactor_apk_content/assets/index.android.bundle -o  ~/hacktivitycon/reactor/react_native_source/
Reading file...
Parsing JS...
Finding modules...
Took 3185.460638999939ms
Pre-parsing modules...
 ████████████████████████████████████████ 100% | ETA: 0s | 403/403
Took 1428.179178999737ms
Tagging...
 ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░ 0% | ETA: 25s | 1/403
Took 62.80381399951875ms
Filtering out modules only depended on ignored modules...
106 remain to be decompiled
Took 96.66759800165892ms
Decompiling...
 ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░ 0% | ETA: 82s | 1/106
Took 774.7864289991558ms
Generating code...
 ████████████████████████████████████████ 100% | ETA: 0s | 106/106
Took 8039.907232999802ms
Saving...
 ████████████████████████████████████████ 100% | ETA: 0s | 106/106
Writing to cache...
Took 496.88148299977183ms
Done!

cd ~/hacktivitycon/reactor/react_native_source/
ls -la
total 464
drwxr-xr-x 2 kali kali  4096 Sep 20 16:23 .
drwxr-xr-x 4 kali kali  4096 Sep 20 16:13 ..
-rw-r--r-- 1 kali kali   230 Sep 20 15:51 0.js
-rw-r--r-- 1 kali kali   432 Sep 20 15:51 10.js
....
-rw-r--r-- 1 kali kali   444 Sep 20 15:51 8.js
-rw-r--r-- 1 kali kali  1612 Sep 20 15:51 400.js

mv null.cache ../     # I move the null cache as it is the original data blob. It hides the results.
```

We get 108 js files, numbered.

## Next we Install the applicaiont on a test Virtual Android 

```
adb install reactor.apk
Performing Streamed Install
Success.
```

## Start the application on my test Android I see the input string of "Insert the pin to show the reactor codes"

![alt text](/assets/reactor_app.png)

## Letfs find that in the decompiled code 
```
grep -R "Insert the pin to show the reactor codes" react_native_source/*

react_native_source/399.js:      'Insert the pin to show the reactor codes.'
```


### Lest see what 399.js does 
```
cat react_native_source/399.js

  const module23 = require('@babel/runtime/helpers/interopRequireDefault')(require('./23'));
  const React = (function (t, n) {
  ....
  
  React.default.createElement(ReactNative.TextInput, {
      style: {
        height: 40,
        fontSize: 15,
        textAlign: 'center',
      },
      placeholder: 'PIN',
      keyboardType: 'number-pad',
      maxLength: 4,
      onChangeText(t) {
        return v(t);
      },
      onSubmitEditing(t) {
        c(require('./400').decrypt(t.nativeEvent.text));
        v('');
      },
      defaultValue: y,
    }),

    ....

exports.default = o;
```


We can see the code has a max length of 4 and we call ("./400.js").decrypt(code) to decrypt the key.

Lets have a look at 400.js
```
cat 400.js
const module401 = require('@babel/runtime/helpers/interopRequireDefault')(require('./401'));

const n = 'cccccccccccccccccccccccccccccccccccccccccccccccccccc';

function o(t, n) {
  for (var o = '', c = t; c.length < n.length; ) {
    c += c;
  }

  for (let f = 0; f < n.length; ++f) {
    o += String.fromCharCode(c.charCodeAt(f) ^ n.charCodeAt(f));
  }

  return o;
}

module.exports.encrypt = function (n, c) {
  return module401.default.encode(o(n, c));
};

module.exports.decrypt = function (c) {
  return o(c, module401.default.decode(n));
};
```
It creates and "o" Object that takes the input string, the decodes "n",  using the object.

With a little cleanup we can run the decrypt function locally

400.js
```
cp 400.js exploit.js
cat exploit.js

const module401 = require('./401'); 

const n = 'CCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCC';

function o(t, n) {
  for (var o = '', c = t; c.length < n.length; ) {
    c += c;
  }

  for (let f = 0; f < n.length; ++f) {
    o += String.fromCharCode(c.charCodeAt(f) ^ n.charCodeAt(f));
  }
  return o;
}

function decrypt (c) {
  return o(c, module401.default.decode(n));
};


for (let i = 0; i < 10000; i++) {
	value = decrypt(""+i);
	if (value.startsWith("flag{")) {
		console.log(value)
	}
}
```

This whould give us what we want...

```
node 400.js
flag{XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX}
```

```
 _____
< WIN >
 -----
        \   ^__^
         \  (oo)\_______
            (__)\       )\/\
                ||----w |
                ||     ||
```

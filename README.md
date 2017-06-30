# SecuritySample
Hiding encrypted secret API keys in C/C++ code and decrypting them with Java code.

Using SafetyNet Attestation API.

# Introduction

## Part1. Get encrypted data from native code through NDK

- Hiding Secret keys in C/C++ code.
- Using [RSAHelper] to encrypt your secret keys, you post those encrypted strings to this project. Then, fill in the parameters generated by [RSAHelper] in this project for decryption to decrypt the messages.

## Part2. Assess the security and compatibility of the Android environments in which your apps run
- Call SafetyNet Attestation API.

# Instruction Part1

## Step1. Generate a pair of RSA keys and encrypt your message.
Running [RSAHelper] to get encrypted messages, a pair of RSA modulus and exponent for decryption.

## Step2. Fill in MODULUS and EXPONENT
In [JNIHelper]

```Java
private final static String MODULUS = "Fill in the modulus created by RSAHelper";
private final static String EXPONENT = "Fill in the exponent created by RSAHelper";
```

## Step3. Add the decryption method to your project
```Java
public static String decryptRSA(String message) throws NoSuchAlgorithmException, NoSuchPaddingException,
        InvalidKeyException, IllegalBlockSizeException, BadPaddingException, UnsupportedEncodingException,
        InvalidAlgorithmParameterException, ClassNotFoundException, InvalidKeySpecException {
    Cipher c2 = Cipher.getInstance(Algorithm.rules.get("RSA")); // 创建一个Cipher对象，注意这里用的算法需要和Key的算法匹配

    BigInteger m = new BigInteger(Base64.decode(MODULUS.getBytes(), Base64.DEFAULT));
    BigInteger e = new BigInteger(Base64.decode(EXPONENT.getBytes(), Base64.DEFAULT));
    c2.init(Cipher.DECRYPT_MODE, convertStringToPublicKey(m, e)); // 设置Cipher为解密工作模式，需要把Key传进去
    byte[] decryptedData = c2.doFinal(Base64.decode(message.getBytes(), Base64.DEFAULT));
    return new String(decryptedData, Algorithm.CHARSET);
}

public static Key convertStringToPublicKey(BigInteger modulus, BigInteger exponent)
        throws ClassNotFoundException, NoSuchAlgorithmException, InvalidKeySpecException {
    byte[] modulusByteArry = modulus.toByteArray();
    byte[] exponentByteArry = exponent.toByteArray();

    // 由接收到的参数构造RSAPublicKeySpec对象
    RSAPublicKeySpec rsaPublicKeySpec = new RSAPublicKeySpec(new BigInteger(modulusByteArry),
            new BigInteger(exponentByteArry));
    // 根据RSAPublicKeySpec对象获取公钥对象
    KeyFactory kFactory = KeyFactory.getInstance(Algorithm.KEYPAIR_ALGORITHM);
    PublicKey publicKey = kFactory.generatePublic(rsaPublicKeySpec);
    // System.out.println("==>public key: " +
    // bytesToHexString(publicKey.getEncoded()));
    return publicKey;
}
```

## Step4. Create C/C++ files
- There are two ways to use JNI -- CmakeLists.txt and Android.mk, I used Android.mk here.
- Create jni folder in main/ .Then add [Android.mk], [Application.mk] and C/C++ files([Config.cpp]).

![JNI 1][NDK1]

- In build.gradle:

```gradle
externalNativeBuild {
    ndkBuild {
        path 'src/main/jni/Android.mk'
    }
}
```

- In MainActivity

```Java
static {
    //relate to LOCAL_MODULE in Android.mk
    System.loadLibrary("keys");
}
/**
 * A native method that is implemented by the 'native-lib' native library,
 * which is packaged with this application.
 */
public native String[] getAuthChain(String key);
```

## Step4. Run your app

```Java
    private final static String TAG = "MainActivity";

@Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        StringBuilder sb = new StringBuilder();
    try {
        // Example of a call to a native method
        TextView tv = (TextView) findViewById(R.id.sample_text);

        String[] authChain = getAuthChain("LOGIN");
        sb.append("Decrypted secret keys\n[ ");
        for (int i = 0; i < authChain.length; i++) {
            sb.append(decryptRSA(authChain[i]));
            sb.append(" ");
        }
        sb.append("]\n");

        String[] authChain2 = getAuthChain("OTHER");
        sb.append("secret keys\n[ ");
        for (int i = 0; i < authChain.length; i++) {
            sb.append(authChain2[i]);
            sb.append(" ");
        }
        sb.append("]");
        Log.d(TAG, sb.toString());
        tv.setText(sb.toString());
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```

# Instruction Part2
**Things you must know before you start developing.**
1. Use SafetyNetApi the deprecated class or you'd probably get 403 error by calling SafetyNet.getClient(context)
2. JWS (JSON Web Token) contains the signature and the body, your environment information is refer to the body (or the payload data).
3. There are two APIs you might need - SafetyNet API and Android device verification API. You get your device and app information with SafetyNet API, and check whether the information is truthful with another. Then let your server decide the next step (like shutting down the app or something).

## Step1. Get API keys from google develop console (optional)
- **You can skip this step if you don't verify your attestation response with google APIs. Or you can also validate the SSL certificate chain by yourself. Google highly recommends you to check your JWS statement.**
- **What "Android Device Verification API" dose is only checking the JWS signature, its response has nothing to do with the Android environments in which your apps run. (The JSON payload)**
- General API key here: https://console.developers.google.com/, and don't forget to add and enable "Android Device Verification API".
- Make sure the API key you post to "Android Device Verification API" is unrestricted.
- There are daily quotas of "Android Device Verification API".

- In gradle.porpeties, add your google API key
```
safetynet_api_key = XXXXXXXXX
```

- In build.gradle
```gradle
android {
  defaultConfig {
          buildConfigField("String", "API_KEY", "\"${safetynet_api_key}\"")
      }
}

```

- DO NOT add any safetyNet meta-data in your manifest
```xml
<!--<meta-data-->
    <!--android:name="com.google.android.safetynet.ATTEST_API_KEY"-->
    <!--android:value="${safetynet_api_key}" />-->
```


## Step2. Build GoogleApiClient and call SafetyNet APIs
```java
SafetyNetHelper safetyNetHelper = new SafetyNetHelper(BuildConfig.API_KEY);
GoogleApiClient googleApiClient = new GoogleApiClient.Builder(contex)
        .addApi(SafetyNet.API)
        .addConnectionCallbacks(googleApiConnectionCallbacks)
        .addOnConnectionFailedListener(googleApiConnectionFailedListener)
        .build();
//Don't forget to connect!
googleApiClient.connect();
```

```Java
byte[] requestNonce = generateOneTimeRequestNonce();
Log.d(TAG, "Nonce:" + Base64.encodeToString(requestNonce, Base64.DEFAULT));
SafetyNet.SafetyNetApi.attest(googleApiClient, requestNonce)
        .setResultCallback(new ResultCallback<SafetyNetApi.AttestationResult>() {

            @Override
            public void onResult(@NonNull SafetyNetApi.AttestationResult attestationResult) {
                Status status = attestationResult.getStatus();
                boolean isSuccess = status.isSuccess();
                if (!isSuccess)
                    callback.onFail(ErrorMessage.SAFETY_NET_API_NOT_WORK, ErrorMessage.SAFETY_NET_API_NOT_WORK.name());
                else {
                    try {
                        final String jwsResult = attestationResult.getJwsResult();
                        final AttestationResult response = new AttestationResult(jwsResult);
                        if (!verifyJWSResponse) {
                            callback.onResponse(response.getFormattedString());
                        } else {
                            if (API_KEY == null)
                                callback.onFail(ErrorMessage.NULL_API_KEY, ErrorMessage.NULL_API_KEY.name());
                            else {
                                AndroidDeviceVerifier androidDeviceVerifier = new AndroidDeviceVerifier(API_KEY, jwsResult);
                                androidDeviceVerifier.verify(new AndroidDeviceVerifier.AndroidDeviceVerifierCallback() {
                                    @Override
                                    public void error(String errorMsg) {
                                        callback.onFail(ErrorMessage.FAILED_TO_CALL_GOOGLE_API_SERVICES, ErrorMessage.FAILED_TO_CALL_GOOGLE_API_SERVICES.name() + ":" + errorMsg);
                                    }

                                    @Override
                                    public void success(boolean isValidSignature) {
                                        if (isValidSignature)
                                            callback.onResponse("isValidSignature true\n\n" + response.getFormattedString());
                                        else
                                            callback.onFail(ErrorMessage.ERROR_VALID_SIGNATURE, ErrorMessage.ERROR_VALID_SIGNATURE.name());
                                    }
                                });
                            }
                        }
                    } catch (JSONException e) {
                        callback.onFail(ErrorMessage.EXCEPTION, e.getMessage());

                    }
                }
            }
        });
```

- [SafetyNetUtils]

## Step3. Call Attestation API to retrieve JWS message
The JWS payloads I got by running this app on the real device and the nox monitor are a little different.

- On my mobile phone, ctsProfileMatch and basicIntegrity were both true.
```JSON
{
  "nonce":"pUkGirEXYOQefux33VWeSEmR0kBkLNGQaiQiZvE3VAc=",
  "timestampMs":1498814112718,"apkPackageName":"com.catherine.securitysample",
  "apkDigestSha256":"FPgrs1x05EaZiJkfKaitzEXTazg+GDDqYtbR5XyJiJE=",
  "ctsProfileMatch":true,
  "extension":"CbRP9k08+pZE",
  "apkCertificateDigestSha256":["9mLFS3eHWOBcHlA4MmODmfGvzgkbg2YSQ2z/ww9lCfw="],
  "basicIntegrity":true
}

```

- On nox monitor, ctsProfileMatch and basicIntegrity were both false.
```JSON
{
  "nonce":"/Md+iYxxW14bSMFFdN6rosLSINPTIvuhv12GJMgeP08=",
  "timestampMs":1498814181775,
  "apkPackageName":"com.catherine.securitysample",
  "apkDigestSha256":"+9Ql8SnUCxQfOVE8abByEJs9zPsxAaHiQs0nH0bwRps=",
  "ctsProfileMatch":false,
  "extension":"CcmOud4/walT",
  "apkCertificateDigestSha256":["9mLFS3eHWOBcHlA4MmODmfGvzgkbg2YSQ2z/ww9lCfw="],
  "basicIntegrity":false
}
```

## Step4. Verify JWS response (optional)
- Finish Step1 first.
- **You can skip this step if you don't verify your attestation response with google APIs. Or you can also validate the SSL certificate chain by yourself. Google highly recommends you to check your JWS statement.**
- **What "Android Device Verification API" dose is only checking the JWS signature, its response has nothing to do with the Android environments in which your apps run. (The JSON payload)**
- [AndroidDeviceVerifier]

## Step5. Back to your application
- Post the JWS payload to your server, let server tells your app what's next.

If you want to read more about google security services for Android, you can watch [Google Security Services for Android : Mobile Protections at Google Scale], the youtube video. Or you could see my note [README_cn], they are almost the same.

# Warnings
When you add new secret keys, you must refill modulus, exponent and the other encrypted keys, because you get different RSA KeyPair(private key and public key) every execution.

# License

```
Copyright 2017 Catherine Chen (https://github.com/Catherine22)

Licensed under the Apache License, Version 2.0 (the "License"); you may not
use this file except in compliance with the License. You may obtain a copy of
the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS, WITHOUT
WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the
License for the specific language governing permissions and limitations under
the License.
```

[RSAHelper]:<https://github.com/Catherine22/RSAHelper>
[MainActivity]: <https://github.com/Catherine22/SecuritySample/blob/master/app/src/main/java/com/catherine/securitysample/MainActivity.java>
[JNIHelper]: <https://github.com/Catherine22/SecuritySample/blob/master/app/src/main/java/com/catherine/securitysample/JNIHelper.java>
[SafetyNet]: <https://github.com/Catherine22/SecuritySample/blob/master/app/src/main/java/com/catherine/securitysample/SafetyNet>
[Android.mk]:<https://github.com/Catherine22/SecuritySample/blob/master/app/src/main/jni/Android.mk>     
[Application.mk]:<https://github.com/Catherine22/SecuritySample/blob/master/app/src/main/jni/Application.mk>
[NDK1]: https://github.com/Catherine22/MobileManager/blob/master/jni1.png  
[Google Security Services for Android : Mobile Protections at Google Scale]:<https://www.youtube.com/watch?v=exU1f_UBXGk>
[README_cn]:<https://github.com/Catherine22/SecuritySample/blob/master/README_cn.md>     
[AndroidDeviceVerifier]: <https://github.com/Catherine22/SecuritySample/blob/master/app/src/main/java/com/catherine/securitysample/SafetyNet/AndroidDeviceVerifier.java>
[SafetyNetUtils]: <https://github.com/Catherine22/SecuritySample/blob/master/app/src/main/java/com/catherine/securitysample/SafetyNet/SafetyNetUtils.java>

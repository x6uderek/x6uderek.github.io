---
layout: default
title: "eclipse接入google In-App-Billing和firebase"
date: 2017-01-20 03:13:00
categories: blog
tags: ["Android", "google play", "firebase", "eclipse"]
description: "eclipse接入google In-App-Billing和firebase"
---


google官方文档现在只介绍在Android studio里面用gradle集成IAB和firebase，如何使用eclipse集成还需要自己探索。

## 一、准备google-service-lib项目
1. 在android-sdk\extras\google\m2repository\com\google\android\gms目录找到play-services-base和play-services-basement，选择一个版本，最好是9.0以上（因为firebase最低是9.0）。
2. 将其中的aar包解压，然后把jar复制出来，重命名为google-service-base.jar和google-service-basement.jar。这两个jar放到google-service-lib项目的lib下面。
3. 将res文件夹合并，作为google-service-lib的资源文件夹。注意这里合并步骤，先把color和drawable手动合并，遇到冲突，手动解决。然后下面的values也手动合并，而values的多语言文件夹可以选几个合并，没必要一个个来，太多了。

## 二、接入In-App-Billing
### 1、使用google官方demo里面的IabHelper，将android-sdk\extras\google\play_billing\samples\TrivialDrive \src\com\example\android\trivialdrivesample\util下面的工具类复制出来，注意更换包名。
### 2、首先是连接google play查询库存。
```java
mIabHelper = new IabHelper(context,LoginUtil.getInstance().gp_key);
mIabHelper.enableDebugLogging(true);
final String[] finalProductIds = productIds;//需要查询详细信息的商品id数组
mIabHelper.startSetup(new IabHelper.OnIabSetupFinishedListener() {
	@Override
	public void onIabSetupFinished(IabResult result) {
		if (!result.isSuccess()) {
			PayUtil.this.isConnected = false;
			Log.e(LoginUtil.TAG, result.toString());
		} else {
			PayUtil.this.isConnected = true;
		}
		if (PayUtil.this.mIabHelper == null) {
			return;
		}
		Log.d(LoginUtil.TAG, "Setup successful. Querying inventory.");
		if (PayUtil.this.isConnected) {
			try {
				PayUtil.this.mIabHelper.flagEndAsync();
			} catch (Exception e) {
				e.printStackTrace();
			}
			PayUtil.this.mIabHelper.queryInventoryAsync(true, Arrays.asList(finalProductIds),mGotInventoryListener);
		} else {
			DialogUtil.showToast(context, "com_popoapp_sdk_gp_init_fail");
		}
	}
});
```
### 3、库存查询回调
```java
private IabHelper.QueryInventoryFinishedListener mGotInventoryListener = new IabHelper.QueryInventoryFinishedListener() {
	@Override
	public void onQueryInventoryFinished(IabResult result, Inventory inv) {
		Log.d(LoginUtil.TAG,"Query inventory finished.");
		if (PayUtil.this.mIabHelper == null) {
			return;
		}
		if (result.isFailure()) {
			return;
		}
		PayUtil.this.mSkuMap.putAll(inv.getSkuDetails());
		List purchases = inv.getAllPurchases();
		if(purchases.isEmpty()){
			Log.d(LoginUtil.TAG, "Purchases are empty");
		}
		List toConsume = new ArrayList();
		for(Purchase purchase:purchases){
			//自动重发逻辑
			if(purchase.getItemType().equals(IabHelper.ITEM_TYPE_INAPP)){
				if(PreferenceUtil.increase(context, purchase.getOrderId())<=MAX_RETRY) {
					Log.d(LoginUtil.TAG,"Complete purchase:"+purchase);
					completeOrder(purchase);
				}
				else {
					//超过重发次数则设置消耗
					toConsume.add(purchase);
				}
			}
		}
		if(toConsume.size()>0) {
			mIabHelper.consumeAsync(toConsume, consumeMultiFinishedListener);
		}
		if(initCallback!=null){
			initCallback.onGetProductDetails(inv.getSkuDetails());
		}
	}
};
```
### 4、消耗回调，这里最好把消耗记录落地到本地数据存储，便于查单
```java
private IabHelper.OnConsumeFinishedListener consumeFinishedListener = new IabHelper.OnConsumeFinishedListener() {
	@Override
	public void onConsumeFinished(Purchase purchase, IabResult result) {
		if(result.isSuccess()){
			Log.d(LoginUtil.TAG, "Consume success:" + purchase);
		}
		else {
			Log.e(LoginUtil.TAG,"Consume fail:"+purchase);
		}
	}
};

private IabHelper.OnConsumeMultiFinishedListener consumeMultiFinishedListener = new IabHelper.OnConsumeMultiFinishedListener() {
	@Override
	public void onConsumeMultiFinished(List purchases, List results) {
		for (int i=0; i<results.size(); i++) {
			IabResult result = results.get(i);
			Purchase purchase = purchases.get(i);
			if (result.isSuccess()) {
				Log.d(LoginUtil.TAG, "Consume success:" + purchase);
			} else {
				Log.e(LoginUtil.TAG, "Consume fail:" + purchase);
			}
		}
	}
};
```
### 5、发起支付，payload字段最好传自己的订单号
```java
mIabHelper.flagEndAsync();
mIabHelper.launchPurchaseFlow((Activity)context,payArgs.productid,PURCHASE_REQUEST_CODE,purchaseFinishedListener,orderid+","+payArgs.pc_id);
```
### 6、接收支付结果
```java
public void onActivityResult(int requestCode, int resultCode, Intent data){
    if(mIabHelper==null){
       return;
    }
    mIabHelper.handleActivityResult(requestCode,resultCode,data);
}
```
### 7、支付回调
```java
private IabHelper.OnIabPurchaseFinishedListener purchaseFinishedListener = new IabHelper.OnIabPurchaseFinishedListener() {
	@Override
	public void onIabPurchaseFinished(IabResult result, final Purchase info) {
		if(result.isSuccess()){
			Log.d(LoginUtil.TAG, "Purchase success,consuming...");

			SkuDetails details = PayUtil.this.mSkuMap.get(info.getSku());
			if(details!=null) {
				float price = details.getFloatPrice();
				//af event
				Map<String, Object> eventValue = new HashMap<String, Object>();
				eventValue.put(AFInAppEventParameterName.REVENUE, price);
				eventValue.put(AFInAppEventParameterName.CURRENCY, details.getCurrency());
				AppsFlyerLib.getInstance().trackEvent(context, AFInAppEventType.PURCHASE, eventValue);
				//fb event
				AppEventsLogger logger = AppEventsLogger.newLogger(context);
				logger.logPurchase(BigDecimal.valueOf((double) price), Currency.getInstance(details.getCurrency()));
			}

			//showLoading("com_popoapp_sdk_delivering");
			completeOrder(info);
		}
		else {
			Log.e(LoginUtil.TAG, "Purchase fail:" + result + ",Info:" + info);
			if(payCallback!=null){
				if(result.getResponse()==IabHelper.BILLING_RESPONSE_RESULT_USER_CANCELED) {
					payCallback.onPayCancel();;
				}
				else {
					payCallback.onPayFail();
				}
			}
		}
	}
};
```
### 8、将订单信息发回app服务器
```java
private void completeOrder(final Purchase info) {
	final Handler handler = new Handler();
	new Thread(){
		public void run(){
			try {
				final String[] parts = info.getDeveloperPayload().split(",");
				final JSONObject obj = GraphApi.completeOrder(parts[1], parts[0], info.getOriginalJson(), info.getSignature());
				final int ret = obj.getInt("ret");
				final String msg = obj.getString("msg");
				handler.post(new Runnable() {
					@Override
					public void run() {
						if(ret==0 || ret==11005) {
							mIabHelper.consumeAsync(info, consumeFinishedListener);
						}
						if(ret==0) {
							if(payCallback!=null) {
								payCallback.onPaySuccess(info.getSku());
							}
						}
						else {
							DialogUtil.showStringToast(context,msg);
							//hideLoading();
							if(payCallback!=null){
								payCallback.onPayFail();
							}
						}
					}
				});
			} catch (JSONException e) {
				handler.post(new Runnable() {
					@Override
					public void run() {
						//hideLoading();
						DialogUtil.showToast(context,"com_popoapp_sdk_delivery_fail");
						if(payCallback!=null){
							payCallback.onPayFail();
						}
					}
				});
			}
		}
	}.start();
}
```
### 9、断开连接
```java
public void onDestroy() {
	context = null;
	isConnected = false;
	if (this.mIabHelper != null) {
		this.mIabHelper.dispose();
		this.mIabHelper = null;
	}
}
```
### 10、服务端验证（php代码）

1. 安装google api sdk:`composer require "google/apiclient ^2.0"`
2. 在google后台创建服务账号，并授予财务权限，下载服务账号的json文件
3. 验证代码
```php
$purchase_data = json_decode($request['inapp_purchase_data'], true);
//获取包的base64 key
$packageInfo = $this->game_service->getPackageInfo($order_info['gid'], $purchase_data['packageName']);
//ssl初步验证
if (function_exists('openssl_verify')) {
	$key = "-----BEGIN PUBLIC KEY-----\n" . chunk_split($packageInfo['gp_key'], 64, "\n") . "-----END PUBLIC KEY-----";
	$key = openssl_pkey_get_public($key);
	if (openssl_verify($request['inapp_purchase_data'], base64_decode($request['inapp_data_signature']), $key) != 1) {
		throw new Exception(11010);
	}
} else {
	throw new Exception(11011);
}
//google api验证
putenv('GOOGLE_APPLICATION_CREDENTIALS='.realpath('GP-serveraccount.json'));
$client = new Google_Client();
$client->useApplicationDefaultCredentials();
$client->addScope(Google_Service_AndroidPublisher::ANDROIDPUBLISHER);
$publisher = new Google_Service_AndroidPublisher($client);
$publisher->purchases_products->get($purchase_data['packageName'], $purchase_data['productId'], $purchase_data['purchaseToken']);
if ($result->getPurchaseState() != 0) {
    throw new Exception(11006);
}
```
## 三、接入firebase

### 1、在`android-sdk\extras\google\m2repository\com\google\firebase`找到firebase-common、firebase-message和firebase-iid。这3个是实现推送功能的最小集合，firebase-analytics、firebase-analytics-impl是firebase统计功能，firebase-crash是崩溃信息统计，这些都是可选的。

### 2、选择一个版本（必须与上面选的google-service版本一致），将aar解压出来，将jar复制到我们的lib添加依赖。firebase没有资源文件夹需要合并，但是里面有很多`AndroidManifest.xml`需要配置，这里总结如下（**包名占位符需要替换**）：

首先是permission
```xml
<uses-permission android:name="com.android.vending.BILLING" />
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
<uses-permission android:name="android.permission.ACCESS_WIFI_STATE" />
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.WAKE_LOCK"/>
<permission android:name="包名.permission.C2D_MESSAGE" android:protectionLevel="signature"/>
<uses-permission android:name="包名.permission.C2D_MESSAGE" />
<uses-permission android:name="com.google.android.c2dm.permission.RECEIVE" />
```
然后receiver，provider，service
```xml
<receiver
    android:name="com.google.firebase.iid.FirebaseInstanceIdReceiver"
    android:exported="true"
    android:permission="com.google.android.c2dm.permission.SEND" >
    <intent-filter>
        <action android:name="com.google.android.c2dm.intent.RECEIVE" />
        <action android:name="com.google.android.c2dm.intent.REGISTRATION" />
        <category android:name="包名" />
    </intent-filter>
</receiver>
<provider android:authorities="包名.firebaseinitprovider"
	android:name="com.google.firebase.provider.FirebaseInitProvider"
	android:exported="false" android:initOrder="100"/>
<service android:name="com.popoapp.api.gcm.PopoMessagingService">
    <intent-filter>
        <action android:name="com.google.firebase.MESSAGING_EVENT" />
    </intent-filter>
</service>
<service android:name="com.popoapp.api.gcm.PopoInstanceIDService">
    <intent-filter>
        <action android:name="com.google.firebase.INSTANCE_ID_EVENT" />
    </intent-filter>
</service>
<receiver android:name="com.google.firebase.iid.FirebaseInstanceIdInternalReceiver" android:exported="false" />
<service android:name="com.popoapp.api.gcm.RegistrationIntentService" android:exported="false"/>
<receiver android:name="com.appsflyer.MultipleInstallBroadcastReceiver" android:exported="true">
    <intent-filter>
        <action android:name="com.android.vending.INSTALL_REFERRER" />
    </intent-filter>
</receiver>
<receiver android:name="com.google.android.gms.measurement.AppMeasurementReceiver" android:enabled="true" android:exported="false">
</receiver>
<receiver android:name="com.google.android.gms.measurement.AppMeasurementInstallReferrerReceiver" android:permission="android.permission.INSTALL_PACKAGES" android:enabled="true">
	<intent-filter>
		<action android:name="com.android.vending.INSTALL_REFERRER"/>
	</intent-filter>
</receiver>
<service android:name="com.google.android.gms.measurement.AppMeasurementService" android:enabled="true" android:exported="false"/>
<meta-data
    android:name="com.google.android.gms.version"
    android:value="@integer/google_play_services_version" />
```
### 3、代码集成

在`res/values/strings.xml`建立字符串资源
```xml
<string name="default_web_client_id" translatable="false">{YOUR_CLIENT}/oauth_client/[first client_type 3]</string>
<string name="gcm_defaultSenderId"   translatable="false">project_info/project_number</string>
<string name="firebase_database_url" translatable="false">project_info/firebase_url</string>
<string name="google_app_id"         translatable="false">{YOUR_CLIENT}/client_info/mobilesdk_app_id</string>
<string name="google_api_key"        translatable="false">{YOUR_CLIENT}/services/api_key/current_key</string>
<string name="google_storage_bucket" translatable="false">project_info/storage_bucket</string>
```
这些字符串的值来源于从firebase下载的json文件，参考下面的参考资料[Firebase Doc][1]

有3个类需要实现

注册firebase token的类
```java
public class RegistrationIntentService extends IntentService {
    public RegistrationIntentService() {
        super("com.popoapp.api.gcm.RegistrationIntentService");
    }

    @Override
    protected void onHandleIntent(Intent intent) {
        try {
            if(LoginUtil.getInstance().is_init==2) {
                String token = FirebaseInstanceId.getInstance().getToken();
                Log.d("fcm token", token+"");
                subscribeTopics();
                //TODO send token to app server
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    private void subscribeTopics() throws IOException {
        FirebaseMessaging.getInstance().subscribeToTopic("global");
    }
}
```
刷新token的类
```java
public class PopoInstanceIDService extends FirebaseInstanceIdService {
    public void onTokenRefresh() {
        Intent intent = new Intent(this, RegistrationIntentService.class);
        startService(intent);
    }
}
```
显示通知的类
```java
public class PopoMessagingService extends FirebaseMessagingService {
    private static final int NOTIFICATION_ID = 1001;

    public void onMessageReceived(RemoteMessage msg) {
        String from = msg.getFrom();
        Log.d("from", from);
        if(msg.getData().size() > 0) {
            String title = msg.getData().get("title");
            String body = msg.getData().get("body");
            sendNotification(title, body);
        }
        else if(msg.getNotification()!=null) {
            String title = msg.getNotification().getTitle();
            String body = msg.getNotification().getBody();
            sendNotification(title, body);
        }
    }

    private void sendNotification(String title, String message) {
        NotificationUitl.sendNotification(this, NOTIFICATION_ID, title, message);
    }
}
```
在app启动的时候调用token注册服务
```java
if(GoogleApiAvailability.getInstance().isGooglePlayServicesAvailable(context)==0) {
    Intent intent = new Intent(context, RegistrationIntentService.class);
    context.startService(intent);
}
```
## 四、服务端发送通知
```php
$send_data = [
	"to" => "/topics/global",
	"notification" => [
		"body" => $send_msg,
		"title" => $send_title,
	],
	"restricted_package_name" => $pkg_name
];
$send_url = 'https://fcm.googleapis.com/fcm/send';
$ch = curl_init();
curl_setopt($ch, CURLOPT_URL, $send_url);
curl_setopt($ch, CURLOPT_HEADER, false);
curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($send_data));
curl_setopt($ch, CURLOPT_HTTPHEADER, ['content-Type:application/json', 'Authorization:key=' . $server_key]);
$res = curl_exec($ch);
```
参考资料：
1. [The Google Services Gradle Plugin | Firebase][1]
2. [Dan Dar3: Eclipse: Integrate Firebase Analytics into your Android app][2]

[1]: <https://firebase.google.com/docs/reference/gradle/#processing_the_json_file>
[2]: <http://dandar3.blogspot.tw/2016/11/eclipse-integrate-firebase-analytics.html>
# Checkout (Android In-App Billing Library)

## Description

<img src="https://github.com/serso/android-checkout/blob/master/app/misc/res/logo256x256.png" align="right" />

**Checkout** is an implementation of [Android In-App Billing API(v3+)](http://developer.android.com/google/play/billing/api.html).
Its main goal is to make integration of in-app products as simple and
straightforward as possible: developers should not spend much time on
implementing boring In-App Billing API but should focus on more important
things - their apps. With this in mind, the library was designed to be
fast, flexible and secure.

**Current version:** 0.7.5

### Why?

**Checkout** solves common problems that developers face while working 
with billing on Android, such as:
- How to cancel all billing requests when Activity is destroyed?
- How to query purchase information in the background?
  See also [Querying for Items Available for Purchase](http://developer.android.com/google/play/billing/billing_integrate.html#QueryDetails)
- How to verify a purchase?
  See also [Security And Design](http://developer.android.com/google/play/billing/billing_best_practices.html)
- How to load all the purchases using ```continuationToken``` or SKU
  details (one request is limited by 20 items)?
- How to add billing with a minimum amount of boilerplate code?

**Checkout** can be used with any dependency injection framework or
without it. It has a clear distinction of a functionality available in
different contexts: purchase can be done only from ```Activity``` while
SKU information can be loaded in ```Service``` or ```Application```. 
Moreover, it has a good test coverage and is continuously build on Travis
CI:  [![Build Status](https://travis-ci.org/serso/android-checkout.svg)](https://travis-ci.org/serso/android-checkout)

## Getting started

### Setup

- Gradle/Android Studio in ```build.gradle```:
```groovy
compile 'org.solovyev.android:checkout:x.x.x@aar'
```
- Maven in ```pom.xml```:
```xml
<dependency>
    <groupId>org.solovyev.android</groupId>
    <artifactId>checkout</artifactId>
    <version>x.x.x</version>
    <type>apklib</type>
</dependency>
```
- Download sources from github and either copy them to your project or import them as a project dependency
- Download artifacts from the [repository](https://oss.sonatype.org/content/repositories/releases/org/solovyev/android/checkout/)

In-app billing requires `com.android.vending.BILLING` permission to be
set in the app. This permission is automatically added to your app's
```AndroidManifest.xml``` by Gradle. You can declare this permission
explicitly by adding the following line to the ```AndroidManifest.xml```:  
```xml
<uses-permission android:name="com.android.vending.BILLING" />
```

### Example

Say there is an app that contains one in-app product with "sku_01" id.
Then ```Application``` class might look like this:
```java
public class MyApplication extends Application {

    private final Billing mBilling = new Billing(this, new Billing.DefaultConfiguration() {
        @Override
        public String getPublicKey() {
            return "Your public key, don't forget abput encryption";
        }
    });

    public Billing getBilling() {
        return mBilling;
    }
}
```
And ```Activity``` class like this:
```java
public class MyActivity extends Activity {

    private final ActivityCheckout mCheckout = Checkout.forActivity(this, MyApplication.get().getBilling());

    private Inventory mInventory;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        mCheckout.start();

        mCheckout.createPurchaseFlow(new PurchaseListener());

        mInventory = checkout.makeInventory();
        mInventory.load(Inventory.Request.create()
                            .loadAllPurchases()
                            .loadSkus(IN_APP, IN_APPS),
                            new InventoryCallback()))
    }
    
    @Override
    public void onClick(Veiw v) {
        mCheckout.whenReady(new Checkout.EmptyListener() {
            @Override
            public void onReady(BillingRequests requests) {
                requests.purchase("sku_01", null, checkout.getPurchaseFlow());
            }
        });
    }

    @Override
    protected void onActivityResult(int requestCode, int resultCode, Intent data) {
        super.onActivityResult(requestCode, resultCode, data);
        mCheckout.onActivityResult(requestCode, resultCode, data);
    }

    @Override
    protected void onDestroy() {
        mCheckout.stop();
        super.onDestroy();
    }
}
``` 

## Advanced usage

### Samples

Sample app is available on [Google Play](https://play.google.com/store/apps/details?id=org.solovyev.android.checkout.app)
and its sources on [gitbug](https://github.com/serso/android-checkout/tree/master/app).

### Building from sources

**Checkout** is built by Gradle. The project structure and build procedure
are standard for Android libraries. An environmental variable ANDROID_HOME
must be set before building and should point to Android SDK installation
folder (f.e. /opt/android/sdk).
Please refer to [Gradle User Guide](http://tools.android.com/tech-docs/new-build-system/user-guide) for more information about the building.

### Classes overview

**Checkout** contains three main classes that represent different levels
of abstractions: [Billing](https://github.com/serso/android-checkout/blob/master/lib/src/main/java/org/solovyev/android/checkout/Billing.java), [Checkout](https://github.com/serso/android-checkout/blob/master/lib/src/main/java/org/solovyev/android/checkout/Checkout.java) and [Inventory](https://github.com/serso/android-checkout/blob/master/lib/src/main/java/org/solovyev/android/checkout/Inventory.java).

**Billing** is a core class of **Checkout**'s implementation of the
billing API. It is responsible for:
- connecting and disconnecting to the [billing service](https://github.com/serso/android-checkout/blob/master/lib/src/main/aidl/com/android/vending/billing/IInAppBillingService.aidl)
- performing [billing requests](https://github.com/serso/android-checkout/blob/master/lib/src/main/java/org/solovyev/android/checkout/BillingRequests.java)
- caching the requests results
- creating [Checkout](https://github.com/serso/android-checkout/blob/master/lib/src/main/java/org/solovyev/android/checkout/Checkout.java) objects
- logging

Only one instance of **Billing** should be used in the app in order to
avoid multiple connections to the billing service. Though this class
might be used directly usually it's better to work with [Checkout](https://github.com/serso/android-checkout/blob/master/lib/src/main/java/org/solovyev/android/checkout/Checkout.java)
instead.

**Checkout** is a middle tier of the library. It uses **Billing** in a
certain context, e.g. in ```Application```, ```Activity``` or ```Service```,
checks whether billing is supported and executes the requests. [ActivityCheckout](https://github.com/serso/android-checkout/blob/master/lib/src/main/java/org/solovyev/android/checkout/ActivityCheckout.java)
is a subclass capable of purchasing items.

**Inventory** loads information about products, SKUs and purchases. Its
lifecycle is bound to the lifecycle of **Checkout** in which it was created.

### Purchase verification

By default, **Checkout** uses simple purchase verification algorithm (see
[DefaultPurchaseVerifier](https://github.com/serso/android-checkout/blob/master/lib/src/main/java/org/solovyev/android/checkout/DefaultPurchaseVerifier.java)). As explained in [Android documentation](http://developer.android.com/google/play/billing/billing_best_practices.html#sign)
it's better to verify purchases on a remote server. **Checkout** allows
you to provide your own [PurchaseVerifier](https://github.com/serso/android-checkout/blob/master/lib/src/main/java/org/solovyev/android/checkout/PurchaseVerifier.java) via ```Billing.Configuration#getPurchaseVerifier```.
[BasePurchaseVerifier](https://github.com/serso/android-checkout/blob/master/lib/src/main/java/org/solovyev/android/checkout/BasePurchaseVerifier.java)  can be used as a base class for purchase verifiers that
should be executed on a background thread.

### Proguard

Library's proguard rules are automatically added to the app project by
Gradle. You can declare them explicitly by copying the contents of [proguard-rules.txt](https://github.com/serso/android-checkout/blob/master/lib/proguard-rules.txt)
to your proguard configuration.

## License

Copyright 2016 serso aka se.solovyev

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.

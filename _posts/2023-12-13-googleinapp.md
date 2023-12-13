---
title: Google Play Billing (server-side)
date: 2023-12-13
categories: [Server, Google Play Billing]
tags: [google play developer api, pub/sub]   # TAG names should always be lowercase
pin: true
---

## Google Play API & RTDN
***
### Intro 
![](/assets/img/googleplay.png)
**Google Cloud Platform & Google Play Console**
1. Create a Project and connect it to Google Play Console.
2. Create a Service Account and download the JSON key in GCP.
   - This key is required when making authenticated calls to the Google Play Developer API.
3. Enable the Google Play Android Developer API
4. Set the necessary permissions in the Google Play Console to allow the service account to view financial data and manage orders.

**RTDN (RealTime Developer Notification)**
1. Create a Pub/Sub topic in Google Cloud Console. (Publish pub/sub)
2. Implement the server to receive and handle the RTDN message. (Subscribe pub/sub)

### Setup
[Getting ready](https://developer.android.com/google/play/billing/getting-ready)
1. Create an Account of Developer
- Publish the App in Google Play Console
2. Create and manage subscriptions
- [Play Console Help](https://support.google.com/googleplay/android-developer/answer/140504?hl=en&;ref_topic=3452890)
3. Set the Google Play Developer API
- [Getting Started](https://developers.google.com/android-publisher/getting_started)
4. Set the RTDN
- [RTDN](https://developer.android.com/google/play/billing/getting-ready)
5. Implement the server
6. Test

<br>

## Implement
***
### Verifying the receipt
```mermaid
sequenceDiagram;
    participant App
    participant Server
    participant Google
    participant DB
        App->>Google: request purchase
        Google->>App: return Purchase
        App->>Server: VerifyGoogleReceiptandJoin(reqBody ReqGoogleReceiptandJoin) string
        Server->>Google: Purchases.Subscriptions.Acknowledge(packageName, token, subscriptionId)
        Google->>Server: return
        Server->>Google: Purchases.Subscriptions.Get(packageName, token, subscriptionId)
        Google->>Server: return SubscriptionPurchase
        Server->>Server: if SubscriptionPurchase.PaymentState == 1
        Server->>DB: insert into member
        DB->>Server: return
        Server->>DB: insert into google_receipt
        DB->>Server: return
        Server->>App: return 
  ```
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
1. Create an account and publish the app. [Getting ready](https://developer.android.com/google/play/billing/getting-ready)
2. Create and manage subscriptions. [Play Console Help](https://support.google.com/googleplay/android-developer/answer/140504?hl=en&;ref_topic=3452890)
3. Set the Google Play Developer API. [Getting Started](https://developers.google.com/android-publisher/getting_started)
4. Set the RTDN. [RTDN](https://developer.android.com/google/play/billing/getting-ready)
5. Implement the server.
6. Test

<br>

## Implement
***
```mermaid
sequenceDiagram
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
![](/assets/img/googleinappSequence.png)

[Purchase](https://developer.android.com/reference/com/android/billingclient/api/Purchase) |
[SubscriptionPurchase](https://developers.google.com/android-publisher/api-ref/rest/v3/purchases.subscriptions)

<br>

### Access to Google Play API
```go

type Factory struct {
   ...
     GoogleAPI       *androidpublisher.Service
     PackageName     string
     SubscriptionID  string
}

func (_this *Factory) AccessToGoogleAPI() *Factory {
   
   // Store the key securely!!! My code is just an example to show how it's working briefly.
   GoogleAPIKey = "/my/secret/google-api/jsonkey.json"
   ctx := context.Background()
   jsonKey, err := ioutil.ReadFile(GoogleAPIKey)
   if err != nil {
     // handle the error
   } 
   
   // Create the JWT config instance
   config, err := google.JWTConfigFromJSON(jsonKey, androidpublisher.AndroidpublisherScope)
   if err != nil {
     // handle the error
   } 
   
   // Create the authenticated HTTP client
   // - Got the access token from JWT config
   client := config.Client(ctx)
   
   // Create the Android Publisher service instance
   // - It offer methods to be able to communicate with Google Plau API
   GoogleAPI, err := androidpublisher.NewService(ctx, option.WithHTTPClient(client))
   if err != nil {
     // handle the error
   } 
   
   _this.GoogleAPI = GoogleAPI   
   
   return _this
}

```

[google package](https://pkg.go.dev/golang.org/x/oauth2/google#JWTConfigFromJSON) |
[androidpublisher package](https://pkg.go.dev/google.golang.org/api/androidpublisher/v2#Service) 

<br>

### Verify the receipt
```go
func (_this *HttpService) verifyGoogleReceipt(reqBody formats.ReqGoogleRequest) error {
	
	err := AckSub(_this.Fac, reqBody.PurchaseToken)
	if err != nil {
	  // handle the error
	}

	sub, err := GetSub(_this.Fac, reqBody.PurchaseToken)
	if err != nil {
	  // handle the error
	}

	if sub.AcknowledgementState != 1 {
	  // handle the error
	}

	if sub.PaymentState == nil {
	  // handle the error
	}

	switch *sub.PaymentState {
	case 0:
	  // handle the error
	case 1:
	  fmt.Printf("Payment is received.")
	default:
	  // handle the error
	}
	
	/** Save the receipt **/
	...

}
```

```go

func GetSub(Fac *factory.Factory, token string) (androidpublisher.SubscriptionPurchase, error) {

	sub := androidpublisher.SubscriptionPurchase{}

	call := Fac.GoogleAPI.Purchases.Subscriptions.Get(Fac.PackageName, Fac.SubscriptionID, token)
	response, err := call.Do()
	if err != nil {
	  // handle the error
	}
	
	// parse the response

	return sub, nil
}
```

<br>

### RTDN 
```go
func ReceiveRTDN(fac *factory.Factory, handler *Handler) {
	fmt.Println("rtdn connected")
	ctx := context.Background()

	projectID := "secret"
	subscriptionID := "secret"

	os.Setenv("GOOGLE_APPLICATION_CREDENTIALS", GoogleAPIKey)

	client, err := pubsub.NewClient(ctx, projectID)
	if err != nil {
	  fmt.Printf("Failed to create client: %v\n", err)
	  return
	}

	sub := client.Subscription(subscriptionID)

	err = sub.Receive(ctx, func(ctx context.Context, msg *pubsub.Message) {

		devNotif := playstore.DeveloperNotification{}

		err := json.Unmarshal(msg.Data, &devNotif)
		fmt.Printf("Parsed Notification: %+v\n", devNotif)
		if err != nil {
			msg.Nack()
			fmt.Printf("Failed to unmarshal message: %v\n", err)
			return
		}

		notif := devNotif.SubscriptionNotification
		fmt.Printf("Parsed Notification: %+v\n", notif)
		
		handler.HandleNotification(devNotif.SubscriptionNotification)
		msg.Ack()
	})
	if err != nil {
		fmt.Printf("Failed to receive message: %v\n", err)
	}
}
```
```go
func (_this *Handler) HandleNotification(notif playstore.SubscriptionNotification) {

	token := notif.PurchaseToken
	notiType := notif.NotificationType

	switch notiType {
	case playstore.SubscriptionNotificationTypeRenewed:	
	case playstore.SubscriptionNotificationTypeCanceled:	
	case playstore.SubscriptionNotificationTypeRestarted:	
	case playstore.SubscriptionNotificationTypeGracePeriod:	
	case playstore.SubscriptionNotificationTypeAccountHold:	
	case playstore.SubscriptionNotificationTypeRecovered:	
	case playstore.SubscriptionNotificationTypePaused:	
	case playstore.SubscriptionNotificationTypePauseScheduleChanged:	
	case playstore.SubscriptionNotificationTypeDeferred:
	case playstore.SubscriptionNotificationTypeRevoked:
	case playstore.SubscriptionNotificationTypeExpired:
	}

	return
}

```
[playstore package](https://pkg.go.dev/github.com/awa/go-iap/playstore#New)|
[pubsub package](https://pkg.go.dev/cloud.google.com/go/pubsub)
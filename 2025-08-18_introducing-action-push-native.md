# Introducing Action Push Native

**Author:** Jacopo Beschi, Programmer, SIP
**Published:** August 18, 2025
**Source:** <https://dev.37signals.com/introducing-action-push-native/>

*A Rails gem for sending push notifications to mobile platforms.*

---

**Note:** Shortly after releasing this gem, we renamed it from *Action Native Push* to *Action Push Native*, in case you arrived here looking for the gem of the former name. [More details here](https://github.com/basecamp/action_push_native/pull/28).

We’ve open-sourced [Action Push Native](https://github.com/basecamp/action_push_native), a Rails gem for sending push notifications to mobile platforms. It supports both Apple and Google push notification services.

---

## Why did we build it?

We created it to migrate off Amazon SNS and Pinpoint, as part of our broader [cloud exit](https://basecamp.com/cloud-exit). We’re using it in [Basecamp](https://basecamp.com/) and [HEY](https://www.hey.com/) to send more than 10 million push notifications per day without a hitch.

Action Push Native relies on HTTP/2 persistent connections to the Apple Push Notification service, which significantly reduced job duration compared to our previous HTTP/1 setup with AWS Pinpoint:

![AWS Pinpoint jobs duration](https://dev.37signals.com/assets/images/introducing-action-push-native/pinpoint-performance.png)

AWS Pinpoint jobs duration

![Action Push Native jobs duration](https://dev.37signals.com/assets/images/introducing-action-push-native/action-push-native-performance.png)

Action Push Native jobs duration

---

## How does it work?

The gem connects directly to the Apple (APNs) and Google (FCM) push notification services. It handles retries, rate-limiting, and deleting dead devices automatically. Configure each platform with your credentials, and you can start sending notifications like this:

```
device = ApplicationPushDevice.create! \
  name: "iPhone 16",
  token: "6c267f26b173cd9595ae2f6702b1ab560371a60e7c8a9e27419bd0fa4a42e58f",
  platform: "apple"

notification = ApplicationPushNotification.new \
  title: "Hello world!",
  body:  "Welcome to Action Push Native"

notification.deliver_later_to(device)
```

[Version 0.1.0](https://rubygems.org/gems/action_push_native) is available now. You can read more on [GitHub](https://github.com/basecamp/action_push_native#readme).

We hope you find it useful!

# A Friday email incident

**Author:** Jorge Manrubia, Principal Programmer, Product
**Published:** May 29, 2023
**Source:** <https://dev.37signals.com/a-friday-email-incident/>

*Because only talking about success stories can be boring, here’s one about a Friday incident that happened while we worked on our cloud departure.*

---

On Friday, May 19th, we received a customer report about emails from other HEY users landing in spam. 37signals uses a two-tier On-call system for support tickets that require technical assistance, and Joan Stewart – the Support team manager who handled the customer ticket – created a new card in the board for Programmer’s On-call in Basecamp:

![](https://dev.37signals.com/assets/images/a-friday-email-incident/initial-on-call-ticket.png)

Initial On-call ticket

Title
[HEY] Emails from BC and HEY marked as spam | May 19

Column
Done May 20

Added by
Joan S. on May 19

**Case URL:** https://secure.helpscout.net/conversation/(redacted)

**Issue:** Customer found emails from his wife (also a HEY personal user) and support@basecamp.com in his Spam.

HEY email URLs:
https://app.hey.com/topics/(redacted)
https://app.hey.com/topics/(redacted)
https://app.hey.com/topics/(redacted)

The ticket shows an iOS screenshot of an email flagged as Spam. Its subject is “New sign in for Basecamp from Chrome in Mac”.

Knowing this looked like a serious issue, Joan also posted a message in the campfire of the SIP team (Security, Infrastructure, and Performance):

![](https://dev.37signals.com/assets/images/a-friday-email-incident/campfire-notice.png)

Discussion in the SIP Campfire

Joan Stewart 11:26pm
Not sure if anyone is available to take a look, but it looks like emails from hey.com are getting marked as spam by HEY itself. I have two customer reports and was able to replicate myself :(. *[Link to the On-call ticket]*

Jorge 11:45pm
hmm those emails seem to be failing DKIM verification.

Jorge 11:45pm

```
#<MailAnalysis::Result:0x00007f7979e4f760
 id: 32363207,
 insights: [SPAM (forged_sender.failed_dkim_verification). allowed_by_dmarc=false, valid_dkim_signature=false. [weight=1.0]],
 analysis_subject_id: 1055086826,
 analysis_subject_type: "Receipt",
 created_at: Fri, 19 May 2023 21:06:05.570162000 UTC +00:00,
 updated_at: Fri, 19 May 2023 21:06:05.570162000 UTC +00:00>
```

Jorge 11:45pm
Matt could this be related to changes with the mail pods with the on-prem move?

Every 37signals programmer participates in weekly On-call rotations of two people during regular working hours. We also have Ops On-call shifts covering 24×7. In this case, no programmer was paged because I saw the message by chance and, seeing the severity, jumped in.

I could reproduce the problem myself. Our analysis system indicated that these emails were failing to validate their [DKIM](https://en.wikipedia.org/wiki/DomainKeys_Identified_Mail) signatures, a standard to validate that emails originate in the domain declared by the senders.

An important consideration is that we can’t access email contents such as the subject, body, or attachments unless customers give explicit consent. And this is not just a guideline; we built a [new encryption technology](https://world.hey.com/jorge/a-story-of-rails-encryption-ce104b67) and set a periodic audit process to enforce it. This is not an issue in most cases – such as this one – because we only need to check non-sensitive headers and metadata.

I looped in Matt Kent, Principal SRE leading the [HEY transition from the cloud to our datacenters](https://world.hey.com/dhh/why-we-re-leaving-the-cloud-654b47e0). We had recently moved our inbound postfix servers to on-prem, and this looked almost certainly related to that. Matt wasn’t at home in that moment. After some initial discussion, I decided to go with a first mitigation for the problem and to raise a critical incident within the company. We do that by posting a message in our “Incidents” Basecamp, where we use [custom categories](https://3.basecamp-help.com/article/46-message-board#message-categories) to capture the incident severity and status:

![](https://dev.37signals.com/assets/images/a-friday-email-incident/internal-incident-notice.png)

Internal message informing about the incident

Title
[HEY] Emails between HEY users landing in spam

What
We received three reports from customers about emails received from HEY senders ending in spam.

The issue
We are failing to validate the DKIM signature in these messages and, thus, we’re marking them as spam. We still don’t know why this is happening.

Alerted by
Joan. On-call card.

Timeline
- 2023-05-19 9:26pm Joan raises the alarm in SIP chat about the problem, which also reported to On-call.
- 2023-05-19 Matt, Jesse and Jorge discuss the problem in the SIP campfire.
- 2023-05-19 Jorge prepares a temporary change to disable DKIM for @hey senders.
- 2023-05-20 0:27am Matt reverts the inbound email changes and fixes the issue. He also reverts the temporary DKIM disablement.
- 2023-05-20 6.33am Jorge ran script to reprocess the affected email and remove the spam flag
- 2023-05-20 8:35am Jorge ran script to notify affected HEY users about what happened.
- 2023-05-20 7:52pm Jorge ran script to notify affected HEY users about problem with basecamp.com emails too.

Remediation
- As a first remediation, we’ll disable DKIM validation for HEY senders. This is far from ideal, but it’s a temporary measure to stop the bleeding while we understand what’s going on.
- Prepare script to “ham” affected email.
- Revert this PR once it’s fixed.

The mitigation consisted in disabling DKIM checks for marking emails as spam in the app. The change in the HEY source looked like this:

```
+++ b/app/models/mail_analysis/inbound_receipt/rules/forged_sender/dkim_verification.rb
@@ -1,7 +1,7 @@
 class MailAnalysis::InboundReceipt::Rules::ForgedSender::DkimVerification < MailAnalysis::InboundReceipt::Rules::ForgedSender::Base
   private
     def analyze_forged_sender
-      if !subject.autoreply_from_us? && subject.domain_requires_verification_via_dkim? && !sender_verified_via_dkim?
+      if !subject.autoreply_from_us? && subject.domain_requires_verification_via_dkim? && !sender_verified_via_dkim? && !hey_sender?
         create_forged_sender_insight code: "forged_sender.failed_dkim_verification", details: insight_details
       end
     end
@@ -13,4 +13,8 @@ class MailAnalysis::InboundReceipt::Rules::ForgedSender::DkimVerification < Mail
     def insight_details
       { allowed_by_dmarc: mail.allowed_by_dmarc_policy?, valid_dkim_signature: mail.valid_dkim_signature? }
     end
+
+    def hey_sender?
+      Array(mail.from).any? { |email_address| email_address =~ /@hey.com$/ }
+    end
 end
```

This was a temporary bandage to stop the bleeding and, indeed, was live for less than one hour. When Matt returned home, he reverted the on-prem move for the email servers and the DKIM disablement for internal email.

This solved the problem for customers, but we still had to fix the affected emails and notify customers about what happened. We considered this a data-loss scenario, as customers would likely miss emails landing in SPAM. I prepared a couple of scripts you can check below:
- [One, to mark the affected emails as ham](https://gist.github.com/jorgemanrubia/5f0ee2d1561e2dcd112887e91873b4d2#file-process_affected_emails_hey-rb)
- [Another one, to notify the affected customers about what happened and link the affected emails](https://gist.github.com/jorgemanrubia/5f0ee2d1561e2dcd112887e91873b4d2#file-notification_email_hey-rb)

The email we sent looked like this:

![](https://dev.37signals.com/assets/images/a-friday-email-incident/hey-notification-email.png)

The email we sent to customers

Hi Jorge,

A recent change resulted in HEY flagging emails from other HEY users as spam. Unfortunately, this problem affected these emails in your account:
- https://app.hey.com/topics/(redacted)

We have already fixed the problem and removed the spam flag from all the affected emails.

We are really sorry that this problem impacted you. Please feel free to reply to this email if you need any further clarification or help with anything else.

The HEY team

The incident affected 2,319 emails in 2,002 accounts, and we sent as many emails. Almost 900 affected emails were internal notices from within the app (such as sign-in from another device notice).

After clearing this, we declared the incident over, but we still had to figure out what happened to continue with the on-prem move.

The investigation continued in a todo as part of the Basecamp project to bring HEY home:

![](https://dev.37signals.com/assets/images/a-friday-email-incident/investigation-todo.png)

Todo to continue investigation

Title
rspamd - don’t skip local delivery

Assigned to
Paul

Per “Re: 🚨[HEY] Emails between HEY users landing in spam - Incidents” we’re seeing local delivery now from our mail relays -> home/work-mx, as traffic doesn’t leave the F5s. See the ip_address here:

Image showing results in Kibana highlighting an IP which is obfuscated.

rspamd (per 'rspamadm configdump') is skipping the scanning on these emails due to 'local_addrs'

```
options {
    control_socket = "/var/lib/rspamd/rspamd.sock mode=0600";

    local_addrs [
        (redacted list of local ip addresses)
    ]
```

which is likely set this way on all 3 instances.

It seems like the easiest mode here is to just remove that, but I’m not clear on the implications for all mail flowing through the system.

Senior SRE Paul Shuvashish first noticed that these emails weren’t failing DKIM but [SPF](https://en.wikipedia.org/wiki/Sender_Policy_Framework). HEY uses [Postfix](https://www.postfix.org) for both inbound and outbound emails. For each email it ingests, Postfix validates DKIM and SPF, and it adds some headers to the email with the results before handling the email to the Rails monolith via [Action Mailbox](https://guides.rubyonrails.org/action_mailbox_basics.html).

This pointed out a flaw in our application-level analysis system: we were assimilating [DMARC](https://en.wikipedia.org/wiki/DMARC) errors – which can be either because of SPF or DKIM – to DKIM errors. So while the app was doing the right thing nevertheless – marking the email as spam – the insight it was collecting internally was misleading.

What happened with the on-prem move is that inbound emails from HEY senders stopped hitting the public IP we were declaring in our SPF record. This is a summary of the issue that Matt shared in that todo:

> The current issue is that this header:

```
Received: from (redacted relay host) (redacted host) [redacted IP])
  by (redacted host) (Postfix) with ESMTP id 7D5A182DB8
  for <some@email.com>; Wed, 24 May 2023 01:55:10 +0000 (UTC)
```

> results in us failing the SPF check in rspamd as our policy is “v=spf1 ip4:204.62.114.0/23 -all” which won’t match the internal load balancer IP we’re seeing.

Matt and Paul explored different alternatives to handle this at the Postfix level, which was quite challenging. We obviously didn’t want to allow internal IPs as valid sources for SPF, since this would open the door to bypass the SPF checks by an attacker.

In the end, Eron Nicholson – director of Ops – came up with a solution at the network level by implementing a custom SNAT rule in our provider [F5](https://www.f5.com), so that the mail servers saw a public IP for outbound traffic in our datacenter as the source for these emails, even if the connection stayed internal in the datacenter. This is a fragment of the comment with the resolution posted in the same todo where the entire investigation happened:

> … That means that the mailservers see the private IP of the F5s, which is causing these DKIM/SPF issues. However, that doesn’t need to be the case. I’ve created this new iRule :

```
# hey_snat_inside_mail_irule
# SNAT inside mail requests to an outside IP for DKIM/SPF
#
# Covers our 10.x/20 - IP::remote_addr is the source IP and LB::server addr is the pool member address in this context.
when LB_SELECTED {
  if { [IP::addr [IP::remote_addr] equals [LB::server addr]/20] } {
    snat (redacted IP)
    set hsl [HSL::open -proto UDP -pool filebeat_pool]
    HSL::send $hsl "<168> hey_snat_inside_mail_irule [IP::remote_addr] -> [LB::server addr] snat override"
  }
}
```

> … which uses a specific, outside SNAT address. This one in particular is the default source IP for outbound traffic in Ashburn. It really doesn’t matter as long as it’s an outside IP in our range that the F5 owns. So now the mailservers see the source of these connections as coming from the outside in our IP range, even though the connection really stays internal in the datacenter.

This fixed the issue for good and let us continue moving HEY to our datacenters.

So far, our effort to move apps off the cloud has been very smooth considering its magnitude, but this doesn’t mean it’s been totally free of issues. Success stories normally omit the bumps along the road, but reality rarely works like that.

This was also a good opportunity to dissect what happens inside 37signals from the moment a customer reports a serious issue until we fix it, and how Basecamp helped to coordinate five different people on three teams to do it.

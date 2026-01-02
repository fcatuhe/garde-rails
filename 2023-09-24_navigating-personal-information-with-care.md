# Navigating personal information with care

**Author:** Jorge Manrubia, Principal Programmer, Product
**Published:** September 24, 2023
**Source:** <https://dev.37signals.com/navigating-personal-information-with-care/>

*Our default for accessing customer information is: we don’t.*

---

Accessing personal information from customers is a serious matter. With the launch of HEY in 2020, we developed some technology and processes to support a very simple principle: employees shouldn’t have access — intentionally or unintentionally — to personal information from our users without their explicit consent. In this post, I’ll show what this looks like in practice.

---

## The programmer side

A few weeks ago, I was On-call and worked on a tricky HEY support ticket that involved email aliases. After some back and forth, I realized I needed access to some email addresses we stored in the database. Those are encrypted since they are personal information, so I reached out to the customer to ask for access permission:

> My name is Jorge. I am a programmer in the HEY team. Thanks for your help troubleshooting this one so far. I have one last request: could you please share the link to that test email showing the problem you linked. Also I need your consent to get decrypted access to that email (and only that email). We store everything encrypted, and this would be very helpful for understanding why we aren’t setting the from header correctly.
> Thanks!

Once the customer replied with affirmative consent, I started a Rails console session to investigate the issue. When starting the session, our system asks for the reason. If it’s an On-call case, there is usually an associated [Card Table](https://basecamp.com/features/card-table) card, so we enter a link to it:

![](https://dev.37signals.com/assets/images/navigating-personal-information-with-care/console-1.png)

Justify the console session

A Rails console prompt:

You have access to production data here. That’s a big deal. As part of our promise to keep customer data safe and private, we audit the commands you type here. Let’s get started!

Commands:
- decrypt!: enter unprotected mode with access to encrypted information

jorge, why are you using this console today?

> Link to the card table card with the On-call case

By default, in console sessions, personal information is encrypted. This is fine for most problems, and it prevents programmers from seeing personal information unintentionally while troubleshooting. For example, this is what I would see right now if I printed the last Todo created in Basecamp 4:

![](https://dev.37signals.com/assets/images/navigating-personal-information-with-care/console-2.png)

Show the last todo title encrypted

bc3> Todo.last.content

#=> Unintelligible ciphertext.

Back to the HEY case, with the explicit customer consent received, I started a decrypted session in the console. It asked me for a link justifying the consent, for which I entered the link to the message from the customer in Help Scout.

![](https://dev.37signals.com/assets/images/navigating-personal-information-with-care/console-3.png)

Get access to encrypted content by justifying consent

A Rails console prompt:

> decrypt!

Ok! You have access to encrypted information now. We pay extra close attention to any commands entered while you have this access. You can go back to protected mode with ‘encrypt!’

WARNING: Make sure you don’t save objects that were loaded while in protected mode, as this can result in saving the encrypted texts.

Before you can access personal information, you need to ask for and get explicit consent from the user(s). jorge, where can we find this consent (a URL would be great)?

> A link to the consent by the customer in Help Scout

---

## The console auditor side

All the commands we enter in production Rails consoles are stored and audited later. When those happen in a decrypted session, we flag them as sensitive and receive special attention.

At 37signals, the members of the SIP team (Security, Infrastructure, and Performance) take shifts to audit console sessions. In this case, Donal was in charge of doing the audit. This is what he saw in our auditing tool where he validated my console  session:

![](https://dev.37signals.com/assets/images/navigating-personal-information-with-care/console-audit-screen.png)

Auditing the session

Screenshot from our auditing tool showing:
- “Jorge” as the name of the console session.
- “Donal McBreen as the name of the console auditor.
- A link to the On-call card justifying the console session.
- A link to the customer consent in Help Scout.
- The list of commands entered in the console session.

There, Donal could check that I accessed only the relevant email I got consent for and review the commands I ran. Then, he approved the session using the same tool.

![](https://dev.37signals.com/assets/images/navigating-personal-information-with-care/console-audit-consent.png)

Approval of the console session

Prompt showing options to approve or flag sessions as inappropriate, along with entering comments.

Every two weeks, console auditors summarize how console audits went in a Basecamp project called SIP: Console user audits. This is [the one that Donal wrote](https://public.3.basecamp.com/p/L3MjDx5bUYhDHQHUNVVhgw3N). My session appears in the group of three HEY sessions that accessed decrypted data during that period.

![](https://dev.37signals.com/assets/images/navigating-personal-information-with-care/report-in-basecamp.png)

While the process is quite streamlined now, these summaries have served to drive the development of internal tooling to reduce the need for decrypted sessions.

---

## Everyone can do this

We have released all the technology we developed to support this workflow. If you run Rails, you can use the very same tools we do. They are free and open source:
- [The database encryption system](https://guides.rubyonrails.org/active_record_encryption.html), which is part of Rails 7.
- [console1984](https://github.com/basecamp/console1984), to protect Rails console sessions and record the commands entered.
- [audits1984](https://github.com/basecamp/audits1984), the auditing tool we use to audit console sessions.
- [mass_encryption](https://github.com/basecamp/mass_encryption), to encrypt data in mass. We used this tool to [encrypt billions of records in Basecamp 4](https://updates.37signals.com/post/new-new-home-screen-the-lineup-doors-and-more).

Everyone agrees that passwords or secret access tokens require special attention and should never appear in logs or be easily accessible by anyone. We believe that personal information deserves the same consideration. And so does the private content users create using your software.

Without your consent, we can’t see the message you wrote in Basecamp when you report a bug about it. And this is a big relief for us: it’s none of our business.

I hope this article encourages you to embrace a similar approach. The tools we released answer the technical needs, so you only need organizational commitment. I guarantee it won’t take long before you start feeling that the previous way of working was off.

# Homographic Spoofing: a new<br /> Ruby toolkit

**Author:** Jacopo Beschi, Programmer, SIP
**Published:** June 25, 2024
**Source:** <https://dev.37signals.com/homographic-spoofing/>

---

![](https://dev.37signals.com/assets/images/homographic-spoofing/email.jpg)

## What is an homograph attack

Homograph characters look the same or very similar to other characters, but are different. For example, the letters “l” and “ӏ” (Cyrrilic “ӏ”) look the same but are different characters.
A homograph attack is a phishing attack where the attacker uses homograph characters to spoof another identity. Homograph attacks became a real security concern after the introduction of [Internationalized Domain Names (IDNs)](https://en.wikipedia.org/wiki/Internationalized_domain_name), which allowed domain names to contain Unicode characters.
For example, a malicious actor could send you a fake reset password email from “support@paypaӏ.com” (note the Cyrrilic “ӏ”) with a link to their phishing website “https://paypaӏ.com” that looks like PayPal (and has a valid SSL certificate) stealing your credentials.

---

## How to protect from homograph attacks

The most common way to protect from homograph attacks, implemented by modern browsers and most email clients, is to convert Unicode characters into [Punycode](https://en.wikipedia.org/wiki/Punycode): a representation of Unicode with the limited ASCII character subset used for Internet hostnames. But converting **all** international domains using non-ASCII Unicode characters to Punycode would be gross because it’d make legitimate domains look like gibberish: Punycode conversion should occur only when there is a real security threat. To help with this, the Unicode Technical Standard #39 ([UTS #39](https://www.unicode.org/reports/tr39/)) provides guidelines to detect security issues with Unicode character usage.

---

## State of the art in Ruby

In [HEY](https://app.hey.com/), our email service, [we had to include a mechanism to protect our users from homograph attacks](https://updates.37signals.com/post/new-in-hey-homographic-phishing-protection). We didn’t find any Ruby implementations for UTS #39, so we had to implement our own.
Recently, we extracted it [to an open-source Gem](https://github.com/basecamp/homographic_spoofing) so you can all benefit from it. Our implementation is complete and in line with [UTS #39](https://www.unicode.org/reports/tr39/). You can use it to protect both IDNs and email addresses. We hope you find it useful!

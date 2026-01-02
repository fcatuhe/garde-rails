# 2023 Summer Intern Program:<br /> Who, what and how

**Author:** Andrea LaRowe, Head of People Ops
**Published:** September 15, 2023
**Source:** <https://dev.37signals.com/2023-summer-interns/>

---

This summer, three interns joined us at the beginning of June, and they ended their work earlier this month. The program was a resounding success, and I’d like to share a little about it.

---

## How we set it up

The process began with posting the openings early in the year. Out of 2,000 applications, we chose three interns for programming and ops, and knew from the start that we needed to get some big things right about how we asked all of them to work. This included a few guiding principles:
- The work they did over summer would be real and impactful — for them and for their teams. No coffee fetching here.
- Our interns would be well paid, and treated much like full time employees (participating in all-hands and team calls, completing check-ins to provide visibility into their daily work).
- We would provide our interns with a couple different facets of professional support.

Our interns came to us with interest in their respective areas and some academic or hobby experience, but no experience working in professional teams with quality and deadline expectations. Our teams put great care into determining the specific work that needed to be done weeks before our interns started. We chose projects they could realistically complete in 3 months, that would also be challenging to them and valuable to the business overall.

All of our interns were onboarded quickly, so they could get to “real work” right away — they were all contributing members of their teams and shipping work by week 2.

Finally, we stood up a couple different pillars of support for our interns. Our People team coached them through the initial interview process more than we would a seasoned candidate. We provided each intern with a mentor on their immediate team, who was available to help work through issues and provide feedback on PRs or technical approach. Interns also had a manager who checked in on their relationship with their mentor, overall experience with the company, and their large projects. They also had a 37signals Buddy — someone not on their immediate team who they could approach with questions or feedback about the culture of the company.

---

## Our interns and their work

### Yusuf Birader *Leicester, United Kingdom*

Yusuf took on a few big projects; improving the speed of Basecamp exports, adding productivity boosts to customer support tooling, and improving internal tooling for spam response in HEY.

In his work on spam wave response, Yusuf tackled the speed of spam response. HEY employs Rspamd rulesets which were previously hardcoded in configuration files and required updates to go through our build/deployment pipelines. Employing Rspamd multimaps Yusuf updated the configuration to be pulled from our internal support tooling, allowing dynamic and on-the-fly updates to be picked up instantly.

Yusuf also enhanced tooling around how we identify spam accounts abusing the HEY service, leaning on heuristics pulled from internal and third party systems (hCaptcha) to list and flag suspicious accounts in a simple to use support interface. Previously, this work would have to be performed by a programmer on the console.

![Fighting Spam Waves in HEY](https://dev.37signals.com/assets/images/2023-summer-interns/rspamd.png)

Yusuf’s mentor Jacopo said that he completed work *“autonomously with very little intervention. He improved the quality of work after each interaction, and additionally, he crafted helpful documentation of all he’s done.”* The work Yusuf delivered in just a matter of weeks was impactful, already paying dividends for our team & products.

**Here’s what Yusuf had to say about his internship:**

> From the very beginning, 37signals’ rapid onboarding process, ethos of trust, and seamless remote work left a lasting impression on me. I was treated as a full-time member of the team from the beginning, entrusted with substantial tasks, and encouraged to take on ambitious projects. At 37signals, there’s no speed limit.
> Reflecting on my internship, I find myself motivated by the challenges and experiences I encountered during my time there. I’ve already ventured into action by creating pzip, a concurrent zip archiver that promises to improve archiving speed by 10x. This project serves as a concrete example of how my internship has empowered me to tackle real-world problems.

---

### Jordan Coil *Vancouver, Canada*

Jordan’s primary project was Basecamp’s new shared public folders feature. The feature addressed a frequent pain point of customers. But just as important was the analysis Jordan did on the technical and real-world implications of public sharing. As with any feature that deals with making customer data public great care and attention has to be taken to avoid unexpected changes in privacy. To address this Jordan introduced a new public folder specific controller and tweaked the way we publish the items they contain. This meant the we could easily scope the public URLs, with each requested item determining its published state based on its parents visibility, nested or otherwise.

![Public Links Announcement](https://dev.37signals.com/assets/images/2023-summer-interns/public-links.png)

Jordan’s mentor, Pratik, shared that he *“tackled tasks and accepted feedback like a seasoned team member, helping us deliver several long-pending HEY and Basecamp features.”* Jordan shipped 5 Basecamp mini-features during his first month alone and delivered with the same care and quality expected of any programmer at 37signals.

**Here’s what Jordan had to say about his internship:**

> My experience at 37signals was everything I could have asked for and more. Right from the very beginning, everyone was incredibly supportive. Public links for Basecamp folders was a big undertaking that challenged me, forcing me to learn about the product, hone my programming skills and keep myself accountable as a manager of one.
> In the end, the sense of accomplishment from taking a project from start to finish made it all worth it. Getting to see the positive response from customers was the cherry on top that made this internship experience one of a kind.

---

### Arman Jindal *New York, USA*

Arman came into our interview process prepared with insightful question about how we approach our infrastructure – which ultimately led to the [great blog post he published](https://dev.37signals.com/leaning-imperative/) recently.

Arman’s most substantial project was organizing, sharpening, and improving the read-, search- and write-ablility of our critical ops docs. His work highlighted the excellent documentation that already exists for our SREs, but perhaps more importantly, it highlighted what is missing. Arman took a thoughtful, iterative approach to re-engineering the docs using Jekyll + JustTheDocs and then re-organizing them one directory at a time.

![Ops Docs](https://dev.37signals.com/assets/images/2023-summer-interns/ops-docs.png)

![Ops Docs Project](https://dev.37signals.com/assets/images/2023-summer-interns/ops-docs-project.png)

Arman’s mentor and manager Eron said, *“From the moment we interviewed Arman for the internship, I knew he was going to be a great fit. He was a joy to work with these past three months and his big project, our new Ops Docs system, will serve us well for years to come.”*

**Here’s what Arman had to say about his internship:**

> My internship on the Ops team was immensely challenging and rewarding. I joined the team at the tail end of an exciting migration off-cloud. The intensity was palpable. Every day, I learned so much: Ruby, Rails, Chef, Kamal, Docker, bash scripting, OS commands, general troubleshooting, and more.
> Beyond the Ops Team, I learned from the whole 37signals crew. Because of how we use Basecamp, I could see the other teams working. I set up 1:1s across the company and got to know some of the kindest, most intelligent people. The summer internship was an incredible first step in my career and out of college.

We hope to offer more internships next summer, so keep an eye on [37signals.com/jobs](https://37signals.com/jobs) in early 2024, for more information. Thanks to Yusuf, Jordan, and Arman for an amazing summer. We all feel honored that they chose to spend it with us.

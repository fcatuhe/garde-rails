# Mission Control — Jobs 1.0 released

**Author:** Rosa Gutiérrez, Programmer, SIP
**Published:** December 4, 2024
**Source:** <https://dev.37signals.com/mission-control-jobs-v1-0/>

*We’ve just published the first major release of Mission Control — Jobs.*

---

We’ve just released [Mission Control — Jobs v1.0.0](https://rubygems.org/gems/mission_control-jobs/versions/1.0.0), the dashboard and set of extensions to operate background jobs that [we introduced earlier this year](https://dev.37signals.com/mission-control-jobs/). This new version is the result of 92 pull requests, 67 issues and the help of 35 different contributors. It includes many bugfixes and improvements, such as:
- Support for Solid Queue’s recurring tasks, including running them on-demand.
- Support for API-only apps.
- Allowing immediate dispatching of scheduled and blocked jobs.
- Backtrace cleaning for failed jobs’ backtraces.
- A safer default for authentication, with Basic HTTP authentication enabled and initially closed unless configured or explicitly disabled.

![Screen showing the recurring tasks view of Mission Control Jobs](https://dev.37signals.com/assets/images/mission-control-jobs/screenshot-recurring.png)

Recurring tasks in Mission Control — Jobs, with a subset of the tasks we run in production

We use Mission Control — Jobs daily to manage jobs [HEY](https://www.hey.com/) and [Basecamp 4](https://basecamp.com/), with both Solid Queue and [Resque](https://github.com/resque/resque), and it’s the dashboard we recommend if you’re using Solid Queue for your jobs. Our plan is to upstream some of the extensions we’ve made to Active Job and continue improving it until it’s ready to be included by default in Rails together with Solid Queue.

If you want to help us with that, are interested in learning more or have any issues or questions, [head over to the repo in GitHub](https://github.com/basecamp/mission_control-jobs#readme). We hope you like it!

In 2019, Airbnb decided to migrate the entirety of the Luxury Retreats business onto the Airbnb technical stack. Post acquisition migrations are an often overlooked but absolutely critical part of acquisitions. In this talk we will go through the journey together, covering technical details, challenges, and key takeaways. The following is an outline of the presentation.


# Outline

### 1.  Why did we decide to migrate as opposed to support two stacks?
*Lessons we learned along the way that you can take away from.*

Post acquisition, Luxury Retreats operated on its own tech stack independently of Airbnb's. We began to notice the following drawbacks:

-   Difficulty of collaboration — different codebases and tools made it difficult for engineering and data science to collaborate across the two stacks
-   Large maintenance cost — maintaining two sets of software on different technical stacks created significant overhead and required two distinct sets of engineers, one for each platform
-   Inability to leverage Airbnb Platform — being on a different stack meant LR would be unable to take advantage of core platform features available on Airbnb (experimentation, internationalization, security/fraud checks, etc)

We decided that migrating over to Airbnb's tech stack would be well worth the investment and lay a strong foundation to future growth.

### 2.  What strategies did we employ to execute the migration?
*The technical complexity here was high and we had to keep the business running throughout the entire process*

Luxury Retreats is a complicated business. Behind the website and app, we also had villa management tools, salesforce instances, calendar and pricing databases, and data reporting pipelines.

The Luxury Retreats migration involved many interdependencies. This played a factor in deciding the sequencing of the various pieces that needed to migrate. In broad terms, we went with the following sequencing.

1.  Read paths
2.  Write paths

Read-only products such as the website and iOS app launched first. By doing this, we were able to keep the source of truth constant. Writes would continue to happen on the legacy side, with changes synced over from LR to Airbnb. Following that, products that impacted the write path, such as host tools to create and modify listings and agent tools to create bookings were launched. These changes would allow us to move the source of truth over to Airbnb’s infrastructure transparently to the clients.

### 3.  How did we leverage Airbnb's infrastructure to rebuild the Luxury Retreats website?
*An architecture deep dive as well as key concerns and challenges that were tackled.*

We rebuilt the entirety of the Luxury Retreats website on Airbnb's infrastructure. We decided to maintain as much design and feature parity as possible to limit the variable effects on business performance. By leveraging Airbnb's infrastructure for traffic, server side react rendering, api gateway, backend services, we were able to rebuild the entire website in a time frame of several months.
![enter image description here](https://i.imgur.com/xy99pxs.png)
Key concerns include handling inconsistencies between search and regions between the two products, and backwards incompatible differences in URL structures for region and villa detail pages.

### 4.  What did the data warehouse migration entail?
*A critical and often overlooked piece of the infrastructure.*

The data warehouse and ETL pipelines between Luxury Retreats and Airbnb were very different. Luxury Retreats used SQL Server and even had an unknown number os Excel sheets configured to have read or write access into the data warehouse.

Additionally, accounting principles and operational terminologies were different. We had to undertake innovative solutions to consolidate differences in revenue tracking.

In order to have a continuous view of the business, we had to move Luxury Retreats' historical data over to Airbnb's data warehouse, which meant mapping and migrating data fields properly.

Before and after of the data warehouse
![enter image description here](https://i.imgur.com/yPiIUvH.png)
### 5.  How did we roll out the new website?
*A technical deep dive on the website rollout and business performance confidence through AB testing.*

With a project of this magnitude with so many moving parts, it’s important to keep in mind that at the end of the day business continuity is critical. Taking the product down or negatively impacting the conversion rate would be disastrous for the business and would go against the goal of migration, which was to lay the foundation and groundwork for future work to grow the business.

It was critical to our success to quantify the guest impact of the migration. Given the seasonality of the business, the most accurate way to do this was by running an AB test of user facing products. This was challenging because it involved testing two entirely different web stacks.

The following steps demonstrate the path towards being fully on Airbnb’s web infrastructure in four phases.

1.  Copy over all DNS records from LR name server to Airbnb’s name server, and update our domain registration to point to the correct server accordingly.  
2.  Update DNS record to point to Akamai, Airbnb’s CDN provider which forwards the request to the load balancer. This step is transparent and gives us a platform upstream for both legacy and Airbnb load balancers where we can deploy business logic.  
3.  Use the CDN to send partial traffic to the Airbnb web stack. This is the phase with live experimentation.  
4.  Update CDN so that 100% of LR web traffic goes to new Airbnb web stack, and deprecate all legacy web components.

The web rollout experiment was subject to a tight timeline. Due to the migration sequencing, the web rollout was first, and any delays would block the rollout of subsequent tracks. The following describes the preparations we made ahead of going live to ensure a smooth rollout.

**Power analysis**

More data means more statistical confidence in results. Relative to Airbnb, Luxury Retreats has a lower booking volume, and has a concept of an inquiry that can mature into a booking. In order to prepare an experimentation plan, we calculated the MDEs (minimum detectable effects) of various metrics under consideration at various P values. That data gave us the ability to make an informed tradeoff between how long to run the experiment and the statistical confidence of changes in the metrics that we were evaluating.

**Target metrics**

As with any experiment, it was important to pre-define what the target metrics were - the primary indicators to consider concluding the experiment and shipping to 100%.

From our power analysis, given the lead time required for an inquiry to turn into a booking, it would not have been feasible to use bookings as a target metric for a short experiment duration.

We sought out a metric that was a leading indicator of bookings that had more reasonable MDEs. We ended up choosing “Contacts” (inquires + online bookings), which had a much more manageable MDE of detecting a 10% change within 2 weeks.

One issue with looking solely at contacts was around the quality of contacts. For example, a product change that introduces confusion may encourage more visitors to inquire, but those incremental inquiries may not lead to additional bookings; this change may actually be bad for the business. In order to address this, we introduced “High Intent Inquiries” as the second target metric that would help us quantify changes to the website. To measure intent, we created an inquiry scoring ML model that took a variety of user signals to generate a quality score. We set a predefined threshold to classify the inquiry as high intent based on signals collected at the time of submission. “High intent inquiries” had a MDE of 15% change within 2 weeks.

Ultimately we decided to use contacts and high intent inquiries as our two target metrics for the web experiment.
Funnel events

We sought to define and implement Google Analytics events for all events in the funnel across both the old and new website. This was important for two reasons:

1.  Events were required in order to build metrics for the experiment
2.  Comprehensive funnel events would allow us to course correct during the experimentation period (there would be no time to add logs and wait several days to collect more data)
    
This was challenging because the product and implementation differed in many areas; we had to use the same funnel but implement the events in a way that was consistent and meaningful.

We also built out an experimentation dashboard on Tableau that utilized the GA events and reported daily on all funnel events so we could closely track and monitor performance.

**Launch plan**

Before starting the experiment, we aligned with all stakeholders on the following plan and launch criteria.

1.  The experiment would last for 2 weeks
2.  As long as the target metrics of Contacts and High Intent Inquiries did not drop more than the predefined threshold, the statistically significantly detectable amount (P=0.05) for that time frame, we could launch to 100% the new website.

**Outcome**

When the experiment was live, we monitored the performance in terms of movements in funnel metrics on a daily basis. We shipped product changes during the experimentation phase based on funnel metrics, so the positive impact was likely to be underrepresented in the final set of cumulative data. We reported on the target metrics, their delta, and their confidence interval using a [one tailed test](https://en.wikipedia.org/wiki/One-_and_two-tailed_tests), as lifts in key metrics would not be launch blockers.

The analysis showed fluctuations on contacters, high intent contacts, and low intent contacts within low digit percentage points. Additionally, the one tailed test showed that the lower bound of the 95% confidence interval was within our predefined criteria. We believed this outcome validates our hypothesis that we successfully migrated without any impact to our users, and rolled out to 100% of users.

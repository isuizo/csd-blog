+++
# The title of your blogpost. No sub-titles are allowed, nor are line-breaks.
title = "The 3Ps of IP Geolocation: Pizza, Policy, and Performance"
# Date must be written in YYYY-MM-DD format. This should be updated right before the final PR is made.
date = 2026-04-06
[taxonomies]
# Keep any areas that apply, removing ones that don't. Do not add new areas!
areas = ["Systems"]
# Tags can be set to a collection of a few keywords specific to your blogpost.
# Consider these similar to keywords specified for a research paper.
tags = ["networking", "routing", "LEO networks"]

[extra]
author = {name = "Isabel Suizo", url = "https://www.isabelsuizo.com/" }
# The committee specification is  a list of objects similar to the author.
committee = [
    {name = "Nicolas Christin", url = "https://www.andrew.cmu.edu/user/nicolasc/"},
    {name = "Christos Faloutsos", url = "https://www.cs.cmu.edu/~christos/"},
    {name = "Noah Singer", url = "https://www.noahsinger.org"},
]
+++
IP Geolocation services are prolific on the Internet today. They're used in a variety of applications including fraud detection, targeted advertising, and localized recommendations. While these geolocation providers may employ different approaches, they all aim to answer the seemingly simple question: Given an IP address, where is this IP address located?

Sample response to query using IPInfo `curl ipinfo.io/128.2.208.106`:

```
{
  "ip": "128.2.208.106",
  "hostname": "radish.sherry.cs.cmu.edu",
  "city": "Carnegie",
  "region": "Pennsylvania",
  "country": "US",
  "loc": "40.4087,-80.0834",
  "org": "AS9 Carnegie Mellon University",
  "postal": "15106",
  "timezone": "America/New_York",
  "readme": "https://ipinfo.io/missingauth"
}
```
However, this seemingly simple task has remained an important challenge within the networking community for over twenty-five years. This is because IP addresses lack a consistent, logical mapping to either the physical or network location of their use. Thus, IP geolocation services rely heavily on operator-published mappings of IPs to physical locations, which we call geofeeds, or latency-based inference methods. Today, IP geolocation is most accurately solved using databases that fuse datamining, latency, and provider-supplied information. Even with modern databases, however, IP geolocation remains difficult due to the dynamic nature of IP address assignments which underscore the importance of consistent snapshots for accurate measurements. 

Even with perfectly up-to-date databses, however, the IP geolocation problem would not be solved. Researchers have noted that IP geolocation "[lacks] a consistent ontology for the meaning of 'location'"[^5] and that "Without that clarity, teams may apply the signal incorrectly in: policy enforcement, regulatory workflows, fraud detection, and content localization" [^6]. These challenges combine with two additional growing tensions for network location: preserving users' location privacy, and increasing legislative emphasis on digital rights management and jurisdiction-specific content access restrictions. In these circumstances, the assumption that there is one location for a particular address seems increasingly fraught. 

Hence, we formalize these fundamental challenges of geolocation as three distinct issues, which we call the 3P's of IP geolocation: Pizza (mapping a user to an approximate location physically near them), Policy (mapping a user to a set of laws and regulations governing them), and performance (mapping a user to a network service that can provide high bandwidth, low-latency service). We argue that the 3P's are not merely challenges towards building a unified and accurate geolocation service, but in reality, related and separate problems that must be handled independently. That is, the 'best' response to a pizza query is definitionally different than that of Performance or Policy query, and thus **any geolocation service cannot provide one correct answer that presumes to be accurate for all three use cases**. To bolster the case for considering geolocation as three separate operations, we provide measurements that highlight extreme and worst-case outcomes from conflating these use cases. 


# Defining the 3P's
1. Pizza - Use cases that belong to this category care mostly about *localization* (i.e. Approximately where is this user located?) Targeted ads, localized recommendations, and online banking security are all considered <span style="font-variant:small-caps;">Pizza</span> use cases. For example, Google may infer a user's location from an IP to help determine, "What pizza parlors are near this user?" In this case, Google is interested in information about where the IP is *physically* located. 
2.  <span style="font-variant:small-caps;">Policy</span> - On matters of policy, applications rely on IP geolocation to identify the jurisdiction users fall under. Jurisdiction is particularly important in matters of digital rights management, government-wide censorship, and age-restricted-content laws. For example, TikTok may use IP geolocation to determine whether a user with IP 1.2.3.4 should be eligible to access the app. In this case, TikTok wants information about the *jurisdiction* the user falls under. 
3. <span style="font-variant:small-caps;">Performance</span> - When considering performance, applications want to estimate an IP's location in the network to identify optimal replica mappings nearby. Content delivery networks (CDNs) and cloud providers are applications that typically belong in this category. Wikipedia, a geo-replicated service, uses IP geolocation to determine which cache to map users to. In this case, Wikipedia cares about the *network* location of the IP.

In the rest of this article, we provide intuition for why these use cases must be handled separately, then present measurements that highlight the consequences of conflating our three notions of geolocation. For the sake of simplicity, we highlight two dimensions in this post, but further dimensions of analysis are available in the full paper. 

1. Why can't we use pizza locations for policy queries?
2. Why can't we use pizza locations for performance queries?


# Why can't I use pizza locations for policy queries?
At first glance, the <span style="font-variant:small-caps;">Pizza</span> and <span style="font-variant:small-caps;">Policy</span> queries seem to care about the same thing -- they both simply want to know where on Earth the user is located. And it is true that the ideal answer of both <span style="font-variant:small-caps;">Pizza</span> and <span style="font-variant:small-caps;">Policy</span> use cases is the same when perfect data is available (say, GPS information). From a pragmatic standpoint, however, IP geolocation services have to deal with something we call **geolocation uncertainty** which derives from privacy preservation and limitations on measurement infrastructure. In reality, all services that care about IP geolocation do not (and some argue, should never) have access to perfect location data. In practice, IP geolocation services must infer a user's location and instead report within a range of possible locations. 

Consider the example below of a user in Geneva, CH. In Geneva, users are subject to Switzerland's data privacy laws (FADP), as opposed to GDPR which applies to EU member countries. For Policy queries, a response of "Switzerland" would be the ideal response to inform the service which policy to apply. However, for recommendation applications, a query concerning "Hikes within 1 hour near me", would prefer a response with a range that is geographically distributed without regard for national borders, including eastern France. Hence, in light of geolocation uncertainty, it is important to consider whether the geolocation task cares about pinpointing the user to an approximate location or a jurisdiction region. The cost of conflating these queries becomes more apparent when we dig into a mechanism that underlies many IP geolocation services today[^1]: geofeeds. 
![This image shows a map of Europe, focused on Geneva, CH. A 100km radius is drawn around Geneva while Switzerland is shaded in. The image shows that the 100km radius that may be relevant for Pizza queries is distributed without regard for borders, overlapping with France.](./geneva_gdpr_100km_zoomedout.png)

## Geofeeds: Decent for locality, terrible for jurisdiction 
So what are geofeeds, and how are IP geolocation services using them? Geofeeds[^2] are operator-published files which specify a mapping of subnet to physical location to enhance IP geolocation accuracy. Typically, operators will assign a user to a subnet that belongs to the city closest to the user[^3]. While this is a reasonable approach for localization tasks, this method of mapping completely disregards any notion of borders. Thus, this method of geolocation has the potential to mischaracterize the jurisdiction of users located in border regions.

### Drawing Voronoi Diagrams from Geofeeds
For this study, we utilize Voronoi diagrams [^4] to gain insights into the prevalence of this problem. A Voronoi diagram is partitioning of a plane with n points into convex polygons such that each polygon contains exactly one generating point, and every point in a given polygon is closer to its generating point than to any other. In this case, our plane is a map of countries and our generating points are the cities listed on an operator's geofeed. Hence, we can visualize the regions that map to each city and quantify how often these regions cross boundaries. 

### Results
The Voronoi diagram for Deutsche Telekom is depicted below. While most of the feed data is in Germany, there are additional cities in the geofeed beyond German borders. As a minimum bound for the number of overlapping countries, we only take into account countries with cities present in the geofeed. 
![This image depicts the Voronoi regions colored according to the number of countries that overlap the region.](./europe_overlap_map.jpg)
We find that outside of Germany, Deutsche Telekom's Voronoi regions often overlap multiple countries, reaching up to seven overlapping countries for the region that corresponds to Budapest, Hungary. Even within Germany, we find regions overlapping with other countries near its borders. This mapping demonstrates that denser geofeed cities can help reduce the number of overlapping countries, but the problem is still relevant in border regions. Even taking into account the density of cities within Germany, we find that over 10% of Voronoi regions overlap with at least 2 countries. 
![This image shows a histogram where the Y axis is the percentage of Voronoi regions and the X axis is the number of countries that overlap with the region. The graph has a thin right tail. ](./europe_overlap_hist.jpg)
While geofeeds may be a reasonable mapping for Pizza queries concerned with approximating the location of a user, they fall short for policy tasks that require high accuracy when identifying user's jurisdiction. 

# Why can't I use pizza locations for performance queries?
Before diving into the impacts of conflating Pizza and Performance locations, we shall first explore why we care about the distinction in the first place. 

For starters, what exactly is the performance location? In this article, we describe the performance (or network location) as *the physical point where the user's traffic exits its ISP to peer with the rest of the Internet* (we will explain why in the next paragraph). For most Internet users, the network location is relatively close in proximity to their physical location, as Internet exchange points exist in almost every major metropolitan city. However in some special cases, like satellite networks or remote regions, the closest peering point may actually be very far away. In the case of Starlink, a low-earth orbiting satellite network, the network location is synonymous with its point of presence (PoP), the component where all satellite traffic meets terrestrial traffic.

But why is performance location the same as their network location? In the below example, we have a Starlink user located in Pittsburgh, PA whose PoP is in New York City, NY and we want to access some website hosted by a CDN. Now, if the mapping decision uses a Pizza location (which is an approximiate location for where the user is physically located), this would result in mapping the user to the server in Pittsburgh. However, in the case of Starlink, this leads to a circuitous route from Pittsburgh, through the fixed PoP, then all the way back to Pittsburgh. Instead, a network-aware mapping could reduce propagation delay by mapping this Starlink user to the replica in New York, shortening the overall distance traffic must travel. Thus, knowledge of the network location, as opposed to the physical location may be more useful in the context of performance. 
![This visualization illustrates two paths that highlight the increased propagation delay when using the Pizza location. The first map shows a circuitous route when the replica near Pittsburgh is selected while the second shows a more efficient path using the replica near New York City.](./mappings.png) 

To quantify just how far a user's physical location (for Pizza queries) may be from their network location (for Performance queries), we use real Starlink probes with user-reported locations on the RIPE Atlas testbed. RIPE Atlas provides the coordinates and IP address of the machine, which we use to identify the probe's PoP. We calculated the PoP to probe distance for 108 Starlink probes and visualize the distribution of distances below. 
![This visual shows a CDF of the distance between Starlink probes and their PoPs. There is a steep beginning curve with a long tail.](./cdf_ripe_probe_pop_dist.jpg)
While most RIPE probes are within a few hundred kilometers from their PoP, we observe this long tail behavior with 14.8% of probes at least 1000km away and 10.2% of probes at least 2000km from their PoP. Therefore, the location that Pizza queries care about, the physical location of the probe, and vary drastically from the location relevant to Performance queries. 

## Measuring the Impact of PoP-Aware mappings
To motivate the importance of distinguishing physical versus network location, we designed a measurement study to quantify the benefits of PoP-Aware mappings on real-world systems. In this study, we focus specifically on Starlink users because their network and physical location can be vastly different. To gather realistic data, we benchmarked real-world services with georeplicated deployments: Meta, Akamai, Cloudfront, and Wikipedia. However, given that we want to compare latencies when mapping using the Performance location (server closest to the PoP) versus the Pizza location (server closest to the probe), we must have knowledge about the deployments of these services and target IPs to each of these deployments. This task brought about several challenges. In this next section, we enumerate three challenges along with our mitigation strategies. 
### Building a mapping of services' locations to live IPs
One of the main challenges of this experiment was determining where services' replicas are geographically located and how to ping them. Determining *where* services were located was a challenge because we did not have access to ground truth. Thus, we relied on a combination of third-party IP geolocation databases, WHOIS records, and RTT-based validation. On the other hand, identifying the address for gathering ping data was a challenge because IPs are extremely dynamic and constantly being reassigned or decommissioned. We present these challenges and our mitigation strategies below. 
1. **Where are services located?** To determine where services are located, we used geographically-dispersed DNS queries to gather a variety of responses. First, we generated a mesh of RIPE Atlas probes such that no two probes were within 200km of one another. With this mesh of roughly 500 probes, we ran a DNS query to the service. We then used IPInfo to geolocate each of the resulting IP addresses. This resulted in a mapping of city to IPs for each service. In order to mitigate temporal effects, we repeated this process once per day for three days. 
2. **How can we validate third-party location data?** As this work suggests, IP geolocation alone is insufficient for determining where these IP address are physically located. Therefore, we added a validation step for all IPs added into our database using RTT measurements from RIPE Atlas probes inspired by prior work[^7]. For each (city,IP) pair derived from Step 1, we launched a ping from a set of RIPE Atlas probe validators to the IP. Since RIPE Atlas reports the location of probes, we calculated a lower bound for the time it takes for the ping to reach the city by dividing the distance between the city and probe by the speed of Internet (2/3 speed of light). $$\text{Lower bound for ping} = \frac{\text{Distance between probe and city}}{2/3*c}$$If any ping was faster than the speed of Internet constraint, the server location was removed from the set. 
3. **How do we handle stale IPs?** During this experiment, we found that server IPs were constantly going stale or being reassigned. Therefore, before each ping was launched, we executed our IP refresher script. In this script, we executed three steps of validation: First, we ensured that the IP was still live by pinging it. Second, we validated that the IP was still owned by our service of interest by checking the WHOIS data. Finally, we re-geolocated the address to make sure that the location still matched the recorded server location. If any of these steps failed, we searched for a new IP from a list of 32 random IPs in the same /24. If none of those IPs passed validation, we launched a DNS query from 3 RIPE probes in the vicinity of the server location and re-ran the validation process on the resulting IPs. Finally, if none of those IPs pass, we assume the server has gone offline and remove it from our set. 
### Experiment Design
For each Starlink RIPE Atlas probe we executed the following steps:
1. Identify replica closest to the probe
2. Identify replica closest to the PoP
3. Ping both replicas from the probe
4. Repeat steps 1-3 every 4 hours for 7 days to mitigate time of day effects
![This GIF displays the steps of the experiment outlined above.](./experiment.gif)

### Analysis
Given the non-deterministic behavior of ping times, we utilize stochastic dominance to determine whether the probe or PoP mapping is better. We say that a PoP-aware mapping stochastically dominates a probe-aware mapping if $$
P\left[RTT_{\mathrm{PoP}}(\mathrm{ms}) > x\right] \leq P\left[RTT_{\mathrm{probe}}(\mathrm{ms}) > x\right] \forall x, x \geq 0
$$
![This image displays the CCDFs of the PoP aware RTTs vs. probe-aware RTTs. The PoP-aware CCDF never crosses the probe-aware CCDF and remains always to the left.](./probe_example.png)

Note: In this case, the smaller RTT stochastically dominates the other. This is because we want the convention that faster ping times dominate slower ping times. For example, in the example above, we say that for probe ID 64237, the PoP mapping stochastically dominates probe mapping. 
![This bar chart shows the number of times we see stochastic dominance across all services. Each bar is annotated with the median speed up using the PoP-aware mapping.](./speed_probe.png)
The figure above depicts the results for the 5 major services across 115 Starlink probes. Each bar is annotated with the median speedup using PoP mapping. From this graph, we conclude that for most probes, the probe and PoP mappings are idential (i.e., the server closest to probe is identical to closest to PoP). However, in the cases when they are different, the PoP mapping is generally more optimal and often stochastically dominates the probe mappings. Even in the case where stochastic dominance is not achieved in the mixed category (when one does not strictly stochastically dominate the other), it is still on average, more advantageous to have PoP-aware mappings. Today, there is evidence that most large services do have PoP-aware mappings either explicitly using information from ISPs like pops.csv or implicitly from performance-based mappings like Akamai. Please see our paper for a deeper dive into these results and comparisons between the RTTs of our PoP-aware mappings and state-of-the-art mappings. 


# Summary
In short, our data suggests that the community needs to stop treating IP geolocation as a one-size-fits-all tool and instead move towards a system in which geolocation services offer distinct APIs that are trifurcated along these three dimensions. While the challenges we talk about have been mentioned by others both implicitly and explicitly, our contribution is to provide data illuminating how fundamentally impossible it is to continue with the current regime. Our goal is to motivate the community to move towards geolocation solutions that explicitly subdivide these related but distinct tasks. If you'd like to learn more about our study, or even participate in our crowd-sourced dataset, please visit [whereareyouproject.org](https://whereareyouproject.org/).
[^5]: https://www.potaroo.net/ispcol/2025-12/geoip.html
[^6]: https://ipinfo.io/blog/geofeeds-in-practice
[^1]: IPInfo, Maxmind
[^2]: https://datatracker.ietf.org/doc/rfc8805/
[^3]: https://developer.apple.com/icloud/prepare-your-network-for-icloud-private-relay/
[^4]: https://en.wikipedia.org/wiki/Voronoi_diagram 
[^7]: https://research.owlfolio.org/pubs/2018-catch-proxies-lie.pdf

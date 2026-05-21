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
IP Geolocation services are prolific on the Internet today. They're used in a variety of applications including fraud detection, targeted advertising, and localized recommendations. While these geolocation providers may employ different approaches, they all aim to answer the simple question: Given an IP address, where is this IP address located?

For example, a query to IPInfo `curl ipinfo.io/128.2.208.106` results in the response:

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
However, this seemingly simple task of geolocating an IP address becomes far more complex when we consider the ideal answer in the context of the applications ingesting the geolocation information. We identify three classes of applications that we argue must must be handled differently by IP geolocation. That is, the optimal answer for Class A may be different from Class B. We refer to these three categories as the 3Ps: <span style="font-variant:small-caps;">Pizza</span>, <span style="font-variant:small-caps;">Policy</span>, and <span style="font-variant:small-caps;">Performance</span>. While modern IP geolocation services provide a singular response and do not discriminate between applications, the best answer really depends on the use case. In this work, we provide measurements to support our claim that **a single geolocation answer cannot optimally serve all three use cases**.

# The 3Ps
1. <span style="font-variant:small-caps;">Pizza</span> - Use cases that belong to this category care mostly about *localization* (i.e. Approximately where is this user located?) Targeted ads, localized recommendations, and online banking security are all considered <span style="font-variant:small-caps;">Pizza</span> use cases. For example, Google may infer a user's location from an IP to help determine, "What pizza parlors are near this user?" In this case, Google is interested in information about where the IP is *physically* located. 
2.  <span style="font-variant:small-caps;">Policy</span> - On matters of policy, applications rely on IP geolocation to identify the jurisdiction users fall under. Jurisdiction is particularly important in matters of digital rights management, government-wide censorship, and age-restricted-content laws. For example, TikTok may use IP geolocation to determine whether a user with IP 1.2.3.4 should be eligible to access the app. In this case, TikTok wants information about the *jurisdiction* the user falls under. 
3. <span style="font-variant:small-caps;">Performance</span> - When considering performance, applications want to estimate an IP's location in the network to identify optimal replica mappings nearby. Content delivery networks (CDNs) and cloud providers are applications that typically belong in this category. Wikipedia, a geo-replicated service, uses IP geolocation to determine which cache to map users to. In this case, Wikipedia cares about the *network* location of the IP.
![todo: ALT TEXT](./bifurcation.png)
To address the distinct goals of each use case, we argue that the notion of "geolocation" must bifurcate along two divides, yielding three distinct entities. The first fork is between use cases that are concerned with physical location (<span style="font-variant:small-caps;">Pizza</span> + <span style="font-variant:small-caps;">Policy</span>) vs. use cases concerned with network location (<span style="font-variant:small-caps;">Performance</span>). The second fork distinguishes localization (<span style="font-variant:small-caps;">Pizza</span>) vs. jurisdiction (<span style="font-variant:small-caps;">Policy</span>). It is true that for median users, the impact of individually addressing each of these use cases is minimal. Still, for the billions of users who do not fit this mold (footnote), this unidimensional geolocation model leads to incredibly frustrating tail behavior both from a latency and usability perspective. In the rest of this article, we provide intuition for why these use cases must be handled separately, then present measurements that highlight the consequences of conflating our three notions of geolocation. 

## Key Findings:
* Correctly mapping users based on network location instead of physical location can speed up propagation delay (time it takes for signal to reach destination) by up to 6.1x and on average by 24-41% depending on the service.
* The impact of conflating network vs. physical location is highly dependent on the density of service replicas and routing infrastructure. Users in regions with sparse service replicas and routing infrastructure deployments (e.g. sub-Saharan Africa and Pacific Island nations) see the greatest improvements with network-aware mappings. 
* Modern geolocation methods, specifically geofeeds, are susceptible to mislabeling jurisdiction in border regions. Over 29\% of Verizon and 84.9\% of T-Mobile geofeed regions touch 2 or more states. 

# Network vs. Physical Location
Before diving into the impacts of conflating network and physical location, let's first explore why we care about the distinction in the first place. For starters, what exactly is the network location? In this article, we describe network location as *the physical point where the user's traffic exits its ISP to peer with the rest of the Internet*. For most terrestrial users, the network location is relatively close in proximity to their physical location, as Internet exchange points exist in almost every major metropolitan city. However in some special cases, like satellite networks or remote regions, the closest peering point may actually be very far away. In the case of Starlink, a low-earth orbiting satellite network, the network location is synonymous with its point of presence (PoP), the component where all satellite traffic meets terrestrial traffic.
In the figure above, we approximate the distance of client and PoP distances using two data sources published by Starlink. 
1. [feed.csv](https://geoip.starlinkisp.net/feed.csv): A mapping of IP subnet to approximate location (e.g. `14.1.94.0/24,AU,AU-VIC,Melbourne,`)
2. [pops.csv](https://geoip.starlinkisp.net/pops.csv): A mapping of IP subnet to PoP location (e.g. `14.1.94.0/24,mlbeaus1,mel`)

While the geofeed is a rough approximation for the location of the user, we can use this information to build a sense for how far clients and PoPs on a Starlink networks can be. Joining these datasets on the subnet, we then get a tuple of (IP Subnet, Geofeed Location, PoP Location) and are able to approximate the distance between physical (Geofeed Location) and network (PoP Location) which we present in the CDF below. 
![todo: ALT TEXT](./pop-to-cdf.png)
While 40% of subnets' physical and network location are reported to be the same, we observe a long tail with distances greater than 2000km for the remaining ~10%. Now, how does physical vs. network distance play into performance? Let's motivate this with a real-world example depicted below. Here, we have a Starlink user located in Pittsburgh, PA whose PoP is in New York City, NY and we want to access some website hosted by a CDN. Now for most users, the intuitive (and optimal) mapping would be to the CDN replica also in Pittsburgh. However, in the case of Starlink, this leads to a circuitous route from Pittsburgh, through the fixed PoP, then all the way back to Pittsburgh. Instead, a network-aware mapping could reduce propagation delay by mapping this Starlink user to the replica in New York, shortening the overall distance traffic must travel. Thus, knowledge of the network location, as opposed to the physical location may be more useful in the context of performance. 
![todo: ALT TEXT](./mappings.png)
## Measuring the Impact of PoP-Aware mappings
In order to motivate the importance of distinguishing physical vs. network location, we designed a measurement study to quantify the benefits of PoP-Aware mappings on real-world systems. In this study, we focus specifically on Starlink users because their network and physical location can be vastly different. In order to gather realistic data, we benchmarked real-world services with georeplicated deployments: Meta, Akamai, Cloudfront, and Wikipedia. However, given we want to compare latencies when routing closest to the PoP vs. closest to the probe, we must have knowledge about the deployments of these services and target IPs to each of these deployments. This task brought about several challenges. In this next section, we enumerate three challenges along with our mitigation strategies. 
### Building a mapping of services' locations to live IPs
One of the main challenges of this experiment was determining where services' replicas are geographically located and how to ping them. Determining *where* services were located is a challenge because we did not have access to ground truth. Thus, we relied on a combination of IP geolocation, PTR records, and RTT-based validation. On the other hand, identifying the address for gathering ping data was a challenge because they are extremely dynamic and constantly being reassigned or decommissioned. We present these challenges and our mitigation strategies below. 
1. **Where are services located?** To determine where services are located, we used geographically-dispersed DNS queries to gather a variety of responses. First, we generated a mesh of RIPE Atlas probes such that no two probes were within 200km of one another. With this mesh of roughly 500 probes, we ran a DNS query to the service. For all services except Wikipedia, this results in hundreds of distinct IPs. To limit IP geolocation costs, we then took one representative IP address per /24 and used IPInfo to geolocate each of these IP addresses. We then grouped all the IPs by (city,region) pairs. This resulted in a mapping of city to IPs per service. In order to mitigate temporal effects, we repeated this process once per day for three days. 
2. **How can we validate third-party location data?** As this work suggests, IP geolocation alone is insufficient for determining where these IP address are physically located. Therefore, we added a validation step for all IPs added into our database using RTT measurements from RIPE Atlas probes. To validate, we first selected a mesh of RIPE Atlas probes for validation. Since we used RTT-based measurements, having confidence in the reported locations of RIPE Atlas probes was absolutely necessary. We filtered these probes using two steps. First, we eliminated probes from a flagged list of location-violating probes from Izhikevich et al. Second, we executed a ping from all to all ping for probes in our set. Any probe that violated speed of Internet (SOI) constraints (2/3 speed of light) was removed from the set. This process resulted in ~200 geographically diverse RIPE Atlas probes. For each server location derived from Step 1, we launched a ping from all RIPE Atlas probe validators to its IP. If any ping violated SOI, the server location was removed from the set. 
3. **How do we handle stale IPs?** During this experiment, we found that server IPs are constantly going stale or being reassigned. Therefore, before each ping was launched, we executed our IP refresher script. In this script, we executed three steps of validation: First, we ensured that the IP was still live by pinging it. Second, we validated that the IP was still owned by our service of interest by checking the WHOIS data. Finally, we re-geolocated the address to make sure that the location still matched the recorded server location. If any of these steps failed, we searched for a new IP from a list of 32 random IPs in the same /24. If none of those IPs passed validation, we launched a DNS query from 3 RIPE probes in the vicinity of the server location and re-ran the validation process on the resulting IPs. Finally, if none of those IPs pass, we assume the server has gone offline and remove it from our set. 
### Experiment Design
For each Starlink RIPE Atlas probe we executed the following steps:
1. Identify replica closest to the probe
2. Identify replica closest to the PoP
3. Ping both replicas from the probe
4. Repeat steps 1-3 every 4 hours for 7 days to mitigate time of day effects
![todo: ALT TEXT](./experiment.gif)

### Analysis
Given the non-deterministic behavior of ping times, we utilize stochastic dominance to determine whether the probe or PoP mapping is better. We say that a PoP-aware mapping stochastically dominates a probe-aware mapping if $$
P\left[RTT_{\mathrm{PoP}}(\mathrm{ms}) > x\right] \leq P\left[RTT_{\mathrm{probe}}(\mathrm{ms}) > x\right] \forall x, x \geq 0
$$
![todo: ALT TEXT](./probe_example.png)

Note: In this case, the smaller RTT stochastically dominates the other. This is because we want the convention that faster ping times dominate slower ping times. For example, in the example above, we say that for probe ID 64237, the PoP mapping stochastically dominates probe mapping. 
![todo: ALT TEXT](./speed_probe.png)
The figure above depicts the results for the 5 major services across 115 Starlink probes. Each bar is annotated with the median speedup using PoP mapping. From this graph, we conclude that for most probes, the probe and PoP mappings are idential (i.e., the server closest to probe is identical to closest to PoP). However, in the cases when they are different, the PoP mapping is generally more optimal and often stochastically dominates the probe mappings. Even in the case where stochastic dominance is not achieved in the mixed category (when one does not strictly stochastically dominate the other), it is still on average, more advantageous to have PoP-aware mappings. Today, there is evidence that most large services do have PoP-aware mappings either explicitly using information from ISPs like pops.csv or implicitly from performance-based mappings like Akamai. Please see our paper for a deeper dive into these results and comparisons between the RTTs of our PoP-aware mappings and state-of-the-art mappings. 

# Localization vs. Jurisdiction
The second fork is much more subtle. At first glance, the <span style="font-variant:small-caps;">Pizza</span> and <span style="font-variant:small-caps;">Policy</span> use cases appear to be the same -- they both simply want to know where on Earth you are located. And it is true that the ideal answer of both <span style="font-variant:small-caps;">Pizza</span> and <span style="font-variant:small-caps;">Policy</span> use cases is the same when you have perfect data (say, GPS information). From a pragmatic standpoint, however, IP geolocation services have to deal with something we call **geolocation uncertainty** which derives from privacy preservation and limitations on measurement infrastructure. In reality, all services that care about IP geolocation do not (and some argue, should never) have access to perfect location data. In practice, IP geolocation services must infer a user's location and instead report within a range of possible locations. As depicted in the example below, in <span style="font-variant:small-caps;">Pizza</span> use cases, the correct range has a tight radius, but can be geographically distributed without regard for borders. On the other hand, <span style="font-variant:small-caps;">Policy</span> use cases may have a wider radius of "acceptable" locations, but have a much lower tolerance for error at the jurisdiction level. For example, in the US, state-level geolocation is necessary, as several neighboring states like Pennsylvania and Ohio do have different policy on age-restricted content. Therefore, in light of geolocation uncertainty, it is important to consider whether the geolocation task cares about pinpointing the user to a specific location (<span style="font-variant:small-caps;">Pizza</span>) or a general region (<span style="font-variant:small-caps;">Policy</span>). This distinction becomes more apparent when we dig into a mechanism that underlies many IP geolocation services[^1] today: geofeeds. 
![todo: ALT TEXT](./jurisdiction.png)

## Geofeeds: Decent for locality, terrible for jurisdiction 
So what are geofeeds, and how are IP geolocation services using them? Geofeeds[^2] are operator-published files which specify a mapping of subnet to physical location to enhance IP geolocation accuracy. Typically, operators will assign a user to a subnet that belongs to the city closest to the user[^3]. While this is a reasonable approach for localization tasks, this method of mapping completely disregards any notion of borders. Thus, this mechanism for geolocating users does have the potential to mischaracterize the jurisdiction of users located in border regions. In the following sections, we quantify these potential cases for mis-characterizing jurisdiction across different providers. 

### Drawing Voronoi Diagrams from Geofeeds
For this study, we utilize Voronoi diagrams [^4] to gain insights into the prevalence of this problem. A Voronoi diagram is partitioning of a plane with n points into convex polygons such that each polygon contains exactly one generating point and every point in a given polygon is closer to its generating point than to any other. In this case, our plane is a map of the United States and our generating points are the cities listed on an operator's geofeed. Hence, we can visualize the regions that map to each city and quantify how often these regions cross boundaries. 

### Results
We benchmark four different geofeeds: two mobile providers (Verizon and T-Mobile), one LEO satellite provider (Starlink), and one proxy service (Apple Private Cloud Relay). 
![todo: ALT TEXT](./combined_us_overlapping_states_map.png)
Based on the Voronoi diagrams drawn above, we find that this mechanism of mapping to the "nearest city" is highly susceptible to inferring jurisdiction wrong, but it is also highly dependent on the density of cities listed in the geofeed. Sparse geofeeds like T-Mobile (~75 cities) have large Voronoi regions that can cover up to 9 states. On the other hand, Apple Private Cloud Relay, which publishes an order of magnitude greater number of cities (~5100 cities) than the others has much smaller Voronoi regions. Still, the border issue is not entirely solved, as roughly 20\% of Apple regions still touch 2 or more states. Henceforth, even with fine granularity, geofeeds in their current deployment are still insufficient in solving the jurisdiction task. 
[^1]: IPInfo, Maxmind
[^2]: https://datatracker.ietf.org/doc/rfc8805/
[^3]: https://developer.apple.com/icloud/prepare-your-network-for-icloud-private-relay/
[^4]: https://en.wikipedia.org/wiki/Voronoi_diagram

# Summary
In short, our data suggests that the community needs to stop treating IP geolocation as a one-size-fits-all tool and instead move towards a system in which geolocation services offer distinct APIs that are trifurcated along these three dimensions. While the challenges we talk about have been mentioned by others both implicitly and explicitly, our contribution is to provide data illuminating how fundamentally impossible it is to continue with the current regime. Our goal is to motivate the community to move towards geolocation solutions that explicitly subdivide these related but distinct tasks. If you'd like to learn more about our study, or even participate in our crowd-sourced dataset, please visit [whereareyouproject.org](https://whereareyouproject.org/). 

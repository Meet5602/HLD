URL Shortener

1. Functional Requirements 

2\. Non Functional Requirements

3\. API Design

4\. Capacity Estimation

5\. Database Design

6\. URL Generation Strategy

7\. High Level Architecture

8\. Scaling Bottlenecks \& Improvements



**Function requirements:-** 



* User should be able to submit a long url and receive a shortened version.
* User can optionally provide a custom alias.
* User can set an expiration date to the Url.'
* Users should be able to access the original URL using the shortened URL.



&#x20;                   +------------------+

&#x20;                   |      User        |

&#x20;                   +------------------+

&#x20;                             |

&#x20;                             |

&#x20;                             v

&#x20;                +-------------------------+

&#x20;                | Submit Long URL         |

&#x20;                | Optional Alias          |

&#x20;                | Optional Expiry         |

&#x20;                +-------------------------+

&#x20;                             |

&#x20;                             |

&#x20;                             v

&#x20;                +-------------------------+

&#x20;                | Generate Short URL      |

&#x20;                +-------------------------+

&#x20;                             |

&#x20;                             |

&#x20;                             v

&#x20;                +-------------------------+

&#x20;                | Return Short URL        |

&#x20;                +-------------------------+



\--------------------------------------------------



User Opens Short URL



&#x20;                +-------------------------+

&#x20;                | tinyurl.com/abc123      |

&#x20;                +-------------------------+

&#x20;                             |

&#x20;                             |

&#x20;                             v

&#x20;                +-------------------------+

&#x20;                | Find Original URL       |

&#x20;                +-------------------------+

&#x20;                             |

&#x20;                             |

&#x20;                             v

&#x20;                +-------------------------+

&#x20;                | Redirect User           |

&#x20;                +-------------------------+


**Non-Function requirements:-**

* **URL shortner** should be available almost all the time. 99.99% availability.
* Redirect should happen very fast means low latency. - means later we need **redis cache.**
* Traffic will grow in future so system should be scalable. - **horizontal scalling.**
* **Durability** once a URL is created we should never lose it - **database replications, backups**



&#x20;┌─────────────────────────────────────┐

&#x20;│ High Availability                   │

&#x20;│ Low Latency (<100ms)                │

&#x20;│ Horizontal Scalability              │

&#x20;│ Durability                          │

&#x20;│ Read Heavy Optimization             │

&#x20;└─────────────────────────────────────┘



**API Design:-**



1. **create a short URL**
2. **Redirect URL**


**\*\*\*Create a short URL: -**



POST /api/v1/urls



request: 

\-longUrl

\-customAlias

\-Expirydate



Response:- 

\-Short url



\*\*\*What kind of validation you will perform on the URL?

**-Url format valid or not?**

**-Alias already exists?**

**-Expiry date in future?**

**-URL length allowed?**


**\*\*\*Redirect :-** 

GET/ {ShortCode}



Response:

302 redirect



**\*\*\*why 302 not 301

301 case browser will cache it forever and if real url got changed then also we will get expired one.**



**so in case of 302 browser checks everytime before redirecting.**



**\*\*\*Get URL Metadat: -** 

GET  /api/v1/urls/{shortCode}



**\*\*\*DeleteUrl: -**

DELETE /api/v1/urls/{shortcode}



                    **Capacity Estimation**



**URLs Stored       : 1 Billion**

**Storage Required  : \~1-2 TB**



**Write QPS         : \~1,200/sec**

**Read QPS          : \~120,000/sec**



**Read >> Write**



**Main Bottleneck:**

**✓ Redirect Traffic**



**Likely Solutions:**

**✓ Redis Cache**

**✓ Read Replicas**

**✓ Horizontal Scaling


Step 5: Short URL Generation Strategy**



**1. Random String**

**2. Hashing**

**3. Base62 Encoding**

**4. Snowflake + Base62**



**Pros/Cons of each**

**Which one should we choose?

# Short URL Generation Strategy**



**## Problem Statement**



**The URL Shortener system must generate a unique short URL for every long URL submitted by users.**



**Example:**



**Long URL:**



**https://www.amazon.in/product/12345**



**Short URL:**



**https://tinyurl.com/aBc123**



**The generated short URL should satisfy the following requirements:**



**\* Unique**

**\* Short and user-friendly**

**\* Fast to generate**

**\* Scalable to billions of URLs**

**\* Support distributed systems**

**\* Minimize or eliminate collisions**



**---**



**# Option 1: Random String Generation**



Generate a random string using characters from:



\* a-z

\* A-Z

\* 0-9



Example:



abc123



xyz789



pqrs45



\## Flow



Generate Random String

|

v

Check Database

|

Exists?

|

Yes -> Generate Again

|

No

|

Save URL



\## Advantages



\* Simple implementation

\* No dependency on database-generated IDs

\* Human-readable URLs



\## Disadvantages



\* Collisions are possible

\* Requires database lookup before insert

\* Retry mechanism needed

\* Performance degrades as the number of URLs grows



\## Verdict



Not suitable for large-scale systems because collision probability increases significantly as the dataset grows.



\---



**# Option 2: Hashing**



Generate a hash of the original URL.



Example:



Long URL:



https://google.com



MD5 Hash:



8ffdefbdec956b595d257f0aaeefd623



Short Code:



8ffdefb



\## Flow



Long URL

|

v

MD5 Hash

|

v

Take First N Characters

|

v

Short URL



\## Advantages



\* Deterministic generation

\* Same URL always produces the same hash

\* Fast generation



\## Disadvantages



\* Hash collisions are possible

\* Same URL always maps to the same short URL

\* Collision handling is required

\* Difficult to support multiple short URLs for the same long URL



\## Verdict



Better than random generation but still not ideal for systems handling billions of URLs.



\---



**# Option 3: Auto Increment ID + Base62 Encoding**



Store the URL in the database and use the generated primary key.



Example:



Database ID = 125



Base62(125) = cb



Short URL:



https://tinyurl.com/cb



\## What is Base62?



Base62 uses:



\* a-z (26 characters)

\* A-Z (26 characters)

\* 0-9 (10 characters)



Total:



62 characters



With only 6 characters:



62^6 ≈ 56 Billion combinations



This is more than enough for 1 Billion URLs.



\## Flow



Database ID

|

v

Base62 Encode

|

v

Short URL



\## Advantages



\* No collisions

\* Simple implementation

\* Short URLs

\* Efficient storage



\## Disadvantages



\* Requires a centralized database ID generator

\* Sequential IDs reveal business growth



Example:



abc1

abc2

abc3

abc4



An external observer can estimate the total number of URLs created.



\## Verdict



Excellent solution for small and medium-scale systems.



\---



**# Option 4: Snowflake ID + Base62 Encoding (Chosen Approach)**



\## What is Snowflake?



Snowflake is a distributed unique ID generation algorithm originally introduced by Twitter.



Each generated ID contains:



\* Timestamp

\* Machine ID

\* Sequence Number



This guarantees uniqueness across multiple servers without requiring a centralized database counter.



Example:



Snowflake ID:



987654321234567



Base62 Encoded:



aBcD12



Final URL:



https://tinyurl.com/aBcD12



\## Flow



Long URL

|

v

URL Service

|

v

Generate Snowflake ID

|

v

Base62 Encode

|

v

Short URL

|

v

Store Mapping in Database



\## Advantages



\* Globally unique IDs

\* No collisions

\* Distributed ID generation

\* No database bottleneck

\* Supports horizontal scaling

\* Suitable for billions of URLs

\* Works efficiently in multi-server environments



\## Why Base62?



Snowflake IDs are large numeric values.



Example:



987654321234567



These values are not user-friendly.



Base62 converts them into compact URL-safe strings.



Example:



aBc123



Benefits:



\* Shorter URLs

\* Easy sharing

\* Human-readable

\* URL-safe characters



\---



**# Why We Chose Snowflake + Base62**



The system requirements specify:



\* 1 Billion URLs

\* 100 Million Daily Active Users

\* Massive scale

\* High traffic

\* Multiple application servers



At this scale, a centralized auto-increment ID generator can become a bottleneck because every server must depend on a single source for generating IDs.



Snowflake solves this problem by allowing each server to independently generate globally unique IDs.



After generating the Snowflake ID, Base62 encoding converts the large numeric value into a short, user-friendly URL.



This approach provides:



\* Collision-free URL generation

\* Distributed architecture support

\* High scalability

\* Better performance

\* No dependency on a centralized ID generator



\---



**# Final Decision**



**Chosen Strategy:**



**Snowflake ID + Base62 Encoding**



Reason:



\* Globally unique IDs

\* No collisions

\* Horizontally scalable

\* Distributed generation

\* No database bottleneck

\* Suitable for billions of URLs

\* Produces short and user-friendly URLs



This approach best satisfies the functional and non-functional requirements of a large-scale URL Shortener system.






**## Database Choice**



PostgreSQL



Reason:

\- Strong consistency

\- ACID guarantees

\- Unique constraints

\- Simple schema



\## Primary Key



Snowflake ID



\## Unique Constraint



short\_code



\## Important Index



idx\_short\_code



\## Hot Query



SELECT long\_url

FROM url\_mapping

WHERE short\_code=?



\## Expiry Handling



Validate expires\_at before redirecting.


## Components



1\. Load Balancer

2\. URL Service

3\. Redis Cache

4\. PostgreSQL



\## Write Flow



User

→ URL Service

→ Snowflake ID

→ Base62 Encode

→ Database

→ Return Short URL



\## Read Flow



User

→ URL Service

→ Redis



Cache Hit:

Redis → Redirect



Cache Miss:

Database → Redis → Redirect



\## Benefits



\- Fast redirects

\- Reduced DB load

\- Horizontally scalable

\- High availability



**##Diagram**

                    +----------------+

&#x20;                   |     Client     |

&#x20;                   +-------+--------+

&#x20;                           |

&#x20;                           v

&#x20;                   +----------------+

&#x20;                   | Load Balancer  |

&#x20;                   +-------+--------+

&#x20;                           |

&#x20;            +--------------+--------------+

&#x20;            |                             |

&#x20;            v                             v

&#x20;     +-------------+              +-------------+

&#x20;     | API Server 1|              | API Server 2|

&#x20;     +------+------+              +------+------+

&#x20;            |                            |

&#x20;            +------------+---------------+

&#x20;                         |

&#x20;                         v

&#x20;                 +---------------+

&#x20;                 | Redis Cache   |

&#x20;                 +-------+-------+

&#x20;                         |

&#x20;                Cache Miss |

&#x20;                         v

&#x20;                 +---------------+

&#x20;                 | Primary DB    |

&#x20;                 +-------+-------+

&#x20;                         |

&#x20;                         v

&#x20;                 +---------------+

&#x20;                 | Read Replica  |

&#x20;                 +---------------+


### Few questions after this step is like
**"What happens if Redis goes down?"**



**"What happens if PostgreSQL goes down?"**



**"Can this architecture handle 120,000 reads/sec?"**



**here comes Database Replication :-**



**## Why Replication?**



\- Improve read scalability

\- Increase availability

\- Enable failover



\## Master



Handles:

\- INSERT

\- UPDATE

\- DELETE



\## Replicas



Handles:

\- SELECT queries



\## Benefits



\- Reduced load on master

\- Better read throughput

\- High availability



\## Challenges



\- Replication lag

\- Failover complexity



\## Replication Type



Chosen:

Asynchronous Replication



Reason:

\- Better performance

\- Acceptable eventual consistency`



**# Redis Caching Strategy**



\## Why Redis?



\- Reduce database load

\- Improve redirect latency

\- Handle read-heavy traffic



\## Cache Key



short\_code



Example:



abc123



\## Cache Value



Original URL



Example:



https://amazon.in/product/12345



\## Chosen Strategy



Cache Aside



Flow:



1\. Check Redis

2\. Cache Hit → Return URL

3\. Cache Miss → Query DB

4\. Populate Redis

5\. Return URL



\## TTL



24 Hours



\## Cache Stampede Protection



\- Distributed Locks

\- Background Refresh



\## Redis Failure



System falls back to database.



\## Scaling



Redis Cluster with sharding.

**# Database Sharding**
---



\## Why Sharding?



Problems with a single database:



\- Storage limits

\- Large indexes

\- Write bottlenecks

\- Scalability constraints



\## Chosen Strategy



Hash-Based Sharding



Shard Selection:



hash(shortCode) % N



Example:



abc123 -> Shard 1

xyz789 -> Shard 2

pqr456 -> Shard 3



\## Advantages



\- Even data distribution

\- Better write scalability

\- Better storage scalability



\## Re-Sharding



Problem:

Adding new shards can require data migration.



Solution:

Consistent Hashing


**# Analytics System**
---



\## Requirements



Track:



\- Click Count

\- Country

\- Browser

\- Device

\- Referrer



\## Problem



Analytics should not increase redirect latency.



\## Solution



Use asynchronous event processing.



Flow:



User Click

→ URL Service

→ Kafka Event

→ Redirect User



Analytics Service

→ Consume Kafka Events

→ Store Analytics Data



\## Benefits



\- Fast redirects

\- Scalable analytics

\- Decoupled architecture

\- Handles high traffic



\## Failure Handling



If Kafka is unavailable:

\- Redirect still succeeds

\- Analytics events may be temporarily lost



\## Design Choice



Kafka + Analytics Service + Analytics Database

&#x09;

**# URL Shortener - Interview Questions \& Answers**



\## Q1: Why did you choose Snowflake ID + Base62?



\### Answer



I considered Random Strings, Hashing, Auto Increment IDs, and Snowflake IDs.



Random Strings and Hashing can lead to collisions and require additional collision handling logic.



Auto Increment IDs work well for small and medium-scale systems but introduce a centralized dependency on the database for ID generation.



Since the system needs to support billions of URLs and millions of users, I chose Snowflake IDs because they allow distributed ID generation without relying on a central database.



The generated Snowflake ID is then encoded using Base62 to create a short, URL-friendly identifier.



\---



\## Q2: Why not use Auto Increment IDs?



\### Answer



Auto Increment IDs are simple and collision-free, but they depend on a centralized database.



At large scale, every application server must contact the database to obtain the next ID, which can become a bottleneck.



Additionally, sequential IDs expose business growth because users can estimate how many URLs have been created.



For small-scale systems, Auto Increment + Base62 is a valid solution.



\---



\## Q3: What if Snowflake generates duplicate IDs?



\### Answer



Snowflake IDs consist of:



\* Timestamp

\* Machine ID

\* Sequence Number



Even if multiple servers generate IDs simultaneously, their machine IDs are different.



Within the same server and timestamp, sequence numbers ensure uniqueness.



Therefore collisions are extremely unlikely when implemented correctly.



\---



\## Q4: Why use Base62?



\### Answer



Base62 uses:



\* a-z

\* A-Z

\* 0-9



for a total of 62 characters.



This allows large numeric IDs to be converted into short, URL-friendly strings.



For example:



987654321234567



becomes:



aBc123



which is easier to share and remember.



\---



\## Q5: Why did you choose PostgreSQL instead of NoSQL?



\### Answer



The URL mapping relationship is simple:



short\_code → long\_url



The system requires:



\* Strong consistency

\* Unique constraints

\* ACID guarantees



SQL databases such as PostgreSQL are well suited for these requirements.



At extremely large scale, sharded SQL or NoSQL databases such as Cassandra or DynamoDB could be considered.



\---



\## Q6: Why is Redis needed?



\### Answer



The system is read-heavy.



For every URL creation, there can be hundreds or thousands of redirect requests.



Without Redis, every redirect would hit the database.



Redis stores frequently accessed URL mappings in memory and serves them with very low latency, reducing database load significantly.



\---



\## Q7: Which caching strategy did you choose?



\### Answer



I chose Cache Aside.



Flow:



1\. Check Redis

2\. Cache Hit → Return URL

3\. Cache Miss → Query Database

4\. Store Result in Redis

5\. Return URL



This strategy ensures that only frequently accessed URLs occupy cache memory.



\---



\## Q8: What if Redis goes down?



\### Answer



Redis is not the source of truth.



If Redis becomes unavailable, the application falls back to the database.



The system remains functional but experiences higher latency and increased database load.



To improve availability, Redis Cluster and automatic failover can be used.



\---



\## Q9: What is Cache Stampede?



\### Answer



Cache Stampede occurs when a popular cache entry expires and thousands of requests simultaneously query the database.



Example:



100,000 requests

|

v

Database



This can overload the database.



Solutions:



\* Distributed Locks

\* Background Cache Refresh

\* Staggered Expiration Times



\---



\## Q10: Why use Database Replication?



\### Answer



Replication improves:



\* Read Scalability

\* Availability

\* Failover Support



Writes go to the master database.



Reads are served from replicas.



This distributes load and improves performance.



\---



\## Q11: Why choose Asynchronous Replication?



\### Answer



Asynchronous replication provides better write performance because the master does not wait for replicas to acknowledge writes.



A small amount of replication lag is acceptable for a URL Shortener system.



\---



\## Q12: What if the Master Database fails?



\### Answer



A replica is promoted to become the new master.



This process is called Failover.



Automatic failover mechanisms can be implemented using managed database services or orchestration tools.



\---



\## Q13: Why do we need Sharding?



\### Answer



As data grows to billions of URLs, a single database can become a bottleneck due to:



\* Storage limits

\* Large indexes

\* Write throughput limitations



Sharding distributes data across multiple databases, improving scalability.



\---



\## Q14: Why choose Hash-Based Sharding?



\### Answer



Range-based sharding can create hotspots because all new writes go to the latest shard.



Hash-based sharding distributes data more evenly across shards.



Example:



hash(shortCode) % N



This ensures balanced traffic distribution.



\---



\## Q15: What is Consistent Hashing?



\### Answer



Consistent Hashing minimizes data movement when shards are added or removed.



Without consistent hashing, adding a new shard can require redistributing most of the existing data.



Consistent Hashing only relocates a small subset of keys.



\---



\## Q16: Why use Kafka for Analytics?



\### Answer



Analytics should not slow down redirects.



Instead of updating analytics data synchronously during every redirect, the URL Service publishes events to Kafka.



Analytics consumers process events asynchronously and store aggregated metrics.



This keeps redirect latency low.



\---



\## Q17: What if Kafka goes down?



\### Answer



Redirect functionality should continue working.



Analytics is a secondary concern.



The system should prioritize user experience over analytics collection.



Temporary loss of analytics events is generally acceptable.



\---



\## Q18: Why use a Load Balancer?



\### Answer



A Load Balancer distributes incoming traffic across multiple application instances.



Benefits:



\* Horizontal Scalability

\* High Availability

\* Better Resource Utilization



\---



\## Q19: Why is this system Read Heavy?



\### Answer



URLs are typically created once but accessed many times.



Example:



Create URL Once

Read Thousands of Times



As a result, read traffic is significantly higher than write traffic.



\---



\## Q20: Why use 302 Redirect instead of 301?



\### Answer



301 indicates a permanent redirect and may be cached aggressively by browsers.



302 indicates a temporary redirect and allows the system to retain control over future redirections.



Most URL Shortener systems use 302 redirects.



\---



\## Q21: How would you handle Hot URLs?



\### Answer



A Hot URL is a URL receiving extremely high traffic.



Solutions:



\* Redis Replication

\* CDN Caching

\* Multiple Cache Nodes

\* Geographic Distribution



These approaches prevent a single cache node from becoming a bottleneck.



\---



\## Q22: How would you prevent Abuse?



\### Answer



Implement rate limiting using Redis.



Common approaches:



\* Token Bucket

\* Leaky Bucket



Example:



100 URL creations per minute per user/IP



This prevents spam and resource abuse.



\---



\## Q23: How would you make the system Multi-Region?



\### Answer



Deploy services across multiple regions.



Example:



\* India

\* US

\* Europe



Traffic is routed to the nearest healthy region using DNS-based routing or global load balancing.



This improves availability and reduces latency.



\---



\## Q24: What is the biggest bottleneck in a URL Shortener?



\### Answer



The primary bottleneck is read traffic.



Redirect requests are significantly higher than URL creation requests.



This is why caching, replication, and CDN strategies are critical parts of the design.



\---



\## Q25: How would you evolve this design from Startup Scale to Internet Scale?



\### Answer



Stage 1:



\* Spring Boot

\* PostgreSQL



Stage 2:



\* Add Redis



Stage 3:



\* Multiple Application Servers

\* Load Balancer



Stage 4:



\* Database Replication



Stage 5:



\* Kafka Analytics



Stage 6:



\* Database Sharding



Stage 7:



\* Multi-Region Deployment



This gradual evolution avoids unnecessary complexity while supporting future growth.







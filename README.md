# Instagram Architecture

## Functional requirements:

1. Users post news feeds as a video or image.
2. Users can like a feed.
3. Users can leave a comment for a feed.
4. When a feed is posted, all followers will be notified.

## Non-functional requirements:

1. Service should be reliable and highly available: it should handle single points of failure in such a way, where if any subsystem is down, the entire system continues to operate and should take minimal time to repair itself.
2. Service should be scalable: it should handle an increase of users or workload.
3. Service should be maintainable: it should be divided into independent microservices, therefore it would be easier to modify the system (independent, language agnostic development and release process).

## Out of scope:

1. Security.
2. Analytics.
3. Celebrity use cases.

## Assumptions:

1. DAU - 100 million.
2. Read RPS - 10000.
3. Write RPS - 1000.
4. Average video size - 100Mb.
5. Average image size - 1Mb.
6. Each feed is delivered to 10 subscribers.
7. Average comment is around 100 characters, which is 0.1 kb.

### Average memory size to store feeds and comments for 1 month:
1000 write RPS x 100000 sec * 30 days = 3B write requests per month.
Size = 3B x 100Mb + 3B x 1Mb (size for comments and additional info (indexes, audit data, etc.)) = ~300 PB

### Storage
1. To store static content we can use object storage, like AWS S3 that will be linked to CDN.
2. Posts, feed, comments, likes and user details can be store in relational database. In this case, to get a post 
with comments and likes, we can execute SQL select with inner join within a single transaction.
3. All user relations (followers information) can be stored in Graph DB.

## Initial Architecture
![architecture-instagram-initial.png](images%2Farchitecture-instagram-initial.png)

## Use cases

### User posts a video/image as a feed
1. Client sends request to DNS, that resolves address to IP address.
2. Request is sent to a web server.
3. Web server redirects traffic to internal post service.
4. Post service saves post data to DB, sends requests to fanout service.
5. Fanout service gets data of followers and sends a request to notification service to notify all subscribers.

### User retrieves a feed
1. Client sends request to DNS, that resolves address to IP address.
2. Request is sent to a web server.
3. Web server redirects traffic to internal feed service.
4. Feed service retrieves feed data from DB.
5. Feed service gets additional data (user info, post details, comments, likes) from DB.
6. Feed service returns full data back to the client.
7. Static content is returned as an url to the client and after that it is retrieved through CDN from object storage.

### Comments and likes
For comments and likes we can use separate tables with post foreign key that points to post table:

```sql
TABLE Posts (
    Id INT PRIMARY KEY,
    user_id INT,
    content TEXT,
    created_at TIMESTAMP,
    -- other columns...
);

TABLE Comments (
    id INT PRIMARY KEY,
    user_id INT,
    post_id INT,
    content TEXT,
    created_at TIMESTAMP,
    -- other columns...
    FOREIGN KEY (post_id) REFERENCES Posts(id)
);

TABLE Likes (
    id INT PRIMARY KEY,
    user_id INT,
    post_id INT,
    created_at TIMESTAMP,
    -- other columns...
    FOREIGN KEY (post_id) REFERENCES Posts(id)
);
```


### When comment is added or like should be increased:
Client sends request to DNS, that resolves address to IP address.
Request is sent to a web server.
Web server redirects traffic to internal post service.
Post service updates post data to posts DB and updates likes and comments DB.

## SPOF
1. CDN: If CDN service is down, user will not be able to retrieve static content, therefore the main functionality of the system will not be working.
2. Web server: in case when web server goes down, users will not be able to authorize requests, read and post feeds.
3. Post service: when post service is down or overloaded, users will not be able to post content, add comments or likes.
4. Feed service: when feed service is down, read feed functionality will be missed.
5. DB: when database is down, data can only be retrieved from cache for some period of time based on TTL. 

To eliminate SPOF the next approaches can be used:
1. Add service redundancy: deploy at least 3 instances of each service to keep services highly available.
2. Put services in autoscaling groups in order to upscale/downscale services during high/low load.
3. Add caching between the service and DB.
4. Split DB into master and slave nodes in order to increase read/write throughput.

## Bottlenecks and Optimizations
1. Web server: all traffic from the client to the system is going through webserver that should serve thousands read/write 
requests per second. In order to avoid situations, when all web server's threads are busy, we need to horizontally scale
the web server. Once we scaled the web server, traffic should be routed between the client and web server instances.
Therefore, we need to add load balancer and also scale it horizontally for high availability.
2. Post service: might be a bottleneck when user has many followers. In this case post service should retrieve all followers
from user table and send post notification to each of them.
3. Database: is a bottleneck, because during high traffic feed and post services will wait if all DB connections will be released
in order to serve new requests. In this case DB can be replicated on read slave and write master replicas. Feed service 
will read data from read replica and post service will write data to master replica. This data will be replicated to read replica.
In order to reduce traffic do DB, we can also provide a cache: feed cache that will store `post_id, user_id`, post cache to store post data,
user cache for users and comments and likes respectively.
We can also split DB by functionality: instead of using single DB, we can split it into users DB and posts DB (DB Federation).
4. When user loads a feed, feed service should get a feed from DB, get additional user details, comments, likes, etc. and send request back to the client.
Instead of keeping a feed in DB, we can add a feed cache and precalculate `post_id, user_id` in order to reduce a time for searching 
additional info. We can also reduce write a post request latency by providing a message queue and workers: when new post was saved 
to posts DB, fanout service pushes `post_id, user_id` to the queue from where workers save messages to feed cache.

## Final Architecture
![architecture-instagram-final.png](images%2Farchitecture-instagram-final.png)





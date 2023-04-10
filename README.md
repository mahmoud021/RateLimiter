## RateLimiter
### What is rate limiting in software systems?
Rate limiting is an API’s configured behavior to reject the sender’s request when the number of requests over a window of time crosses a limit or threshold. The sender here could be another system identified using a user id, client id, IP address, or other such identifier.

### Why do software systems need rate limiting ?
If a public API allowed its users to make an unlimited number of requests per hour, it could lead to :
- resource exhaustion
- decreasing quality of the service
- denial of service attacks

This might result in a situation where the service is unavailable or slow. It could also lead to more unexpected costs being incurred by the service.any system should always protect itself against tremendous rps as it directly impacts its availability, cost,latency and it’s overall experience. 

it is clear that rate-limiter as a design choice is fit for following use cases:
- Keep infrastructure cost as per the planned budget.
- Gracefully reject client’s request in case of high load.
- Implement different revenue model for different kind of user’s subscription.
- Security against attack and suspicious activities

## Design for our usecase

As shown in the  diagram, the rate limiter is part of the application and will cap the requests on the server based on only the individual server’s state where it is running.
![1623139562756](https://user-images.githubusercontent.com/122011790/230999895-cbe30dc0-248d-44d1-8a76-630208ca00fb.png)

## The Token Bucket Algorithm
Token Bucket is an algorithm that you can use to implement rate limiting. In short, it works as follows:
- A bucket is created with a certain capacity (number of tokens).
- When a request comes in, the bucket is checked. If there is enough capacity, the request is allowed to proceed. Otherwise, the request is denied.
- When a request is allowed, the capacity is reduced.
- After a certain amount of time, the capacity is replenished.

### Token Bucket Algorithm Concepts
![-viber-2022-01-07-18-44-42-191](https://user-images.githubusercontent.com/122011790/231000651-c35c969b-ca62-4bed-afdb-f7e7cfa7204d.png)

Bucket: As you can see, he has a fixed volume of tokens (if you set 1000 tokens into our bucket, this is the maximum value of volume).  
Refiller: Regularly fills missing tokens based on bandwidth management to Bucket (calling every time before Consume).  
Consume: Takes away tokens from our Bucket (take away 1 token or many tokens -- usually it depends on the weight of calling consume method, it is a customizable and flexible variable, but in 99% of cases, we need to consume only one token).  

Below, you can see an example of a refiller working with Bandwidth management to refresh tokens every minute:

![refill](https://user-images.githubusercontent.com/122011790/231002444-9d2729dd-1ad2-498e-b277-c332c7778315.jpg)



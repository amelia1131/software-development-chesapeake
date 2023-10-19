# Migrating an On-Prem ERP to the Cloud as Microservices leveraging MongoDB

As the lead architect at [Hybrid Web Agency](https://hybridwebagency.com/), one of the most demanding projects I've ever undertaken was the transformation of a legacy monolithic ERP system into a modern microservices architecture. This outdated monolith had become costly to maintain and was holding back my client's business from advancing.

Thankfully, our years of experience providing customized [Software Development Services in Chesapeake](https://hybridwebagency.com/chesapeake-va/best-software-development-company/) had prepared us for challenges like this. We understood the difficulties organizations face when burdened by inflexible and hard-to-change systems. That's why I'm excited to share the step-by-step approach we took to restructure the data model and break down the monolith at both the code and database levels.

During this migration, my team and I discovered some lesser-known best practices for modeling domain entities in a document database like MongoDB to support fully independent microservices. If you've ever wondered how to future-proof your database architecture for the cloud while preserving historical data, you're in for some valuable insights.

By the end of this article, you'll have a clear roadmap for migrating your own legacy systems to a modern architecture. I'll provide practical tips to help you avoid common pitfalls and deliver new features to your customers faster. Let's get started!

## The Advantages of a Microservices Architecture

Implementing a microservices architecture offers numerous benefits over a monolithic approach. Microservices are independently deployable, allowing you to develop and release features quickly without disrupting the entire application.

Each service can be developed using different programming languages and frameworks, giving you the flexibility to choose the best technologies for each domain. For example, you can leverage machine learning libraries in Python for a recommendation engine while using React for the user interface.

This decoupling of concerns enables specialized teams to work autonomously on separate services. It also allows for rapid validation of ideas through prototypes before committing to a full monolithic rewrite. New team members can contribute to a single service that aligns with their skills.

In a rapidly evolving technology landscape, microservices reduce the risk of being tied to specific technologies. When new technologies emerge, you can replace small, isolated parts of the system rather than rewriting the entire monolith. Well-defined interfaces make it seamless to migrate components to new implementations.

#### Independent Scaling

Managing compute resources becomes more efficient when services can scale independently based on demand. For instance, a frontend API gateway can route traffic based on URLs and deploy securely behind a load balancer to handle traffic spikes. During peak periods, such as holidays, only the order processing service needs additional servers, avoiding unnecessary scaling of the entire system.

```
ordersAPI.scale(replicas: 5);
```

Horizontal scaling at the service layer leads to significant cost savings through precise resource allocation. Idle support microservices, such as user profiles, don't require costly overprovisioning to handle traffic that doesn't impact them.

## Analyzing the Data Model

### Understanding Entity Relationships

The first step in migrating to microservices is analyzing how entities are related within the monolithic data model. We examined each collection in the MongoDB database to identify clusters of domains and transaction boundaries.

Common entities like Users, Products, and Orders formed the core of bounded contexts. Relationships between these core objects became candidates for service decomposition. For example, we noticed that Orders contained foreign keys to Users for customer information and to Products to represent purchased items.

To gain a better understanding of these cross-dependencies, we created sample documents to visualize associated fields. We discovered that legacy code merged data that now belonged to separate business capabilities. For instance, shipping addresses redundantly stored user profiles instead of using lightweight references.

Analyzing relationships revealed tight coupling between modules that resulted in cascading updates. Normalizing redundant data removed obstacles to the independent development of user profiles and shipping namespaces.

We used database tools to explore these connections. In MongoDB Compass, we created diagrams of relations through $lookup pipelines and ran aggregate queries to count references between entities. This exposed crucial breakpoints for dividing logic into coherent services.

These relationships guided the definition of domain boundaries and ensured that services had clean interfaces. Well-defined contracts empowered autonomous teams to develop and deploy modules incrementally, acting as micro frontends without blocking each other.

### Identifying Transactional Boundaries

Beyond analyzing relationships, we also examined transactions within the existing codebase to understand business process flows. This helped us identify where data modifications needed to be contained within single services to maintain data consistency and integrity.

For example, in order processing, we observed that any updates related to the order itself, associated payments, inventory levels, and shipment notifications needed to occur transactionally within a single service. This informed the definition of our Order Management service boundary.

Analyzing both relationships and transactions provided crucial insights for refactoring the data model and logic into independently deployable microservices with clearly defined interfaces.

## Refactoring for Microservices

### Normalizing Data Schemas

To support independent services that might use different data stores, we normalized schemas to eliminate duplication and include only the essential data required by each service.

For example, the original Orders schema included the entire User object. We refactored this to use a lightweight reference:

```
// Before
Orders: {
  user: {
    name: 'John',
    address: '123 Main St...' 
  }
  //...
}

// After  
Orders: {
  userId: 1234
  //...  
}
```

Similarly, we separated product details from Orders into their own collections, allowing these entities to evolve independently over time.

### Applying Domain-Driven Design

We applied the concept of bounded contexts from Domain-Driven Design to logically separate services, such as Order Fulfillment and User Profiles. Interfaces abstracted data access:

```
interface UserRepository {
  getUser(id): User;
  updateProfile(user: User): void;
}

class MongoUserRepository implements UserRepository {

  users = db.collection('users');

  async getUser(id) {
    return this.users.findOne({_id: id}); 
  }

  //...
}
```

### Evolving Data Access Patterns

Queries and commands also needed refactoring to support the new architecture. Previously, services accessed data directly using calls like `db.collection.find()`. We introduced abstraction through data access libraries:

```
// order.service.ts

constructor(private ordersRepo: OrderRepository) {}

getOrders() {
  return this.ordersRepo.find();
}

// mongodb.repo.ts

@Injectable()
class MongoOrderRepository implements OrderRepository {

  constructor(private mongoClient: MongoClient) {}

  find() {
   return this.mongoClient.db.collection('orders').find(); 
  }

}
```

This ensured flexibility to migrate databases without changing consumer code.

## Deploying Microservices

### Independent Scaling

With microservices, autoscaling is done at the individual service level rather than application-wide. We implemented scaling logic using Docker Swarm and Kubernetes.

Deployment manifests defined scaling policies based on CPU and memory usage:

```yaml
# docker-compose.yml

services:
  orders:
    image: orders
    deploy:
      resources:
        limits:
          cpus: '0.50'
          memory: 512M
      restart_policy: any
      placement:
        constraints: [node.role == worker]  
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
      rollback_config:
        parallelism: 1
        delay: 10s
        failure

_action: rollback
      scaler:
        min_replicas: 2    
        max_replicas: 6
```

These scaling tiers provided a reservoir of capacity and adeptly managed elastic overflow. The orders service autonomously monitored its performance and spawned or terminated containers as needed:

```js
// orders.js

const cpusUsed = process.cpuUsage();

if(cpusUsed > 80) {
  swarm.scaleService('orders', 1);
}

if(cpusUsed < 20) {
  swarm.scaleService('orders', -1);  
}
```

Load balancers such as Nginx and Traefik meticulously directed traffic to scaled replica sets. This optimization not only enhanced resource utilization but also bolstered throughput while concurrently reducing operational costs.

### Implementing Resilience

Our arsenal of resiliency techniques included retry policies, timeouts, and circuit breakers. Rate limiting and throttling acted as safeguards against cascading failures, while the Platform service shouldered the responsibility of transient error policies for dependent services.

Homegrown and open-source solutions, including Polly, Hystrix, and Resilience4j, stood as stalwart guardians against potential mishaps. Centralized logging through Elasticsearch proved instrumental in tracing errors across the vast landscape of distributed applications.

### Ensuring Reliability

Incorporating reliability into microservices demanded the implementation of a spectrum of techniques designed to stave off single points of failure. We prioritized automated responses to transient errors and designed measures to tackle scenarios of overload.

Leveraging the Resilience4J library, we implemented circuit breakers to gracefully manage faults:

```java
// OrdersService.java

@Slf4j
@Service
public class OrdersService {

  @CircuitBreaker(name="ordersService", fallbackMethod="fallback")
  public List<Order> getOrders() {
    return orderRepository.getOrders();
  }

  public List<Order> fallback(Exception e) {
    log.error("Circuit open, returning empty orders");
    return Collections.emptyList();
  }
}
```

Rate limiting was introduced to mitigate the risk of service flooding during periods of heightened stress:

```java 
// RateLimiterConfig.java

@Configuration
public class RateLimiterConfig {

  @Bean
  public RateLimiter ordersLimiter() {
    return RateLimiter.create(maxOrdersPerSecond); 
  }

}
```

Timeouts were meticulously enforced to curtail prolonged calls:

```java
// ProductClient.java

@HystrixCommand(fallbackMethod="getProductFallback",
               commandProperties={
                 @HystrixProperty(name="execution.isolation.thread.timeoutInMilliseconds", value="1000")  
               })
public Product getProduct(Long id) {
  return productService.getProduct(id);
}
```

We introduced retry logic through policies that were finely defined at the client level:

```java
// RetryConfig.java

@Configuration
public class RetryConfig {

  @Bean
  public RetryTemplate ordersRetryTemplate() {
    SimpleRetryPolicy policy = new SimpleRetryPolicy();
    policy.setMaxAttempts(3);
    return a RetryTemplate(policy);
  }

} 
```

These techniques guaranteed uniform responses and put a halt to cascading failures across the expanse of services.

## In Conclusion

The migration of our monolithic ERP system into the realm of microservices has emerged as an enlightening odyssey. It transcends the boundaries of a mere technical transition and signifies an organizational metamorphosis that equips our client to cater to the ever-evolving needs of their customer base with unmatched agility.

Through the disintegration of tightly interconnected layers and the establishment of well-defined domain boundaries, our development team has unlocked a new realm of agility. Features can now be sculpted and deployed independently, driven solely by business priorities rather than architectural constraints. This newfound capacity for swift experimentation and refinement positions the application to remain perpetually attuned to the dynamic demands of the market.

Simultaneously, our operations team now enjoys unfettered visibility and control over each constituent of the system. Anomalous behaviors are detected promptly through enhanced monitoring of individual services. The deployment of scaling and failover mechanisms has shifted from manual endeavors to automated orchestration, fostering an elevated level of resilience that will serve as the bedrock for our client's sustained expansion.

While the merits of a microservices architecture are beyond dispute, embarking on such a migration is not without its share of challenges. Our painstaking efforts in meticulous relationship analysis, interface definition, and the introduction of abstraction, rather than resorting to a brute 'rip and replace' approach, have bestowed upon us a flexible architectural framework capable of evolving harmoniously with the evolving needs of our clientele.

Above all, I remain profoundly grateful for the opportunity this project has afforded us to collaborate closely with our client on their digital transformation journey. The act of demystifying and sharing our experiences and insights in this article is my way of paying it forward, with the hope that it empowers more businesses to embrace modernization. The rewards, both for customers and businesses alike, are unquestionably worth the endeavor.

## References 

- MongoDB Documentation - Official documentation covering data modeling, queries, deployment, and more. [https://docs.mongodb.com/](https://docs.mongodb.com/)

- Microservices Pattern - Martin Fowler's seminal overview of microservices architecture principles. [https://martinfowler.com/articles/microservices.html](https://martinfowler.com/articles/microservices.html)

- Domain-Driven Design - Eric Evans' book introducing DDD concepts for structuring services around business domains. [https://domainlanguage.com/ddd/](https://domainlanguage.com/ddd/)

- Twelve-Factor App Methodology - Best practices for building software-as-a-service apps that are readily deployable to the cloud. [https://12factor.net/](https://12factor.net/)

- Container Journal - In-depth articles on containerization, orchestration, and cloud platforms. [https://containerjournal.com/](https://containerjournal.com/)

- Docker Documentation - Comprehensive guides for building, deploying, and managing containerized apps. [https://docs.docker.com/](https://docs.docker.com/)

- Kubernetes Documentation - Kubernetes, the leading container orchestrator, and its architecture. [https://kubernetes.io/docs/home/](https://kubernetes.io/docs/home/)

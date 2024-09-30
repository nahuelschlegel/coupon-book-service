# Coupon Book Service

## System Architecture

![System Architecture](./img/CouponBookService.svg "System Architecture")


## Database Design

![Database Design](./img/ERD.svg "Database Design")


## API Endpoints

OpenAPI documentation:

1. Go to [Swagger editor](https://editor-next.swagger.io/).
2. Copy & paste the entire file [openapi.yaml](./openapi.yaml).

For more detailed information, click [here](./docs/endpoints.md).


## Deployment Strategy

I would choose NestJS for the development, which can create different modules for each ECS service. Locally, I use Docker, ensuring that everything that works on my computer will also work when I deploy it to AWS. I would use Terraform or Pulumi as the IaC tool to set up the different environments and accounts in AWS so that each environment is isolated from the others and there is no risk of affecting others' resources. Also, it gives us the facility that once we have the environment set up, it is easily redeployed, and it also allows us to move from the cloud or have a multi-cloud setup.

For the deployment, I will use GitHub Actions or GitLab CI to analyze code, run tests, build, and deploy the containers into Amazon ECR, making the ECS Tasks auto-deploy once it recognizes a new task version. We can use GitHub/Lab cache to improve performance and deployment time. Also, if a migration needs to be run, it can be done from a one-time task for SQL Migration after each deployment.

Each ECS Service will have different auto-scaling rules based on usage and the need for high availability. The PostgreSQL Database can have the Master exclusively for Write operations and other replicas for Read operations, improving the application's performance. We can also store the most frequently read data on Redis, which is even faster and will reduce database access.

I would use a complementary tool to Cloudwatch for monitoring and observability, like New Relic or Grafana.
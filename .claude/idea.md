# Main idea

Ceate solution which can develop and deploy features and build infrastructure in microservice 
infrastructure

# Tech Stack

 - Azure Cloud
 - ASP.NET
 - Mediator 
 - Clean Architecture
 - Feature Slicing 
 - Kubernetes and Azure Container Aps
 - Azure Redis for caching
 - Azure Blobs for files
 - Azure Durable Functions based on docker or Task Scheduler - TBA
 - Cosmos DB - for no sql infra
 - Grpc for internal communication
 - Front Door, Api Gateway, V-net - etc. 


 # Solution Info

 Solution should be consist of agents as team members each agent - represent a specific role.

 - Developer - who develops apps on asp.net and .NET
 - Tech Lead - who architects certain projects in ASP.NET based on SOLID, Clean Architecture, DDD, etc.
 - Solution Architect - who drives e2e - solution how apps communicates to each other, NFR-s, Architecture Significant Features
 - Devops Architect - who drives deployment design
 - QA - who drives testing strategy and creates integration tests
 - Business Analytic - who transforms raw feature text into Functional Requirements and communicates with Solution Architect to sort out NFR-s 

Solution should be able to work as on local machine using claude agents and as end solution in Azure Foundry (Future)

# First Steps

## Determine Architecture for Asp.NET projects (internal)
- How nuget packages will be organized
- How projects will be organized 
- etc
## Determine Architecture for Azure 
- DAPR communication 
- Azure Infrastructure
- etc
## Determine Architecture for Github 
- Deployment
- Service per repo strategy
- Infrastructure as a code

# Project Info

## Main

Technically we want to build universal system based on microservices which can address different business features. At the same time for first step let's start with some certain business case to make a sceleton based on somethign.

## Business Info

We have a portal which can maintain documents
 - Generate Documents
 - Process Documents - (Transaction flow with different states)
 - Store documents in a cloud storage and move between different folders and containers.
 - System will be multi-tenant
 - So we have, Organization, Workspace, Folder, and probably other containers.
 - To each Container we can assign and unassgin different users.

# Part1:

Documents should be generated using template 
Assumption - template will be based on XML (Suggest Better)
We need a portal which can help user to generate a template 
Output formats -> Excel | CSV, PDF, DOCX
For now we do not know data structure but let's assume that it will be relational data and it will be stored in SQL, like customer, product, order, etc. Main idea to generate reports, statements and other type of documents.

# Part2:

We need to have a portal where we manage documents change statuses, apply changes, allow simultaneous changing (like office 365)
Archive documents and so, on

This is just rough info which when we have agents team will be drilled by Solution Architect (SA) and Business Analyst (BA)

## Infra Part 

- For now we consider a solution which can be deployed to AKS or to ACA depends on a flag.
- We need to use DAPR to use all comunications retries etc.
- We use only Managed Identity for secured connection service to service or serrvice to any SaaS like Redis Cache
- We use gRPC for internal connections and REST for public
- all other related infra
- for now - no V-NET design is required however in future we will be trying to address different policies like PCI DSS or something like that, so we should look forward.
- As a multi-tenant infra - we need to address for future:
  1. Dedicated compute
  2. Dedicated logs
  3. Dedicated storage
  based on Tenant Tiers
- Logs -> AppInsights and Log Analytics using OTEl Collector
- Durable functions will be deployed and used in Docker and will be used for Transaction Flows and will stand as a orchestrator microservice if needed.
- For now we will have only generic services - this is a first step - no business services.
- We will have assets structure
   - Core Asset - contains core service which stands for generic use cases like orchestrator, Authorization service, etc.
   - Business assets - which will be dedicated to specific business usecase asset.
   - Technically Assent can be shared to multiple tenants for instance our case with document generation can have many tenants but asset will be DocGen. 
- In github we will have iaac repo which will have yaml and will control all variables which are used during deployment, variables which should be provisioned during deployment in runtime Azure App Configuration and Keyvaults. Github secrets which will be provisioned for runtime (if it is needed) in repo we will have reference to github secret. File type - yaml. Example - TBA
- Resources which are dedicated to a specific asset - must have an appropriate tags.
- We have a Github owner and repos, 1 repo - 1 service, repo should have some indication to which asset it belongs.
- Deployment - TBA

Again this is just a draft which we will discuss with SA and Devops Architect.



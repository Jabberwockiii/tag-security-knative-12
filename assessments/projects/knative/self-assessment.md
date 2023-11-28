# Knative Self-Assessment
Security reviewers: Maya Humston, Ethan Vazquez, Zhikang Xu, Shambhavi Seth
This document is intended to assess the current state of security for the Knative project. 
## Table of Contents
* [Metadata](#metadata)
  * [Security links](#security-links)
* [Overview](#overview)
  * [Actors](#actors)
  * [Actions](#actions)
  * [Background](#background)
  * [Goals](#goals)
  * [Non-goals](#non-goals)
* [Self-assessment use](#self-assessment-use)
* [Security functions and features](#security-functions-and-features)
* [Project compliance](#project-compliance)
* [Secure development practices](#secure-development-practices)
* [Security issue resolution](#security-issue-resolution)
* [Appendix](#appendix)
## Metadata
| | |
|-----------|------|
| Software | https://github.com/knative/serving https://github.com/knative/eventing https://github.com/knative/func |
| Security Provider? | No — the primary function of Knative is to provide a set of components and building blocks to extend Kubernetes functionality and should not be considered primarily a security provider. |
| Languages | Go |
| Software Bill of Materials | Packages: This Knative repo is a storage site for common Knative packages across all other Knative repositories for each component: https://github.com/knative/pkg |
| | Each version release description contains a list of added or updated direct dependencies available to view. https://github.com/knative/func/releases https://github.com/knative/serving/releases https://github.com/knative/eventing/releases |
| |However, all the Go libraries that Knative is dependent on are not readily available. Additionally, I cannot find a FULL auto-generated list of Knative’s direct dependencies other than the ones that were added or changed in each version release. This is a potential weakness and should be added to the roadmap. |
| Security Links | Knative security and disclosure information: https://knative.dev/docs/reference/security/ |
| | Knative threat model: https://github.com/knative/community/blob/main/working-groups/security/threat-model.md | 
| | Response policy to security breach: https://github.com/knative/community/blob/main/working-groups/security/responding.md |
| | Configuration details for serving and eventing, including default configmaps: https://github.com/knative/serving/blob/main/config/README.md https://github.com/knative/eventing/tree/main/config |
|-----------|------|
## Overview
Knative is an open-source platform that enhances Kubernetes, focusing on the efficient deployment and management of serverless, cloud-native applications. It introduces two main components: Serving and Eventing. Serving automates the scaling and management of serverless workloads, while Eventing deals with event-driven architecture, managing and triggering serverless functions. This integration with Kubernetes not only streamlines the deployment process but also boosts developer productivity and reduces operational costs. Knative's cloud-agnostic nature and extensibility make it a versatile choice for diverse serverless computing environments.
Source: https://knative.dev/docs/
### Background
Historically, in the world of container orchestration and cloud-native development, managing and deploying serverless applications has been a complex task. Traditional methods often required developers to manually handle aspects like scaling, updating, and event management for each application separately. This approach was not only time-consuming but also prone to inconsistencies, especially in environments with a diverse set of applications and services.
In cloud-native ecosystems, especially those using Kubernetes, the management of serverless workloads presented unique challenges. Kubernetes, while powerful, did not natively offer specialized tools for serverless architecture, leading developers to either rely on external tools or build custom solutions. These methods, though effective, lacked uniformity and efficiency, especially in handling auto-scaling, rapid deployment, and event-driven architecture.
Knative emerged as a solution to these challenges. It was designed to simplify the process of deploying and managing serverless applications on Kubernetes. By providing middleware components, Knative enables developers to focus more on building their applications rather than worrying about the underlying infrastructure.
### Actors
**Serving Controller** (Service) (https://knative.dev/docs/serving/): This is a central component in Knative responsible for orchestrating the deployment and scaling of serverless workloads. It operates independently, managing the lifecycle of Knative services and their revisions. The Serving Controller is isolated in such a way that even if it encounters issues or vulnerabilities, these do not directly compromise the serverless workloads it manages.

1. Deployment and Management of Services: The Serving Controller is responsible for deploying Knative services. It manages the entire lifecycle of these services, from creation and updating to scaling and deletion.

2. Handling Revisions: Each time a Knative service is updated (like a change in its code or configuration), the Serving Controller creates a new Revision. Revisions are immutable snapshots of a service at a particular point in time. The Serving Controller ensures that each Revision is correctly deployed and operational.

3. Scaling: One of the key features of Knative is automatic scaling, including scaling to zero when a service is not in use. The Serving Controller manages this by monitoring traffic patterns and adjusting the number of active pods (instances of Revisions) accordingly.

4. Traffic Routing: The Serving Controller also plays a role in routing traffic to different Revisions. This is important for features like canary deployments or blue-green deployments, where traffic needs to be gradually shifted from one version of a service to another.
Isolation and Security: The Serving Controller is designed to be isolated in its operation. This means that issues or vulnerabilities within the Serving Controller do not directly compromise the serverless workloads it manages. This isolation is crucial for maintaining the security and integrity of the serverless applications running on Knative.
Integration with Kubernetes Knative Serving, and by extension the Serving Controller, is built on top of Kubernetes. It leverages Kubernetes features like custom resources and controllers, providing a familiar environment for those who are already experienced with Kubernetes.

5. Customization and Extensibility: The Serving Controller allows customization of serverless workloads. Users can specify various configurations like environment variables, resource limits, and request timeouts. This level of customization makes it flexible to cater to diverse application needs.

6. Monitoring and Logging: The Serving Controller supports integration with monitoring and logging tools, allowing users to keep track of the performance and health of their serverless applications.

**Revisions** (https://knative.dev/docs/concepts/serving-resources/revisions/): These are immutable snapshots of serverless applications, each representing a specific version of a workload. Revisions in Knative are functionally independent; they are scaled and managed autonomously based on demand. Their isolation is crucial as it ensures that a compromise or failure in one Revision does not affect the others or the broader system.


1. Immutable Snapshots: Revisions in Knative are immutable snapshots of serverless applications. Each Revision captures a specific point in time, containing the application code and configuration for each change made to a Knative Service.


2. Automatic Creation and Immutability: A new Revision is automatically created whenever there is an update to a Configuration. Once created, Revisions cannot be modified directly. This immutability ensures the stability and reliability of the deployed services.


3. Version Control: One can compare them to version control tags. Like tags that mark specific commits in a version control system, Revisions in Knative represent specific versions of the application. However, similar to tags, these Revisions cannot be edited or updated once created.


4. Scaling and Management: Knative Revisions can be scaled up or down based on traffic demands. This feature allows for efficient resource utilization, as Revisions can be dynamically adjusted to handle varying levels of user requests.


5. Isolation and Independence: Each Revision in Knative is functionally independent and is managed autonomously. This means that changes or issues in one Revision do not affect others. This isolation is crucial for maintaining the stability and security of the entire system.


**Route and Configuration Resources** (https://knative.dev/docs/serving/traffic-management/): These components in Knative act as independent actors. The Route resource is responsible for managing the routing of traffic to different Revisions, while the Configuration resource maintains the desired state for the workloads. Both Route and Configuration operate independently and are isolated from each other and from the workloads they manage, ensuring that vulnerabilities or issues in one do not impact the other or the serverless applications.


**Eventing Components** (https://knative.dev/docs/eventing/): Eventing is handled by independent components such as Brokers and Triggers. These actors manage the flow and filtering of events in the system. Their isolation is key, as it prevents a compromised event source or broker from affecting other parts of the system, thereby limiting the scope of potential security breaches.
Knative Eventing is a key part of Knative's event-driven architecture, designed to manage the flow and filtering of events in serverless applications. Here are more details about its components and functionalities:


1. API-Driven Event Handling Knative Eventing uses a collection of APIs to enable event-driven architecture in applications. These APIs facilitate the routing of events from producers (sources) to consumers (sinks). Additionally, sinks can be configured to respond to HTTP requests by sending response events.

2. Support for Various Workloads: It supports a range of workloads, including standard Kubernetes Services and Knative Serving Services. The eventing model is versatile and accommodates different application types.


3. Standard Communication Protocol: Communication between event producers and sinks is handled through standard HTTP POST requests. These events adhere to the CloudEvents specifications, ensuring compatibility and ease of use across different programming languages.


4. Loosely Coupled Components: The components within Knative Eventing are loosely coupled, allowing them to be developed and deployed independently. This flexibility means that event producers can generate events even if there are no active consumers yet, and consumers can express interest in events even before they are produced.


5. Publishing and Consuming Events: Knative Eventing allows for the publication of events without the need for a specific consumer. Events can be sent to a broker via HTTP POST, with binding used to decouple the destination configuration from the event-producing application. Conversely, events can be consumed without a designated publisher, using triggers to consume events from a broker based on their attributes.


6. Isolation and Security: The isolation of components like Brokers and Triggers is crucial in Knative Eventing. It ensures that a compromised event source or broker does not affect other parts of the system. This design limits the scope of potential security breaches, maintaining the integrity and security of the entire event-driven system.


**Knative Functions**(https://knative.dev/docs/functions/):
Knative Functions is an extension of the Knative ecosystem, designed to simplify the deployment and management of serverless functions. Similar to Knative Eventing, it has several key components and features that enable efficient and scalable function execution in a cloud-native environment:


1.   Serverless Function Deployment  : Knative Functions focuses on the deployment of serverless functions, allowing developers to write code in their language of choice. These functions are automatically scaled up or down based on demand, ensuring efficient resource utilization.


2.   Integration with Knative Eventing  : Functions in Knative can be triggered by events managed by Knative Eventing. This integration enables a seamless flow where functions are executed in response to various events, enhancing the event-driven architecture.


3.   Simplified Function Management  : The platform provides tools and APIs that simplify the process of deploying, updating, and managing functions. This includes straightforward CLI tools and declarative configurations, making it easier for developers to handle their serverless functions.


4.   Support for Multiple Languages  : Knative Functions supports a wide range of programming languages, allowing developers to write functions in the language they are most comfortable with. This flexibility facilitates the adoption of serverless architecture across different development teams.


5.   Automatic Scaling  : Like Knative Serving, Knative Functions can automatically scale the number of function instances based on the incoming request volume. This feature ensures that functions are highly available and can handle varying loads efficiently.


6.   Built-In Observability  : Knative Functions come with built-in monitoring and logging capabilities. This allows developers to easily track the performance of their functions and diagnose issues, ensuring the reliability and stability of the functions.


7.   Event Source Integration  : Functions can be directly integrated with various event sources, allowing them to react to a wide range of events from different sources. This makes Knative Functions ideal for building complex, event-driven applications.


8.   Container-Based Function Execution  : Functions in Knative are executed in containers, leveraging Kubernetes' container orchestration capabilities. This approach ensures portability, scalability, and consistent execution environments for functions.


### Actions
Configuration Reading by Serving Controller: The Serving Controller in Knative reads the user's configuration to determine the setup of serverless workloads. It customizes runtime behavior based on parameters like scaling options and traffic splitting.


Initialization of Revisions: Sequentially, Knative initiates Revisions, which are individual instances of serverless applications. Each Revision is independently initialized, where it reads its specific configuration detailing aspects like environment variables and resource limits. This process is driven by the Serving Controller based on user-defined specifications in the service configuration.


Log and Metrics Handling: Knative's integrated logging and monitoring framework enables each component, such as Revisions and Eventing Brokers, to generate and emit logs and metrics. These logs provide insights into the execution and performance of the serverless workloads and event flows.


Event Processing in Eventing Components: In the event-driven model, Eventing components like Brokers and Triggers process events independently. They handle event routing, filtering, and delivery to the appropriate services or Revisions, based on the eventing configuration specified by the user.


Summary and Reporting: Upon completion of the processing of serverless requests and events, Knative components, particularly the monitoring and logging systems, can provide summaries and detailed reports on the overall performance, including metrics like request count, response times, and event delivery success rates.


1. Services ("service.serving.knative.dev")
For services, it has the following 
- Purpose  : Manages the entire lifecycle of your serverless workload.
-   Common Operations  :
  -   Create a Service  : "kubectl apply -f service.yaml" where "service.yaml" defines the service.
  -   Update a Service  : Modify the "service.yaml" file and reapply it.
  -   Delete a Service  : "kubectl delete -f service.yaml".
-   Key Features  :
  - Automatically manages related Routes, Configurations, and Revisions.
  - Supports both automatic routing to the latest revision and pinning to specific revisions.


 2. Routes ("route.serving.knative.dev")


-   Purpose  : Routes requests to different Revisions based on defined rules.
-   Common Operations  :
  -   Create a Route  : Define the route in a YAML file and apply it using "kubectl apply -f route.yaml".
  -   Update a Route  : Modify the route configuration in the YAML file and reapply.
  -   Delete a Route  : "kubectl delete -f route.yaml".
-   Key Features  :
  - Can split traffic between different revisions.
  - Supports named routes for customized URL paths.


 3. Configurations ("configuration.serving.knative.dev")


-   Purpose  : Maintains the desired state for deployments, separating code from configuration.
-   Common Operations  :
  -   Create a Configuration  : "kubectl apply -f configuration.yaml" with the desired state.
  -   Update a Configuration  : Modify "configuration.yaml" and reapply.
  -   Delete a Configuration  : "kubectl delete -f configuration.yaml".
-   Key Features  :
  - Changes trigger new revisions.
  - Follows Twelve-Factor App principles for cloud-native applications.


 4. Revisions ("revision.serving.knative.dev")


-   Purpose  : Immutable snapshots of application code and configuration at a point in time.
-   Common Operations  :
  -   View Revisions  : "kubectl get revisions" to list all revisions.
  -   Inspect a Revision  : "kubectl describe revision <revision-name>".
  -   Delete a Revision  : Typically managed automatically, but can be manually deleted using "kubectl delete revision <revision-name>".
-   Key Features  :
  - Automatically scaled based on traffic.
  - Direct creation by developers is restricted.
5.  Event Sources ("source.eventing.knative.dev") 
   -  Purpose : Provides mechanisms to fetch events from various external systems and deliver them to event consumers in Knative.
   -  Common Operations :
     -  Create an Event Source : Use a YAML file to define the event source and apply it with "kubectl apply -f source.yaml".
     -  Update an Event Source : Modify the "source.yaml" file and reapply it.
     -  Delete an Event Source : Execute "kubectl delete -f source.yaml".
   -  Key Features :
     - Supports various event sources like Kafka, GCP Pub/Sub, AWS SQS, etc.
     - Facilitates event-driven architecture by bringing external events into Knative.


6.  Brokers ("broker.eventing.knative.dev") 
   -  Purpose : Acts as an event ingress and delivery point within a Knative Eventing system.
   -  Common Operations :
     -  Create a Broker : Define the broker in a YAML file and apply it using "kubectl apply -f broker.yaml".
     -  Update a Broker : Modify the "broker.yaml" file and reapply.
     -  Delete a Broker : Use "kubectl delete -f broker.yaml".
   -  Key Features :
     - Provides event delivery, filtering, and retry logic.
     - Integrates with various Channels for event distribution.


7.  Triggers ("trigger.eventing.knative.dev") 
   -  Purpose : Allows for subscribing to events from a specific Broker and filtering them based on event attributes.
   -  Common Operations :
     -  Create a Trigger : Define the trigger in a YAML file and apply it with "kubectl apply -f trigger.yaml".
     -  Update a Trigger : Adjust the "trigger.yaml" file and reapply.
     -  Delete a Trigger : Execute "kubectl delete -f trigger.yaml".
   -  Key Features :
     - Enables event filtering at the source level, reducing unnecessary network traffic.
     - Simplifies the linkage between event producers and consumers.


8.  Channels ("channel.messaging.knative.dev") 
   -  Purpose : Provides a durable event storage mechanism and ensures reliable delivery of events to multiple subscribers.
   -  Common Operations :
     -  Create a Channel : Use a YAML file to define the channel and apply it with "kubectl apply -f channel.yaml".
     -  Update a Channel : Modify the "channel.yaml" file and reapply.
     -  Delete a Channel : Perform "kubectl delete -f channel.yaml".
   -  Key Features :
     - Ensures ordered, once-only delivery of events.
     - Can be backed by different messaging systems like Kafka, NATS, etc.
9.  Creating a Function 
   -  Purpose : Allows developers to create a serverless function from a simple piece of code without worrying about the underlying infrastructure.
   -  Common Operations :
     -  Develop Function : Write the function code in a supported language.
     -  Define Function Configuration : Specify the function's attributes, dependencies, and triggers in a configuration file.
     -  Deploy Function : Use a command like "kn func deploy" to deploy the function to a Knative environment.


10.  Updating a Function 
   -  Purpose : Modify existing serverless functions to adapt to new requirements or fix issues.
   -  Common Operations :
     -  Update Code : Modify the function code as needed.
     -  Update Configuration : Adjust the function's configuration file to reflect any changes in dependencies, environment variables, etc.
     -  Redeploy Function : Redeploy the function using the deployment command, which often automatically builds and updates the function in the Knative environment.


11.  Deleting a Function 
   -  Purpose : Remove a function from the Knative environment when it is no longer needed.
   -  Common Operations :
     -  Delete Command : Execute a command like "kn func delete" to remove the function from the environment.


12.  Monitoring and Logging 
   -  Purpose : Provide insights into the function's performance and issues.
   -  Common Operations :
        -  Access Logs : Retrieve logs generated by the function for debugging and monitoring purposes.
        -  Performance Metrics : Utilize integrated monitoring tools to track various metrics such as invocation count, execution time, and resource usage.


13.  Scaling 
   -  Purpose : Automatically scale function instances based on demand.
   -  Common Operations :
     -  Auto-Scaling : Functions are automatically scaled up or down based on the number of incoming requests or events, leveraging Knative's built-in scaling capabilities.
     -  Manual Scaling : Optionally, developers can specify scaling parameters, like min and max number of instances, in the function configuration.


14.  Event Binding 
   -  Purpose : Connect functions to various event sources for event-driven architectures.
   -  Common Operations :
     -  Define Event Sources : Specify event sources in the function configuration.
     -  Event Processing : Write function logic to process incoming events from the defined sources.


15.  Versioning and Revisions 
   -  Purpose : Manage different versions of a function.
   -  Common Operations :
     -  Create Revisions : Each update to a function can create a new immutable revision.
     -  Rollback : Ability to roll back to a previous version of the function if needed.


## Goals
Knative aims to make scalable, secure, stateless architectures available quickly by abstracting away the complex details of a Kubernetes installation and enabling developers to focus on what matters.


Security Goals -
1. Secure Communication: Knative aims to ensure secure communication over the internet. This includes securing communication between various Knative components, as well as securing the communication channels through which end-users interact with serverless applications.

2. Input Validation and Sanitization: Given that serverless applications often touch the internet, Knative aims to provide mechanisms for validating and sanitizing input to prevent common security vulnerabilities. This involves addressing issues such as injection attacks and ensuring that untrusted input is properly handled.

3. Sensitive Data Handling: Knative recognizes the importance of handling sensitive data securely. This includes mechanisms to protect sensitive information within serverless applications, whether it's during data processing, storage, or transmission.

4. Role-Based Access Control (RBAC): Knative aims to implement robust RBAC mechanisms to control access to its components and resources. This helps ensure that only authorized users and services can interact with and modify KNative deployments.

5. Container Image Security: As Knative involves building and deploying container images, the project is likely to emphasize best practices for securing container images, such as using signed images, regularly updating dependencies, and implementing image scanning for vulnerabilities.
### Non-Goals
- Long-Term Data Storage - KNative may not intend to provide a solution for long-term data storage. Users might be directed to leverage other storage solutions or databases that are better suited for persistent data.
- Direct Management of Kubernetes Resources - KNative might not aim to expose all the intricacies of Kubernetes resource management directly to end-users. The abstraction provided by KNative is designed to shield users from low-level Kubernetes operations.
- Fine-grained Network Security Policy - Knative may not aim to offer fine-grained network security policy management at the application level. Users are encouraged to leverage Kubernetes Network Policies or other networking solutions for such requirements.
 
## Self-assessment Use
This self-assessment is created by a team of cybersecurity students at NYU to perform an external analysis of the project's security. It is not intended to provide a security audit of Knative, or function as an independent assessment or attestation of Knative's security health.

This document serves to provide Knative users with an initial understanding of Knative's security, where to find existing security documentation, Knative plans for security, and general overview of Knative security practices, both for development of Knative as well as security of Knative.

This document provides the CNCF TAG-Security with an initial understanding of Knative to assist in a joint-assessment, necessary for projects under incubation. Taken together, this document and the joint-assessment serve as a cornerstone for if and when Knative seeks graduation and is preparing for a security audit.


## Security functions and features
| Component | Applicability | Description of Importance |
| --------- | ------------- | ------------------------- |
| Code Signature Verification | Security Relevant | Knative signs their releases with cosign to verify binaries, ensuring integrity and authenticity for users. |
| Istio | Security Relevant | Istio is an open platform-independent service mesh that provides traffic management, policy enforcement, and telemetry collection. Knative Eventing uses Istio for networking operations like traffic rules, authorization policies, and encryption. |
| Kubernetes Infrastructure | Critical | Knative is built upon Kubernetes, which is configured with security tools like RBAC, network policies, and secrets. |
| Security Guard | Security Relevant | Guard help secure microservices, serverless containers and serverless function. It detects and block exploits sent to services and can detect and restart compromised service pods. |

## Project Compliance
Not enough information could be gathered to determine exactly what regulatory standards Knative itself complies with. However, Knative is built on Kubernetes, which follows the following regulatory standards depending on what industry it is deployed in: SOC 2, PCI DSS, HIPAA, NIST SP 800-53, GDPR, ISO 27001. It could be said that Knative follows the CNCF Community Code of Conduct and the CNCF IP Policy. This should be easily accessible information and should be considered in the documentation. 

## Secure Development Practices
The Knative project is still incubating. The security policies and development practices detailed below are based on open source guidelines and preliminary threat modeling. 

### Deployment Pipeline
- Committers are required to sign the CNCF EasyCLA.
    - https://github.com/knative/community/blob/21336731e0cbcca340a5560730ac28640460265f/CONTRIBUTING.md#contributor-license-agreements
- Knative does not have fuzzing on every pull request, however a fuzzing audit was performed on Knative a couple of months ago
    - https://knative.dev/blog/events/fuzzing-audit-2023/
- At least one reviewer is required for a pull request to be approved
    - https://github.com/knative/community/blob/main/REVIEWING.md 
- Code owners must have multiple contributions, have active participation in the project, and be subscribed to the Knative development mailing list. Additionally, one must be nominated by two Knative Members (at least one of whom does not work for the same employer) in order to become a member.
    - https://github.com/knative/community/blob/21336731e0cbcca340a5560730ac28640460265f/ROLES.md#member
- Knative’s release process is not automated
- Every release does not include an automatically generated Software Bill of Materials. However, a request for such a feature has been made 3 weeks ago
    - https://github.com/oracle/graal/issues/7729
- Releases are signed with cosign
    - https://knative.dev/docs/reference/security/
- Container images are immutable after a change has been made to its configuration
    - https://github.com/knative/docs/blob/385ecbdc436fdd54c25e1add58527f247b6a8ce8/docs/serving/services/README.md


### Communication Channels
This is a public Google group that Knative users can join for announcements and communication: https://groups.google.com/g/knative-users 

This is a private Google group that developers in the Knative community can request to join to communicate announcements with each other: https://groups.google.com/g/knative-dev 

The CNCF Slack has several dedicated Knative channels that developers can use for ease of communication: https://github.com/knative/community/blob/main/SLACK-GUIDELINES.md 

The Knative community has monthly meetings where development and project groups are discussed. A public calendar can be found here: https://github.com/knative/community/blob/main/CALENDAR.MD 
### Ecosystem
Knative is built on the existing Kubernetes framework and provides components to build and run serverless applications on Kubernetes. Other projects in the ecosystem like Cloud Run are built on the Knative framework. 
## Security Issue Resolution
Official Knative security policy https://knative.dev/docs/reference/security/ 
### Responsible Disclosure Practice
Any user that finds a security issue can send a detailed email outlining the issue to security@knative.team. The team will respond within 3 business days and will determine whether the report constitutes a vulnerability, at which point they will open a security advisory.  (https://github.com/knative/community/blob/main/SECURITY.md) Certain parties can request to be notified of security vulnerabilities early. (https://github.com/knative/community/blob/main/SECURITY.md) However this is intended for close Knative partners, not individuals. 
### Incident Response
The response policy is outlined here: https://github.com/knative/community/blob/main/working-groups/security/responding.md 

A small team of Knative developers is included in the security@knative.team mailing. This team is kept small to avoid excessive disclosure of vulnerabilities. Each reported security issue is assigned a leader based on a rota that every team member must sign up for. This leader is responsible for determining whether the issue is indeed a vulnerability, in which case they will create a Github Security Advisory. All vulnerabilities are evaluated based on the Knative Threat Model, a work-in-progress model that can be viewed here: https://github.com/knative/community/blob/main/working-groups/security/threat-model.md The vulnerability is also given a Common Vulnerability Scoring System (CVSS) based on a calculator to help evaluate urgency. 
## Appendix

*Known issues over time*

The dev team at Knative is continuously working towards making the platform better and more convenient for the end users. They have found certain issues which they are working on. Some of them include - 

- Internal Encryption on/off requires the activator to be restarted - Service should continue to serve ingress traffic, however instead the client receives 503 and the service stops serving ingress traffic - https://github.com/knative/serving/issues/13754
- If k8s opens IPv4/IPv6 dual-stack feature, Knative returns error: "Service is invalid: spec.ipFamily: Required value" - https://github.com/knative/serving/issues/9045
- The conformance test ContainerExitingMsg is very flaky and currently dependant on a custom progressDeadline - https://github.com/knative/serving/issues/13465
- TestHTTPProbeAutoHTTP2 is quite flaky currently - https://github.com/knative/serving/issues/10962
multiple knative intermittents reconcile failure were observed, that is the controller met some problems in-middle, and mark the knative service status as "Failed". But given the controllers will do retry, so the knative service itself becomes "ready=true" in the end.- https://github.com/knative/serving/issues/10511

*CII Best Practices*

The Knative project has achieved the passing level criteria and is in the process of working towards attaining a silver badge in Open Source Security Foundation (OpenSSF) best practices badge - https://www.bestpractices.dev/en/projects/5913

*Case studies* 

Knative has been adopted by multiple organizations. Some of the cases studies are as follows:  
- PNC Bank, one of the largest U.S. banks, faced a significant challenge with its manual 30-day compliance review process for new code, hindering efficient code deployment. Leveraging Knative, the cloud-native serverless and eventing framework, PNC developed internal tools, including the sophisticated Policy-as-Code service. This service, utilizing Knative's eventing and serverless capabilities, automates code compliance checks, providing an immediate pass/fail status for developers. By bridging processes between Apache Kafka and CI/CD toolchain events, PNC streamlined deployment, replacing a time-consuming manual process with near real-time compliance checks. The modular architecture enabled by Knative and TriggerMesh improved efficiency, allowing PNC to handle a vast technical infrastructure, with 6,000 application components benefiting from the automated compliance process. The result is faster code deployment, elimination of manual compliance delays, and enhanced productivity for PNC's large development teams.
https://www.cncf.io/case-studies/pnc-bank/
- Relay - Puppet, a software company specializing in infrastructure automation, utilized Knative to create Relay, a cloud-native workflow automation platform. Facing challenges in managing modern cloud-native applications due to manual workflows, Puppet leveraged Knative and Kubernetes to develop Relay, an event-driven DevOps tool. Knative Serving enabled Relay to incorporate serverless applications and functions, providing scalability and flexibility. Containers and webhooks are central to Relay's architecture, allowing businesses to deploy workflows as discrete, self-contained units. The platform's open architecture supports integration with various services and platforms. Relay functions as a comprehensive infrastructure management tool, offering a serverless microservice environment powered by Knative, resulting in efficient, cost-effective, and automated business processes for diverse workload management needs across multiple cloud service providers. The success of Relay's development and deployment in just three months underscores Knative's role in streamlining and enhancing Puppet's infrastructure automation capabilities. - https://github.com/knative/docs/blob/main/docs/about/case-studies/puppet.md


*Related projects/vendors*
- Volcano: Both Volcano and Knative are designed to work seamlessly with Kubernetes, leveraging its orchestration capabilities. But, Volcano is tailored for batch processing workloads, while Knative is focused on serverless workloads and event-driven architectures. Knative has gained broader adoption and has a more extensive scope, covering serverless application deployment and event-driven architectures, making it a more versatile choice for a variety of cloud-native applications.
- Emissary-Ingress: Emissary-Ingress primarily focuses on traffic management and routing within Kubernetes, with an emphasis on integrating with different service mesh solutions while Knative provides a broader set of features for serverless application deployment, event-driven architectures, and build automation.
- OpenKruise - OpenKruise is focused on providing additional controllers for Kubernetes to enhance resource management and application deployment. It provides controllers that offer more granular control over various aspects of application lifecycle management within Kubernetes. However, Knative is focused on providing a higher-level serverless platform on Kubernetes, abstracting away much of the infrastructure management. Knative abstracts away many of the complexities associated with deploying and managing serverless applications on Kubernetes.



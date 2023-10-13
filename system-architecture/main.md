### **Notes of System Architecture**

1. [Introduction]()
   - [Definition of System Architecture]()
   - [Components and Connectors]()
   - [Layers and Tiers]()
   - [Notion of Structure in System Architecture]()
   - [Boundary of Reponsibilities]()
  
## **1. Introduction**
### Definition of System Architecture

**System Architecture**: At its essence, system architecture is the structured framework employed to conceptualize software elements, relationships, and behaviors of a system or systems. It's the high-level structuring of a system's components, their interrelationships, and the principles governing their design and evolution over time.

**Purpose and Benefits**:
- **Guidance**: It provides a comprehensive blueprint for developers, ensuring coherence with the organization's objectives.
- **Consistency**: Promotes uniformity in design, ensuring that the system remains robust and sustainable.
- **Efficiency**: Facilitates early detection of potential issues, aiding in cost reduction and swift development.
- **Communication**: Establishes a common language for stakeholders, ensuring transparency and clear dialogue.

### Components and Connectors

**Components and Connectors**: In the realm of system architecture, a "component" represents a modular and deployable part of a system that encapsulates specific functionality or behavior. Components can be anything from software modules, classes, or subsystems to physical elements like servers or databases. Connectors, on the other hand, are the mechanisms that enable interaction, communication, and data flow between components.

**Significance**:
- **Modularity**: Components allow for modularity, promoting reusability, maintainability, and adaptability.
- **Interoperability**: Connectors ensure that disparate components can effectively communicate and work together.

### Layers and Tiers

**Layers** in system architecture represent a logical division of functionalities within an application or system. Typically, systems are divided into three primary layers:

1. **Presentation Layer**: This is the user interface, where users interact with the system. It's responsible for displaying data to the user and interpreting user commands.

2. **Business Logic Layer (or Domain Layer)**: Contains the core functional logic of the system. It interprets commands from the presentation layer, makes logical decisions and evaluations, and performs calculations. 

3. **Data Access Layer**: Acts as a bridge between the business logic and data storage, ensuring data is correctly saved and retrieved from databases or other storage mechanisms.

**Tiers**, on the other hand, refer to the physical separation of the responsibilities of the layers. They define on which server or infrastructure component a particular layer runs. Systems can be:

1. **Single-tier**: Everything runs on one machine (like many traditional desktop applications).
   
2. **Two-tier (Client-Server)**: Presentation layer runs on the client's machine, and the server machine hosts both business logic and data access layers.

3. **Three-tier**: Each layer runs on a separate machine. This is common in web applications where a web server handles the presentation, an application server manages the business logic, and a database server controls the data.

4. **N-tier**: More complex systems might split out functionality even further, for example, separating business logic into microservices running on different servers.

**Benefits and Challenges**:

- **Flexibility**: Separating concerns via layers and tiers provides flexibility. For example, updating the user interface without touching the underlying business logic.
  
- **Scalability**: Different tiers can be scaled independently based on demand.
  
- **Maintenance**: Logical separation makes maintenance easier, but physical separation (tiers) can introduce complexity in deployment and communication.

### Notion of Structure in System Architecture

System architecture's structure is about understanding and organizing the components of a system and how they interact. This encompasses various dimensions:

1. **Functional Structure**:
   - **Definition**: Focuses on the system's capabilities, the operations, functionalities, and tasks the system performs.
   - **Example**: In a banking application, the functional structure encompasses distinct operations like funds transfer, balance inquiries, and loan applications. Each of these functionalities represents a specific capability that the system offers to its users.

2. **Physical Structure**:
   - **Definition**: Deals with the tangible, physical components of a system - the actual hardware or software components and their arrangement.
   - **Example**: For a cloud-based CRM application, the physical structure might consist of web servers that handle user requests, application servers that process the business logic, databases that store customer data, and Content Delivery Networks (CDNs) that ensure fast delivery of content to users worldwide.

3. **Behavioral Structure**:
   - **Definition**: Describes the dynamic aspects of the system, how the system behaves and reacts over time, and in response to various inputs or conditions.
   - **Example**: Consider a smart home system. The behavioral structure involves sensors detecting changes in the environment (like motion or temperature), the system processing this information, and then reacting accordingly, such as by adjusting room temperature or turning on lights.

4. **Temporal Structure**:
   - **Definition**: Pertains to how the system's operations relate to time, including aspects like synchronous and asynchronous operations, cycles, or event-driven behaviors.
   - **Example**: In a chat application, sending a message might be asynchronous: users can send multiple messages without waiting for each to be delivered. However, when a message arrives, the application might produce a synchronous notification, alerting the user in real-time.

The notion of structure in system architecture provides a holistic understanding of the system's design, ensuring the system is robust, adaptable, and aligned with both technical and business objectives.

### Boundary of Reponsibilities

| **Foundational Concepts**    | **System Architects**                                                                                     | **Developers**                                                                                                           | **Product Managers**                                                                                                             |
|----------------------|--------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| **Definition of System Architecture** | Define high-level structure; Ensure alignment with technical & business requirements.   | Implement based on architecture blueprint; Adhere to defined structural principles.                                 | Provide insights on user needs and business requirements; Ensure system aligns with business and market objectives. |
| **Components & Connectors**  | Determine main components and connectors; Design interactions & specify interfaces.     | Build and code components; Ensure proper inter-component communication.                                             | Provide input on user experience and business logic; Ensure components align with user needs.                        |
| **Layers & Tiers**            | Define functional layers; Decide number of physical tiers based on scalability & performance. | Implement logic for layers; Deploy to appropriate tiers; Manage inter-layer communications.                    | Offer insights on user experience for presentation layer; Ensure business logic layer aligns with business objectives. |
| **Notion of Structure**       | Outline functional, physical, behavioral, and temporal structures.                          | Bring to life the defined structures through code and configuration; Ensure dynamic behaviors are implemented correctly. | Bridge gap between technical design and user needs; Guide feature incorporation based on feedback and market trends.   |


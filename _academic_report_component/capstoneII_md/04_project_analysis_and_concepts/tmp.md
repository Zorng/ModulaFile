## 4.3 Systems Users and Behavioral Modeling

This subsection analyzes the system from a user and behavioral perspective by identifying system actors and illustrating their interactions through use case and activity diagrams.

### 4.3.1 System Users

This subsection identifies the primary users of the Modula system and analyzes their roles, responsibilities, and access boundaries. Clearly defining system users is essential to ensure that functionalities, permissions, and workflows are aligned with real operational practices in Food & Beverage (F&B) businesses.

Modula adopts a role-based user model to balance operational efficiency, security, and accountability. Each user role is associated with a specific set of responsibilities and permissions, enforced by the authentication and authorization mechanism of the system.

-   Admin: The admin represents the business owner or a trusted individual responsible for overall system configuration and oversight.
    -   Responsibilities: 
        -   Configure tenant-level settings such as policies related to tax, currency, inventory behavior, attendance, and cash handling.
        -   Manage core operational data including menu items, inventory items, and branch information.
        -   Create and manage staff accounts and assign roles.
        -   View system-wide records such as sales reports, inventory reports, attendance logs, and cash session summaries.
        -   Approve sensitive operations such as sale void requests, cash refunds, and out-of-shift attendance when required.
    -   Permission:
        -   Full access to all system modules within the tenant context.
        - Read and write access to administrative features.
        - Approval authority for restricted or high-risk actions.
-  Manager: The Manager role represents supervisory staff responsible for overseeing daily operations at one or more assigned branches.
   -  Responsiblities:
       - Monitor sales, orders, and inventory status within assigned branches.
       - Approve operational requests such as sale voids, cash refunds, or out-of-shift attendance, depending on policy settings.
       - Review attendance records and cash session summaries for assigned branches.
       - Support cashiers in resolving operational issues during shifts.
   - Permission:
       - Limited administrative access scoped to assigned branches.
       - Approval privileges for selected actions as defined by policy.
       - Read-only access to most configuration settings.
- Staff: The Cashier role represents frontline staff responsible for executing sales and basic operational tasks.
    - Responsiblities:
      - Perform sales transactions by selecting menu items, modifiers, and sale types.
      - Manage active orders, including updating order statuses.
      - Start and close cash sessions when required by policy.
      - Record attendance through check-in and check-out actions.
      - View personal attendance history and assigned work shifts.
    - Permission:
      - Access to sales and order-related functionalities.
      - Restricted access to configuration and reporting features.
      - No authority to modify finalized sales or system policies.
  
By clearly separating responsibilities and permissions among Admin, Manager, and Cashier roles, Modula ensures operational clarity, security, and traceability. This role-based model forms the foundation for subsequent analysis of use cases and activity workflows in the following sections.

### 4.3.2 Usecase Diagram

To provide an overall understanding of how users interact with the Modula POS system, a single high-level use case diagram is presented. Rather than enumerating every individual system operation, this diagram focuses on the primary goals and responsibilities of users when operating the system.
The diagram abstracts internal system processes and emphasizes the relationship between user roles and core system capabilities. Common interactions, such as authentication, are shared across user roles, while management and approval-related use cases are associated only with users holding higher privileges. This approach reflects Modula’s role-based access model without introducing implementation-level complexity.
By presenting a consolidated, high-level view, the use case diagram establishes a conceptual foundation for the system’s functional scope. More detailed workflows and operational behavior are subsequently illustrated through activity diagrams and module-level descriptions in later sections.

//OLD USECASE DIAGRAM HERE//

//Diagram Description//
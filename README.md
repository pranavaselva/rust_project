# rust_project

Architecture Overview
This project utilizes a decoupled, three-tier architecture designed for high performance and strict type safety.

Frontend (Angular): The user interface is built with Angular components. These components capture user events (like form submissions or button clicks) and delegate data handling to Angular Services. These services use the HttpClient module to communicate with the backend via RESTful APIs.

Backend (Rust): The server is powered by Rust (using the Axum/Actix framework). Rust provides a high-performance, memory-safe environment to process business logic. It receives HTTP requests, validates the data, and interacts with the database.

Database (PostgreSQL): All persistent data is stored in PostgreSQL. Rust communicates with the database using a driver or ORM (like sqlx or Diesel), ensuring that data remains consistent and structured.

Reflection: Why Separation Matters
Separating the frontend and backend is crucial for scalability and maintainability. By decoupling the two, the backend can be optimized for heavy processing without affecting the user experience, while the frontend can be updated or redesigned independently. Additionally, the combination of TypeScript on the frontend and Rust on the backend provides end-to-end type safety, significantly reducing runtime errors.

(![Flow Diagram](<rust image.png>))
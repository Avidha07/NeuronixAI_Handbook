# Running Backend Before Postman (NeuronixAI)

- If I closed IntelliJ or didn't run the project since yesterday → Run the Spring Boot application first.
- Wait until the console shows:
  - `Tomcat started on port 8080`
  - `Started NeuronixAiApplication`
- Then open Postman.
- First call: `POST /api/v1/auth/login` to get a Bearer Token.
- Use that token for all protected APIs (Create, Get, Delete Conversation).
- No need to restart the backend for every Postman request unless the server stops or I change backend code.

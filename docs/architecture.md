# System Architecture Diagrams

## Class Diagram

```mermaid
classDiagram
    class User {
        - id : int
        - username : str
        - email : str
        - password_hash : str
        - focus_streak : int
        - daily_target : int
        + register() : JWT
        + login() : JWT
    }
    
    class Task {
        - id : int
        - title : str
        - subject : str
        - pomodoros_est : int
        - pomodoros_done : int
        - user_id : int FK
        + create() : Task
        + increment_pomodoro()
    }
    
    class Quiz {
        - id : int
        - topic : str
        - questions : JSON
        - score : int
        - user_id : int FK
        - created_at : datetime
        + generate(topic) : Quiz
        + submit(answers) : int
    }
    
    class StudyLog {
        - id : int
        - activity_type : str
        - duration_mins : int
        - user_id : int FK
    }
    
    class AIEngine {
        - heuristics : dict
        + gen_quiz(topic)
        + chat(query) : str
        + recommend() : list
    }
    
    User "1" *-- "0..*" Task
    User "1" o-- "0..*" Quiz
    StudyLog ..> User : logs
    Quiz ..> AIEngine : <<uses>>
```

## Authentication Sequence Diagram

```mermaid
sequenceDiagram
    participant Browser
    participant Flask API
    participant SQLite DB
    participant JWT
    
    Note over Browser, JWT: Register
    Browser->>Flask API: POST /register {username, password}
    Flask API->>Flask API: hash password
    Flask API->>SQLite DB: INSERT user row
    SQLite DB-->>Flask API: user_id
    Flask API->>JWT: sign({user_id})
    JWT-->>Flask API: JWT token
    Flask API-->>Browser: 201 Created + JWT
    Browser->>Browser: store in localStorage
    
    Note over Browser, JWT: Login
    Browser->>Flask API: POST /login {email, password}
    Flask API->>SQLite DB: SELECT WHERE email=?
    SQLite DB-->>Flask API: user row
    Flask API->>Flask API: bcrypt.check(pwd, hash)
    Flask API->>JWT: sign({user_id})
    JWT-->>Flask API: JWT token
    Flask API-->>Browser: 200 OK + profile
    
    Note over Browser, JWT: Protected API request
    Browser->>Flask API: GET /api/tasks Authorization: Bearer JWT
    Flask API->>JWT: verify(token)
    JWT-->>Flask API: user_id
    Flask API->>SQLite DB: SELECT tasks WHERE user_id=?
    SQLite DB-->>Flask API: tasks[]
    Flask API-->>Browser: 200 JSON tasks
```

## Pomodoro Workflow Sequence Diagram

```mermaid
sequenceDiagram
    participant Browser
    participant Flask API
    participant Pomodoro timer
    participant SQLite DB
    
    Note over Browser, SQLite DB: Create task
    Browser->>Flask API: POST /api/tasks {title, subject, est}
    Flask API->>SQLite DB: INSERT task row
    SQLite DB-->>Flask API: task_id
    Flask API-->>Browser: 201 Created + task
    
    Note over Browser, SQLite DB: Start Pomodoro (25 min)
    Browser->>Flask API: POST /api/tasks/{id}/start
    Flask API->>Pomodoro timer: startTimer(task_id)
    Pomodoro timer-->>Flask API: timer started
    Flask API-->>Browser: 200 OK
    Pomodoro timer->>Pomodoro timer: 25-min countdown
    
    Note over Browser, SQLite DB: Pomodoro complete
    Pomodoro timer->>Flask API: onComplete(task_id)
    Flask API->>SQLite DB: UPDATE task SET pomodoros_done+1
    Flask API->>SQLite DB: INSERT study_log (task, 25 min)
    Flask API->>SQLite DB: UPDATE user SET streak+1
    SQLite DB-->>Flask API: OK
    Flask API-->>Browser: pomodoro logged
    
    Note over Browser, SQLite DB: Mark task complete (optional)
    Browser->>Flask API: PATCH /api/tasks/{id} {done: true}
    Flask API->>SQLite DB: UPDATE task SET status=done
    SQLite DB-->>Flask API: OK
    Flask API-->>Browser: 200 task updated
```

## Quiz Workflow Sequence Diagram

```mermaid
sequenceDiagram
    participant Browser
    participant Flask API
    participant AIEngine
    participant SQLite DB
    
    Note over Browser, SQLite DB: Generate quiz
    Browser->>Flask API: POST /api/quiz/generate {topic}
    Flask API->>AIEngine: gen_quiz(topic, n=5)
    AIEngine->>AIEngine: build 5 MCQ questions
    AIEngine-->>Flask API: questions[]
    Flask API->>SQLite DB: INSERT quiz (topic, questions)
    SQLite DB-->>Flask API: quiz_id
    Flask API-->>Browser: 200 quiz + questions
    Browser->>Browser: student answers questions
    
    Note over Browser, SQLite DB: Submit answers
    Browser->>Flask API: POST /api/quiz/{id}/submit {answers[]}
    Flask API->>AIEngine: grade(answers, answer_key)
    AIEngine->>AIEngine: compute score
    AIEngine-->>Flask API: score, feedback[]
    Flask API->>SQLite DB: UPDATE quiz SET score=?
    Flask API->>SQLite DB: INSERT study_log (quiz, 15 min)
    SQLite DB-->>Flask API: OK
    Flask API-->>Browser: 200 score + feedback
    Browser->>Browser: render results to student
```

## AI Chat Workflow Sequence Diagram

```mermaid
sequenceDiagram
    participant Browser
    participant Flask API
    participant AIEngine
    participant SQLite DB
    
    Note over Browser, SQLite DB: Student sends query
    Browser->>Flask API: POST /api/chat {query, history[]}
    Flask API->>AIEngine: chat(query, history)
    AIEngine->>AIEngine: parse intent<br/>select heuristic<br/>build response
    AIEngine-->>Flask API: reply text
    
    Note over Browser, SQLite DB: Log study interaction
    Flask API->>SQLite DB: INSERT study_log (chat, 1 min)
    SQLite DB-->>Flask API: OK
    
    Note over Browser, SQLite DB: Return response
    Flask API-->>Browser: 200 {reply, suggestions[]}
    Browser->>Browser: render reply in chat UI
    
    Note over Browser, SQLite DB: Feynman follow-up loop (optional)
    Browser->>Browser: student asks follow-up
    Browser->>Browser: repeats from top
```

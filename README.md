
# Smart Task Analyzer – Backend (Django)

This is a small Django REST API that helps users decide **which tasks to do first.

Given a list of tasks with due dates, importance, effort and dependencies, the API:

- Calculates a priority score** for each task  
- Returns the list sorted by priority  
- Suggests the **top 3 tasks to work on today** with explanations

1. Tech Stack

- Python 3.x  
- Django  
- Django REST Framework

 2. Project Structure (Backend)

text
backend/
├── manage.py
├── requirements.txt
├── task_analyzer/
│   ├── __init__.py
│   ├── settings.py
│   ├── urls.py
│   └── wsgi.py
└── tasks/
    ├── __init__.py
    ├── scoring.py
    ├── serializers.py
    ├── views.py
    ├── models.py   # optional, for DB structure only
    └── tests.py    # optional unit tests


 3. Setup Instructions

1. Clone / open the project**

   bash
   cd backend
  

2. reate & activate virtual environment**

  bash
   python -m venv venv

   # Windows
   venv\Scripts\activate

   macOS / Linux
   source venv/bin/activate
   

3. Install dependencies**

   
   pip install -r requirements.txt
   

4. **Run migrations** (only needed if using the Task model)

   bash
   python manage.py makemigrations
   python manage.py migrate
   

5. **Start the development server**

   bash
   python manage.py runserver
   

The API will be available at:


http://127.0.0.1:8000/


 4. Task Data Structure

Each task is represented as a JSON object with the following fields:

```json
{
  "title": "Fix login bug",
  "due_date": "2025-11-30",
  "estimated_hours": 3,
  "importance": 8,
  "dependencies": []
}
```

* `title` (string, required) – short task name
* `due_date` (YYYY-MM-DD, optional) – when the task is due
* `estimated_hours` (number, optional)– rough time estimate
* `importance` (1–10, required) – how important the task is
* `dependencies` (list of task IDs, optional) – tasks that must be completed first

If `id` is not provided, the backend will auto-generate IDs.


 5. Priority Algorithm (How Scoring Works)

The goal is to combine several factors into one **priority score**.

The algorithm considers:

1. Urgency (due date)**

   * Past-due tasks get the highest urgency score.
   * Tasks due today or in the next few days get higher scores than tasks far in the future.

2. Importance (1–10)

   * Direct user rating.
   * Higher importance → higher contribution to the final score.

3. Effort (estimated hours)

   * Lower effort tasks are treated as quick wins and get a higher “effort score”.
   * Very long tasks get a lower effort contribution.

4. **Dependencies

    If many tasks depend on a given task, that task gets a dependency boost because it unblocks others.

5. Circular Dependencies

   * Before scoring, the code builds a dependency graph and checks for cycles.
   * If a circular dependency is detected, the API returns an error.

Scoring Formula (high level)

For each task:

* Compute:

  * `urgency_score` from `due_date`
  * `effort_score` from `estimated_hours`
  * `dependency_boost` from dependency graph
* Combine them with importance:


final_score =
    0.4 * importance +
    0.35 * urgency_score +
    0.15 * effort_score +
    0.10 * dependency_boost


The API also returns a human-readable `explanation` string for each task, describing why it got that score (importance, urgency, effort, and whether it unblocks others or is overdue).

Tasks are then sorted in **descending order of `score`.


6. API Endpoints

 6.1 `POST /api/tasks/analyze/`

Accepts a list of tasks and returns them with calculated scores, sorted by priority.

Request

http
POST /api/tasks/analyze/
Content-Type: application/json


json
{
  "tasks": [
    {
      "title": "Fix login bug",
      "due_date": "2025-11-30",
      "estimated_hours": 3,
      "importance": 8,
      "dependencies": []
    },
    {
      "title": "Write unit tests",
      "due_date": "2025-12-02",
      "estimated_hours": 2,
      "importance": 7,
      "dependencies": [1]
    }
  ]
}


Success Response (200)

json
[
  {
    "id": 1,
    "title": "Fix login bug",
    "due_date": "2025-11-30",
    "estimated_hours": 3.0,
    "importance": 8,
    "dependencies": [],
    "score": 9.2,
    "explanation": "Importance=8/10. Urgency score=8.0. Effort score=8.0."
  },
  {
    "id": 2,
    "title": "Write unit tests",
    "due_date": "2025-12-02",
    "estimated_hours": 2.0,
    "importance": 7,
    "dependencies": [1],
    "score": 8.7,
    "explanation": "Importance=7/10. Urgency score=6.0. Effort score=8.0. Unblocks other tasks (dependency boost 1.0)."
  }
]


Error Cases

* Missing `tasks` field
* Invalid data (e.g., importance not between 1 and 10)
* Circular dependencies between tasks



6.2 `GET /api/tasks/suggest/`

Returns the **top 3 tasks** from the most recent `/analyze/` call.

Request

http
GET /api/tasks/suggest/


Success Response (200)**

json
[
  {
    "id": 1,
    "title": "Fix login bug",
    "due_date": "2025-11-30",
    "estimated_hours": 3.0,
    "importance": 8,
    "dependencies": [],
    "score": 9.2,
    "explanation": "..."
  },
  {
    "id": 2,
    "title": "Write unit tests",
    "due_date": "2025-12-02",
    "estimated_hours": 2.0,
    "importance": 7,
    "dependencies": [1],
    "score": 8.7,
    "explanation": "..."
  }
]


**Error Response (if no analysis yet)**

json
{
  "detail": "No analyzed tasks yet. Call /api/tasks/analyze/ first."
}


> Note: For simplicity, the API stores the last analysis result **in memory** (`LAST_ANALYZED`).
> In a production system, this would live in a database or cache.


 7. How This Can Be Used From a Frontend

A frontend (HTML/CSS/JS, React, etc.) can:

1. Collect tasks from the user (forms or JSON input).
2. Send them to `POST /api/tasks/analyze/` to get scores and explanations.
3. Display the sorted tasks in a table or cards.
4. Call `GET /api/tasks/suggest/` to show the top 3 recommended tasks to work on today.


 8. Running Tests (optional)

If you add unit tests in `tasks/tests.py`, run:


python manage.py test


9. Summary

* This backend focuses on **task prioritization**, not on user accounts or persistence.
* The main value is the **priority algorithm** combining urgency, importance, effort and dependencies.
* The API is small but easy to hook into any frontend.

You can now plug this backend into your web app and build the UI on top of these endpoints.


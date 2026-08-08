# SQL Injection Detection & Prevention System

A Flask-based web application that detects and blocks SQL Injection attempts in real time. The system inspects login form inputs against a set of pattern-matching rules, blocks malicious requests, logs attack attempts to a database, and sends email alerts to configured recipients.

## Features

- **Login form protection** — validates `username` and `password` fields before they ever reach a SQL query.
- **Pattern-based SQL Injection detection** — flags common attack signatures, including:
  - Boolean-based tautologies (e.g. `OR 1=1`)
  - SQL comment sequences (`--`, `#`, `/* */`)
  - Dangerous keywords (`SELECT`, `INSERT`, `UPDATE`, `DELETE`, `DROP`, `UNION`, `ALTER`, `EXEC`)
  - `UNION ... SELECT` injection chains
  - Statement terminators (`;`)
  - Time-based blind injection (`SLEEP()`, `WAITFOR`)
  - Information schema probing (`INFORMATION_SCHEMA`)
  - Hex-encoded payloads (`0x...`)
  - Repeated/escaped quote characters
  - Abnormally long input (>50 characters)
- **Whitelist input validation** — a secondary check (`is_safe_input`) restricts input to a safe character set (letters, numbers, `_`, `@`, `.`, space).
- **Attack logging** — every blocked attempt is written to an `attack_logs` table with the offending input and timestamp.
- **Email alerts** — sends a real-time email notification (via Gmail SMTP) to configured recipients whenever an injection attempt is detected, including the submitted input and the attacker's IP address.
- **Admin dashboard** — displays a live count of detected attacks and a full history of logged attempts, most recent first.
- **Parameterized queries** — legitimate login checks use parameterized SQL to prevent injection at the query level as a second layer of defense.

## Tech Stack

| Layer          | Technology                     |
|----------------|---------------------------------|
| Backend        | Python, Flask                  |
| Database       | MySQL                          |
| Frontend       | HTML, CSS                      |
| Detection      | Regex (`re` module)            |
| Alerting       | SMTP (Gmail) via `smtplib`     |

## Project Structure

```
.
├── app.py               # Flask app: routes, detection logic, alerting
├── db_config.py         # MySQL connection configuration
├── SQL database.sql     # Database schema and seed data
├── templates/
│   ├── login.html       # Login form (referenced by app.py, not included here)
│   └── dashboard.html   # Attack log dashboard (referenced by app.py, not included here)
└── README.md
```

## Database Schema

The application uses a database named `security_project` with two tables:

**`users`**
| Column   | Type         |
|----------|--------------|
| id       | INT, AUTO_INCREMENT, PRIMARY KEY |
| username | VARCHAR(50)  |
| password | VARCHAR(50)  |

**`attack_logs`**
| Column     | Type                                   |
|------------|-----------------------------------------|
| id         | INT, AUTO_INCREMENT, PRIMARY KEY        |
| input_text | TEXT                                    |
| timestamp  | DATETIME, DEFAULT CURRENT_TIMESTAMP     |

## Setup & Installation

### 1. Clone the project and install dependencies
```bash
pip install flask mysql-connector-python
```

### 2. Set up the database
Run the provided SQL script in MySQL:
```bash
mysql -u root -p < "SQL database.sql"
```
This creates the `security_project` database along with the `users` and `attack_logs` tables, and inserts a default user (`admin` / `admin123`).

### 3. Configure the database connection
Edit `db_config.py` with your MySQL credentials:
```python
def get_connection():
    return mysql.connector.connect(
        host="localhost",
        user="root",
        password="your_password",
        database="security_project"
    )
```

### 4. Configure email alerts
In `app.py`, update the `send_alert()` function with your own sender/receiver email addresses.

> ⚠️ **Security note:** Do not hardcode email credentials directly in the source file. Use environment variables (e.g. via `os.environ` and a `.env` file with `python-dotenv`) and a Gmail **App Password**, not your main account password. If credentials have ever been committed or shared, revoke and regenerate them immediately.

### 5. Run the application
```bash
python app.py
```
The app will automatically open in your browser at `http://127.0.0.1:5000`.

## How It Works

1. A user submits the login form with a username and password.
2. Both inputs are checked against:
   - A whitelist regex (`is_safe_input`) that only allows safe characters.
   - A blacklist of SQL injection patterns (`detect_sql_injection`).
3. **If malicious input is detected:**
   - The request is blocked immediately.
   - An email alert is sent with the offending input and requester's IP address.
   - The attempt is logged to the `attack_logs` table.
4. **If input passes validation:**
   - A parameterized SQL query checks the credentials against the `users` table.
   - On success, the user is redirected to `/dashboard`.
5. The `/dashboard` route displays the total attack count and a full log of all detected attempts.

## Future Scope

- Integrate **machine learning / AI-based detection** for higher-accuracy, adaptive SQL Injection detection beyond static regex patterns.
- Add **real-time monitoring** and **advanced threat analysis**.
- Build **graphical dashboards** with charts/visualizations of attack trends.
- Support **multi-user authentication** with role-based access.
- Add **cloud database integration**.
- Extend detection to other attack types such as **Cross-Site Scripting (XSS)** and **Brute Force attacks**.

## Known Limitations

- Passwords in the `users` table are stored in plaintext — should be hashed (e.g. with `bcrypt`) in a production system.
- Email credentials are currently hardcoded in `app.py` — should be moved to environment variables.
- Detection relies on regex pattern matching, which can produce false positives/negatives compared to ML-based approaches.

## License

This project was developed for educational/academic purposes as part of a seminar/security project.

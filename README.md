# AWS Step Functions — Class Attendance Processor

A serverless attendance management system built with **AWS Lambda** and **AWS Step Functions** that automates student attendance tracking. This project processes images sent from an Arduino-based capture device, validates student recognition, and manages attendance records in **Amazon DynamoDB**.

This project was developed as a college study on serverless architectures, AWS Step Functions orchestration, and automated attendance processing.

---

## Table of Contents

- [How It Works](#how-it-works)
- [Architecture](#architecture)
- [Project Structure](#project-structure)
- [Dependencies](#dependencies)
- [Environment Variables](#environment-variables)
- [Getting Started](#getting-started)
- [Lambda Functions](#lambda-functions)
- [Step Functions Workflow](#step-functions-workflow)
- [DynamoDB Tables](#dynamodb-tables)
- [Related Projects](#related-projects)
- [Credits](#credits)

---

## How It Works

1. A **scheduler Lambda** checks a DynamoDB calendar table to determine if there is a class scheduled for today.
2. If a class is found, it calculates the wait time until the class starts (plus a 30-minute buffer for attendance).
3. **AWS Step Functions** orchestrates the flow — waiting until class time using the computed wait time.
4. After the class period ends, a **processor Lambda** scans the student table and marks students as absent if no attendance record was registered for that day.
5. Attendance is originally registered when a student is recognized by the image processing system (fed by photos captured from the Arduino device).

---

## Architecture

```
┌──────────────────────────┐
│  Arduino Image Capture   │  (companion project)
│  study-imagine-capture-  │
│  arduino                 │
└───────────┬──────────────┘
            │ HTTP POST (image upload)
            ▼
┌──────────────────────────┐
│    REST API Endpoint     │
│  (receives & processes   │
│   student photos)        │
└───────────┬──────────────┘
            │
            ▼
┌─────────────────────────────────────────────┐
│            AWS Step Functions               │
│                                             │
│  ┌─────────────────┐   ┌─────────────────┐ │
│  │   Scheduler      │──▶│    Wait Time     │ │
│  │   (Lambda)       │   │   (Step Fn wait) │ │
│  └─────────────────┘   └────────┬────────┘ │
│                                  │          │
│                                  ▼          │
│                        ┌─────────────────┐ │
│                        │   Processor      │ │
│                        │   (Lambda)       │ │
│                        └─────────────────┘ │
└──────────────────────────┬──────────────────┘
                           │
                           ▼
                  ┌─────────────────┐
                  │   DynamoDB      │
                  │                 │
                  │  • Calendario   │
                  │  • Aluno        │
                  └─────────────────┘
```

---

## Project Structure

```
study-lambda-step-function-AWS/
├── classcheckscheduler.py    # Lambda — checks class schedule & calculates wait time
├── classcheckprocessor.py    # Lambda — marks absent students after class ends
├── .gitignore                # Python + IDE ignore rules
└── README.md                 # This file
```

---

## Dependencies

| Package | Purpose |
|---------|---------|
| [boto3](https://boto3.amazonaws.com/) | AWS SDK for Python — DynamoDB access |
| [pytz](https://pypi.org/project/pytz/) | Timezone handling (`America/Sao_Paulo`) |
| [python-dotenv](https://pypi.org/project/python-dotenv/) | Loads environment variables from `.env` file |

> **Note:** These dependencies are required for local testing. When deployed to AWS Lambda, they should be packaged in a deployment layer or bundled in the Lambda deployment zip.

---

## Environment Variables

Create a `.env` file in the project root (it is already gitignored):

```env
REGION=sa-east-1
TABLE_NAME_CALENDARIO=Calendario
TABLE_NAME_ALUNO=Aluno
MY_AWS_ACCESS_KEY_ID=your-access-key-id
MY_AWS_SECRET_ACCESS_KEY=your-secret-access-key
```

| Variable | Description |
|----------|-------------|
| `REGION` | AWS region where DynamoDB tables are hosted |
| `TABLE_NAME_CALENDARIO` | DynamoDB table name for class schedule/calendar |
| `TABLE_NAME_ALUNO` | DynamoDB table name for student records |
| `MY_AWS_ACCESS_KEY_ID` | AWS access key with DynamoDB permissions |
| `MY_AWS_SECRET_ACCESS_KEY` | AWS secret access key |

---

## Getting Started

### Prerequisites

- **Python 3.8+**
- An **AWS account** with DynamoDB and Lambda access
- AWS CLI configured (optional, for deployment)

### Local Setup

1. **Clone the repository:**
   ```bash
   git clone git@github.com:WagnerCaetano/study-lambda-step-function-AWS.git
   cd study-lambda-step-function-AWS
   ```

2. **Create a virtual environment and install dependencies:**
   ```bash
   python -m venv venv
   source venv/bin/activate   # Linux/Mac
   venv\Scripts\activate      # Windows

   pip install boto3 pytz python-dotenv
   ```

3. **Create the `.env` file** with your AWS credentials and table names (see [Environment Variables](#environment-variables)).

4. **Test locally:**
   ```bash
   python classcheckscheduler.py
   python classcheckprocessor.py
   ```

---

## Lambda Functions

### `classcheckscheduler.py` — Scheduler

Checks if there is a class scheduled for today and calculates the wait time until class starts.

**Flow:**
1. Queries the `Calendario` DynamoDB table for today's day of the week (mapped to Portuguese: `segunda`, `terça`, etc.)
2. If no schedule exists → raises `SchedulerBypassAttendanceToday` (Step Function skips the rest)
3. If class already passed today → raises `SchedulerBypassAttendanceToday`
4. Otherwise → returns the class schedule and wait time (in seconds) until class start + 30-minute buffer

**Output:**
```json
{
  "classSchedule": { "lista-dias-aulas": "segunda", "horario": "08:00" },
  "waitTime": 5400
}
```

**Custom Exceptions:**
- `SchedulerBypassAttendanceToday` — raised when there is no class today or class already passed, causing the Step Function to bypass attendance processing

---

### `classcheckprocessor.py` — Processor

Scans all students and marks those without an attendance record for today as absent.

**Flow:**
1. Scans the `Aluno` DynamoDB table for all student records
2. For each student, checks their `historico` (attendance history) for today's date
3. If no entry exists for today → creates a new history entry with `presente: false`
4. Updates the student's record in DynamoDB

**Output:**
```json
{
  "message": "Attendance check completed"
}
```

**Custom Exceptions:**
- `StudentsNotFoundException` — raised when no students are found in the database

---

## Step Functions Workflow

The Step Function state machine orchestrates the attendance flow:

```
[Scheduler] ──▶ [Wait] ──▶ [Processor]
     │                        │
     ▼                        ▼
  (Catch:                  (Catch:
   no class                 no students
   today)                   found)
```

1. **Scheduler** — determines if there is class today and how long to wait
2. **Wait** — pauses execution using the `waitTime` from the scheduler output
3. **Processor** — after waiting, marks absent students
4. **Error handling** — `SchedulerBypassAttendanceToday` and `StudentsNotFoundException` are caught to gracefully handle edge cases

---

## DynamoDB Tables

### Calendario (Calendar)

| Key | Type | Description |
|-----|------|-------------|
| `lista-dias-aulas` (PK) | String | Day of the week in Portuguese (`segunda`, `terça`, `quarta`, `quinta`, `sexta`, `sábado`, `domingo`) |
| `horario` | String | Class time in `HH:MM` format |

### Aluno (Student)

| Key | Type | Description |
|-----|------|-------------|
| `matricula` (PK) | String | Student registration/ID number |
| `historico` | List | Array of attendance records |
| `historico[].data` | String | Date in `DD/MM/YYYY` format |
| `historico[].dia` | String | Day of the week in Portuguese |
| `historico[].hora` | String | Time in `HH:MM` format |
| `historico[].id_historico` | String | Sequential history entry ID |
| `historico[].presente` | Boolean | Whether the student was present |

---

## Related Projects

- **[study-imagine-capture-arduino](https://github.com/WagnerCaetano/study-imagine-capture-arduino)** — The Arduino + Java desktop application that captures student images and sends them to the REST API endpoint. Together, these two projects form the complete attendance system.

---

## Credits

- **Author:** [Wagner Caetano](https://github.com/WagnerCaetano)
- Developed as an academic study on serverless computing, AWS Step Functions, and automated student attendance management.

---

## License

This project is intended for educational and academic purposes.
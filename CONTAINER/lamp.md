# Student Management System with Docker + LAMP

A beginner-friendly full-stack CRUD application using Linux containers,
Apache, PHP, MySQL, Docker Compose, and phpMyAdmin.

## 1. What we are building

The Student Management System supports:

-   Add a student
-   View all students
-   Edit student information
-   Delete a student
-   Search students

### Technology stack

  Component                 Technology
  ------------------------- ----------------------------------
  Operating system          Linux (inside Docker containers)
  Web server                Apache
  Backend                   PHP 8.3
  Database                  MySQL 8.4
  Frontend                  HTML, CSS, Bootstrap 5
  Database administration   phpMyAdmin
  Containerization          Docker Compose

## 2. Architecture

The browser sends requests to Apache. Apache runs PHP, and PHP connects
to MySQL using the Docker Compose service name `db`. phpMyAdmin provides
a separate browser interface for managing MySQL.

``` text
Browser
   |
   v
Apache + PHP  <---->  MySQL
   |
   +---- phpMyAdmin (separate web interface)
```

## 3. Prerequisites

Install Docker Desktop (Windows/macOS) or Docker Engine and Docker
Compose V2 (Linux).

Check the installation:

``` bash
docker --version
docker compose version
```

Docker Desktop: https://www.docker.com/products/docker-desktop/

## 4. Project structure

Create a folder named `student-management` with this structure:

``` text
student-management/
├── docker-compose.yml
├── Dockerfile
├── config/
│   └── database.php
├── database/
│   └── init.sql
└── src/
    ├── index.php
    ├── add.php
    ├── edit.php
    ├── delete.php
    └── style.css
```

## 5. Docker configuration

### `Dockerfile`

``` dockerfile
FROM php:8.3-apache

RUN docker-php-ext-install mysqli
RUN a2enmod rewrite

WORKDIR /var/www/html

RUN chown -R www-data:www-data /var/www/html

EXPOSE 80
```

This builds an Apache web server image with PHP 8.3 and the MySQLi
extension enabled.

### `docker-compose.yml`

``` yaml
services:
  web:
    build: .
    container_name: student_web
    ports:
      - "8080:80"
    volumes:
      - ./src:/var/www/html
      - ./config:/var/www/config:ro
    depends_on:
      db:
        condition: service_healthy
    restart: unless-stopped

  db:
    image: mysql:8.4
    container_name: student_db
    environment:
      MYSQL_ROOT_PASSWORD: root123
      MYSQL_DATABASE: student_db
      MYSQL_USER: student_user
      MYSQL_PASSWORD: student123
    volumes:
      - db_data:/var/lib/mysql
      - ./database/init.sql:/docker-entrypoint-initdb.d/init.sql:ro
    ports:
      - "3307:3306"
    healthcheck:
      test: ["CMD", "healthcheck.sh", "--connect", "--innodb_initialized"]
      interval: 10s
      timeout: 5s
      retries: 10
    restart: unless-stopped

  phpmyadmin:
    image: phpmyadmin:latest
    container_name: student_phpmyadmin
    environment:
      PMA_HOST: db
      PMA_PORT: 3306
    ports:
      - "8081:80"
    depends_on:
      db:
        condition: service_healthy
    restart: unless-stopped

volumes:
  db_data:
```

**Important:** These credentials are for local development only. Change
them and use secrets or environment variables before deploying anywhere
public.

The PHP container connects to MySQL using hostname `db` and port `3306`.
It must not use `localhost`, because MySQL runs in a separate container.

  Service        Container                Host port
  -------------- ---------------------- -----------
  Apache + PHP   `student_web`                 8080
  MySQL          `student_db`                  3307
  phpMyAdmin     `student_phpmyadmin`          8081

## 6. Create the database table

### `database/init.sql`

``` sql
USE student_db;

CREATE TABLE IF NOT EXISTS students (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(150) NOT NULL UNIQUE,
    phone VARCHAR(20),
    course VARCHAR(100) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

The SQL script runs automatically when MySQL initializes a fresh data
volume. It does not automatically rerun every time the container
restarts.

## 7. PHP database connection and shared helpers

### `config/database.php`

``` php
<?php

$host = "db";
$username = "student_user";
$password = "student123";
$database = "student_db";

$conn = new mysqli($host, $username, $password, $database);

if ($conn->connect_error) {
    http_response_code(500);
    die("Database connection failed.");
}

$conn->set_charset("utf8mb4");

if (session_status() === PHP_SESSION_NONE) {
    session_start();
}

function e($value) {
    return htmlspecialchars((string)$value, ENT_QUOTES, 'UTF-8');
}

function csrf_token() {
    if (empty($_SESSION['csrf_token'])) {
        $_SESSION['csrf_token'] = bin2hex(random_bytes(32));
    }

    return $_SESSION['csrf_token'];
}

function verify_csrf() {
    if (
        !isset($_POST['csrf_token']) ||
        !hash_equals($_SESSION['csrf_token'] ?? '', $_POST['csrf_token'])
    ) {
        http_response_code(403);
        exit("Invalid request. Please try again.");
    }
}
```

The `e()` helper escapes output before displaying it in HTML. Prepared
statements are used for database queries, and CSRF tokens protect
state-changing form submissions.

## 8. CSS styling

### `src/style.css`

``` css
body {
    background: #f4f6f9;
    font-family: Arial, sans-serif;
}

.container {
    max-width: 1100px;
    margin: 40px auto;
}

.header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 25px;
}

.card {
    background: white;
    padding: 25px;
    border-radius: 10px;
    box-shadow: 0 2px 10px rgba(0,0,0,0.05);
}

h1 {
    color: #263238;
}

table {
    width: 100%;
    border-collapse: collapse;
    background: white;
}

th, td {
    padding: 14px;
    text-align: left;
    border-bottom: 1px solid #eee;
}

th {
    background: #263b5e;
    color: white;
}

.btn {
    padding: 9px 14px;
    border: none;
    border-radius: 5px;
    color: white;
    text-decoration: none;
    cursor: pointer;
    display: inline-block;
}

.btn-primary { background: #2563eb; }
.btn-success { background: #16a34a; }
.btn-danger { background: #dc2626; }
.btn-warning { background: #d97706; }

input, select {
    width: 100%;
    padding: 11px;
    margin: 8px 0 18px;
    border: 1px solid #ddd;
    border-radius: 5px;
    box-sizing: border-box;
}

label {
    font-weight: bold;
}

.actions {
    display: flex;
    gap: 8px;
    align-items: center;
}

.actions form {
    margin: 0;
}

.search {
    display: flex;
    gap: 10px;
    margin-bottom: 20px;
}

.search input {
    margin: 0;
}

@media (max-width: 650px) {
    .container {
        margin: 15px;
        overflow-x: auto;
    }

    .header {
        gap: 10px;
        flex-wrap: wrap;
    }

    table {
        min-width: 650px;
    }
}
```

## 9. List and search students

### `src/index.php`

``` php
<?php
require_once __DIR__ . '/../config/database.php';

$search = trim($_GET['search'] ?? '');

if ($search !== '') {
    $sql = "SELECT * FROM students
            WHERE name LIKE ? OR email LIKE ? OR course LIKE ?
            ORDER BY id DESC";

    $stmt = $conn->prepare($sql);
    $term = "%$search%";
    $stmt->bind_param("sss", $term, $term, $term);
} else {
    $stmt = $conn->prepare("SELECT * FROM students ORDER BY id DESC");
}

$stmt->execute();
$result = $stmt->get_result();

$message = $_SESSION['message'] ?? '';
unset($_SESSION['message']);
?>

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>Student Management System</title>
    <link rel="stylesheet"
          href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/css/bootstrap.min.css">
    <link rel="stylesheet" href="style.css">
</head>
<body>
<div class="container">
    <div class="header">
        <h1>Student Management</h1>
        <a href="add.php" class="btn btn-primary">+ Add Student</a>
    </div>

    <?php if ($message): ?>
        <div class="alert alert-success"><?= e($message) ?></div>
    <?php endif; ?>

    <div class="card">
        <form method="GET" class="search">
            <input type="text" name="search"
                   placeholder="Search students..."
                   value="<?= e($search) ?>">
            <button class="btn btn-primary">Search</button>
            <a href="index.php" class="btn btn-warning">Reset</a>
        </form>

        <div class="table-responsive">
            <table>
                <tr>
                    <th>ID</th>
                    <th>Name</th>
                    <th>Email</th>
                    <th>Phone</th>
                    <th>Course</th>
                    <th>Actions</th>
                </tr>

                <?php if ($result->num_rows > 0): ?>
                    <?php while ($row = $result->fetch_assoc()): ?>
                        <tr>
                            <td><?= e($row['id']) ?></td>
                            <td><?= e($row['name']) ?></td>
                            <td><?= e($row['email']) ?></td>
                            <td><?= e($row['phone']) ?></td>
                            <td><?= e($row['course']) ?></td>
                            <td>
                                <div class="actions">
                                    <a href="edit.php?id=<?= e($row['id']) ?>"
                                       class="btn btn-warning">Edit</a>

                                    <form method="POST" action="delete.php"
                                          onsubmit="return confirm('Delete this student?')">
                                        <input type="hidden" name="id"
                                               value="<?= e($row['id']) ?>">
                                        <input type="hidden" name="csrf_token"
                                               value="<?= e(csrf_token()) ?>">
                                        <button class="btn btn-danger">Delete</button>
                                    </form>
                                </div>
                            </td>
                        </tr>
                    <?php endwhile; ?>
                <?php else: ?>
                    <tr>
                        <td colspan="6">No students found.</td>
                    </tr>
                <?php endif; ?>
            </table>
        </div>
    </div>
</div>
</body>
</html>
```

## 10. Add a student

### `src/add.php`

``` php
<?php
require_once __DIR__ . '/../config/database.php';

$name = '';
$email = '';
$phone = '';
$course = '';
$error = '';

$courses = [
    'BCA',
    'BBA',
    'BSc Computer Science',
    'BCom',
    'MCA',
    'MBA'
];

if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    verify_csrf();

    $name = trim($_POST['name'] ?? '');
    $email = trim($_POST['email'] ?? '');
    $phone = trim($_POST['phone'] ?? '');
    $course = trim($_POST['course'] ?? '');

    if ($name === '' || $email === '' || $course === '') {
        $error = "Please fill all required fields.";
    } elseif (!filter_var($email, FILTER_VALIDATE_EMAIL)) {
        $error = "Please enter a valid email address.";
    } elseif (
        strlen($name) > 100 ||
        strlen($email) > 150 ||
        strlen($phone) > 20 ||
        !in_array($course, $courses, true)
    ) {
        $error = "Please check the submitted values.";
    } else {
        $stmt = $conn->prepare(
            "INSERT INTO students (name, email, phone, course)
             VALUES (?, ?, ?, ?)"
        );
        $stmt->bind_param("ssss", $name, $email, $phone, $course);

        if ($stmt->execute()) {
            $_SESSION['message'] = "Student added successfully!";
            header("Location: index.php");
            exit;
        } elseif ($stmt->errno === 1062) {
            $error = "This email is already registered.";
        } else {
            $error = "Unable to save student. Please try again.";
        }
    }
}
?>

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>Add Student</title>
    <link rel="stylesheet"
          href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/css/bootstrap.min.css">
    <link rel="stylesheet" href="style.css">
</head>
<body>
<div class="container">
    <div class="card">
        <h1>Add New Student</h1>

        <?php if ($error): ?>
            <div class="alert alert-danger"><?= e($error) ?></div>
        <?php endif; ?>

        <form method="POST">
            <input type="hidden" name="csrf_token" value="<?= e(csrf_token()) ?>">

            <label>Student Name *</label>
            <input type="text" name="name" maxlength="100"
                   value="<?= e($name) ?>" required>

            <label>Email *</label>
            <input type="email" name="email" maxlength="150"
                   value="<?= e($email) ?>" required>

            <label>Phone Number</label>
            <input type="text" name="phone" maxlength="20"
                   value="<?= e($phone) ?>">

            <label>Course *</label>
            <select name="course" required>
                <option value="">Select Course</option>
                <?php foreach ($courses as $option): ?>
                    <option value="<?= e($option) ?>"
                        <?= $course === $option ? 'selected' : '' ?>>
                        <?= e($option) ?>
                    </option>
                <?php endforeach; ?>
            </select>

            <button type="submit" class="btn btn-success">Save Student</button>
            <a href="index.php" class="btn btn-danger">Cancel</a>
        </form>
    </div>
</div>
</body>
</html>
```

## 11. Edit a student

### `src/edit.php`

``` php
<?php
require_once __DIR__ . '/../config/database.php';

$id = filter_input(INPUT_GET, 'id', FILTER_VALIDATE_INT);

if (!$id || $id < 1) {
    http_response_code(400);
    exit("Invalid student ID.");
}

$stmt = $conn->prepare("SELECT * FROM students WHERE id = ?");
$stmt->bind_param("i", $id);
$stmt->execute();
$student = $stmt->get_result()->fetch_assoc();

if (!$student) {
    http_response_code(404);
    exit("Student not found.");
}

$courses = [
    'BCA',
    'BBA',
    'BSc Computer Science',
    'BCom',
    'MCA',
    'MBA'
];

$error = '';

if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    verify_csrf();

    $name = trim($_POST['name'] ?? '');
    $email = trim($_POST['email'] ?? '');
    $phone = trim($_POST['phone'] ?? '');
    $course = trim($_POST['course'] ?? '');

    if ($name === '' || $email === '' || $course === '') {
        $error = "Please fill all required fields.";
    } elseif (!filter_var($email, FILTER_VALIDATE_EMAIL)) {
        $error = "Invalid email address.";
    } elseif (
        strlen($name) > 100 ||
        strlen($email) > 150 ||
        strlen($phone) > 20 ||
        !in_array($course, $courses, true)
    ) {
        $error = "Please check the submitted values.";
    } else {
        $stmt = $conn->prepare(
            "UPDATE students
             SET name = ?, email = ?, phone = ?, course = ?
             WHERE id = ?"
        );
        $stmt->bind_param("ssssi", $name, $email, $phone, $course, $id);

        if ($stmt->execute()) {
            $_SESSION['message'] = "Student updated successfully!";
            header("Location: index.php");
            exit;
        } elseif ($stmt->errno === 1062) {
            $error = "This email is already used by another student.";
        } else {
            $error = "Unable to update student.";
        }
    }

    $student['name'] = $name;
    $student['email'] = $email;
    $student['phone'] = $phone;
    $student['course'] = $course;
}
?>

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>Edit Student</title>
    <link rel="stylesheet"
          href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/css/bootstrap.min.css">
    <link rel="stylesheet" href="style.css">
</head>
<body>
<div class="container">
    <div class="card">
        <h1>Edit Student</h1>

        <?php if ($error): ?>
            <div class="alert alert-danger"><?= e($error) ?></div>
        <?php endif; ?>

        <form method="POST">
            <input type="hidden" name="csrf_token" value="<?= e(csrf_token()) ?>">

            <label>Name *</label>
            <input type="text" name="name" maxlength="100"
                   value="<?= e($student['name']) ?>" required>

            <label>Email *</label>
            <input type="email" name="email" maxlength="150"
                   value="<?= e($student['email']) ?>" required>

            <label>Phone</label>
            <input type="text" name="phone" maxlength="20"
                   value="<?= e($student['phone']) ?>">

            <label>Course *</label>
            <select name="course" required>
                <?php foreach ($courses as $option): ?>
                    <option value="<?= e($option) ?>"
                        <?= $student['course'] === $option ? 'selected' : '' ?>>
                        <?= e($option) ?>
                    </option>
                <?php endforeach; ?>
            </select>

            <button type="submit" class="btn btn-success">Update Student</button>
            <a href="index.php" class="btn btn-danger">Cancel</a>
        </form>
    </div>
</div>
</body>
</html>
```

## 12. Delete a student

### `src/delete.php`

``` php
<?php
require_once __DIR__ . '/../config/database.php';

if ($_SERVER['REQUEST_METHOD'] !== 'POST') {
    http_response_code(405);
    exit("Method not allowed.");
}

verify_csrf();

$id = filter_input(INPUT_POST, 'id', FILTER_VALIDATE_INT);

if (!$id || $id < 1) {
    http_response_code(400);
    exit("Invalid student ID.");
}

$stmt = $conn->prepare("DELETE FROM students WHERE id = ?");
$stmt->bind_param("i", $id);

if ($stmt->execute() && $stmt->affected_rows > 0) {
    $_SESSION['message'] = "Student deleted successfully!";
} else {
    $_SESSION['message'] = "Student not found or already deleted.";
}

header("Location: index.php");
exit;
```

## 13. Start the application

Open a terminal in the `student-management` project folder.

Build and start the containers:

``` bash
docker compose up -d --build
```

Check the container status:

``` bash
docker compose ps
```

Wait until the MySQL service reports healthy.

### Open the application

-   Student Management System: http://localhost:8080
-   phpMyAdmin: http://localhost:8081

### phpMyAdmin login

  Field      Value
  ---------- ----------------
  Server     `db`
  Username   `student_user`
  Password   `student123`
  Database   `student_db`

Open the `student_db` database and select the `students` table.

## 14. Test the CRUD operations

1.  Open the application.
2.  Add a student named Rahul Sharma.
3.  Enter `rahul@example.com` as the email.
4.  Select BCA as the course.
5.  Save the student and confirm that the record appears.
6.  Search for Rahul Sharma.
7.  Edit the course to MCA.
8.  Delete the student.
9.  Confirm that the record is removed from the database.

## 15. Useful Docker commands

  Command                          Purpose
  -------------------------------- ---------------------------------------
  `docker compose up -d --build`   Build and start services
  `docker compose ps`              View container status
  `docker compose logs web`        View Apache/PHP logs
  `docker compose logs db`         View MySQL logs
  `docker compose restart`         Restart services
  `docker compose down`            Stop and remove containers
  `docker compose up -d`           Start existing services
  `docker compose down -v`         Remove containers and database volume

**Warning:** `docker compose down -v` deletes the persistent database
volume and its data. Do not run it unless you intentionally want to
reset the development database.

## 16. Troubleshooting

  -----------------------------------------------------------------------
  Problem                             What to check
  ----------------------------------- -----------------------------------
  Database connection failed          Run `docker compose logs db` and
                                      check the connection settings

  Website does not open               Run `docker compose ps` and check
                                      port 8080

  Port already in use                 Change the host-side port in
                                      `docker-compose.yml`

  PHP error                           Run `docker compose logs web`

  Table does not exist                Check `database/init.sql` and
                                      whether the database volume was
                                      initialized
  -----------------------------------------------------------------------

If you modify `database/init.sql` after MySQL has initialized its
volume, the script will not automatically run again. Apply the SQL
manually through phpMyAdmin, or reset the development volume if its data
can be discarded.

## 17. What you have learned

You have built a beginner-level full-stack application and practiced:

-   **Linux:** Running Linux-based application containers.
-   **Apache:** Serving the PHP application.
-   **PHP:** Processing forms and implementing CRUD operations.
-   **MySQL:** Creating and managing a relational database.
-   **Docker:** Packaging and running services in containers.
-   **Docker Compose:** Connecting multiple containers.
-   **Security basics:** Prepared statements, output escaping, input
    validation, and CSRF protection.

## 18. Suggested improvements

After the basic application is working, consider adding:

1.  Admin login, logout, and authentication.
2.  Dashboard with student and course statistics.
3.  Pagination for student records.
4.  Student profile photo uploads.
5.  REST API endpoints.
6.  Admin and staff roles.
7.  Production deployment with secure credentials, HTTPS, backups, and
    restricted database access.

## 19. Final project checklist

-   [ ] Docker and Docker Compose installed
-   [ ] Project folders created
-   [ ] Dockerfile configured
-   [ ] Docker Compose services configured
-   [ ] MySQL table created
-   [ ] PHP database connection working
-   [ ] Student listing page working
-   [ ] Add student working
-   [ ] Edit student working
-   [ ] Delete student working
-   [ ] Search working
-   [ ] Data visible in phpMyAdmin

------------------------------------------------------------------------

**Note:** This is a local learning project, not a production-hardened
application. Before public deployment, move credentials out of source
files, configure secure sessions and HTTPS, restrict database exposure,
add authorization, and implement backups.

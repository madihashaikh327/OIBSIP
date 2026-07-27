Task 3 · SQL Injection on DVWA (Low Security)
What is SQL Injection?
SQL Injection is a web security vulnerability that occurs when an application builds a database query by directly inserting user-supplied input into the query string, without properly separating "code" (SQL syntax) from "data" (the actual value being searched for).
Because the database cannot tell whether the text in a query came from the developer or from a user typing into a form, a specially crafted input can change the logic of the query itself — allowing an attacker to bypass authentication, view unauthorized data, or extract entire tables from the database.
Tech Stack / Tools
DVWA (running locally via XAMPP)
Web browser
Setup Summary
Installed XAMPP and started Apache + MySQL services.
Downloaded DVWA from GitHub and placed it inside htdocs.
Configured config.inc.php with valid database credentials for the local MySQL instance.
Used the DVWA setup page to create/reset the database.
Logged in to DVWA using the default credentials (admin / password).
Navigated to DVWA Security and set the security level to Low.
Exploitation Walkthrough
Payload 1 — Authentication/Filter Bypass
Input:
```
' OR '1'='1
```
Why it works: DVWA (Low) builds its query by directly concatenating user input, roughly:
```sql
SELECT first\\\_name, last\\\_name FROM users WHERE user\\\_id = '$id';
```
When the input `' OR '1'='1` is inserted, the query becomes:
```sql
SELECT first\\\_name, last\\\_name FROM users WHERE user\\\_id = '' OR '1'='1';
```
The clause `'1'='1'` is always true. Since the condition is joined with OR, the overall WHERE clause evaluates to true for every row in the table, not just the intended single user.
Data exposed: All user records (first and last names) stored in the users table were returned, instead of the single record that was supposed to be looked up.
Payload 2 — UNION-Based Data Extraction
Input:
```
' UNION SELECT user, password FROM users -- 
```
Why it works: UNION SELECT combines the results of two separate queries into one output, as long as both queries return the same number of columns. The trailing `--` comments out any remaining part of the original query so it doesn't cause a syntax error.
The resulting query becomes:
```sql
SELECT first\\\_name, last\\\_name FROM users WHERE user\\\_id = ''
UNION SELECT user, password FROM users -- ';
```
Data exposed: Usernames and password hashes from the users table were displayed in place of the expected first/last name output — data the application was never designed to reveal through this form.
How a Developer Would Fix This
The root cause is that user input is treated as part of the SQL command itself. The fix is to use parameterized queries / prepared statements, which strictly separate SQL code from user-supplied data. Example using PHP with PDO:
```php
$stmt = $pdo->prepare("SELECT first\\\_name, last\\\_name FROM users WHERE user\\\_id = ?");
$stmt->execute(\\\[$id]);
```
With prepared statements, the database treats $id purely as a literal value to search for — never as executable SQL — regardless of what characters it contains. Even an input like `' OR '1'='1` would simply be searched for as a literal (nonexistent) ID, and no rows would be returned.
DVWA GitHub Repository
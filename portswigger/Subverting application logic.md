[sql injections]

Lab instructions: This lab contains a SQL injection vulnerability in the login function.

To solve the lab, perform a SQL injection attack that logs in to the application as the `administrator` user.

This application utilizes a simple SQL query to authenticate user login:

```SQL
SELECT * FROM users WHERE username = 'administrator' AND password = '???'
```

In this instance, the password is unknown but that doesn't matter because we are going to construct the SQL injection to drop the password query and only look at the username.

I injected `administrator'--` which effectively tells the SQL query, "Hey, kill this password check." -- appends the remainder of the SQL command as a comment, and since "administrator" is a legitimate admin login, we're in.
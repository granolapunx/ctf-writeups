Natas 22 is vulnerable to a simple URL modification. 

The php begins with a gate 

```php
if(array_key_exists("revelio", $_GET)) {
    // only admins can reveal the password
    if(!($_SESSION and array_key_exists("admin", $_SESSION) and $_SESSION["admin"] == 1)) {
    header("Location: /");
    }
}
```

If "revelio" is found in the GET request, then the flag is revealed. If it is not, you are redirected to the homepage via the header command.The vulnerability is that the gate is never closed. It calls 

```php
header("Location: /")
```

but never closes the loop. So PHP keeps firing, which executes the print block:

```php
<?php
    if(array_key_exists("revelio", $_GET)) {
    print "You are an admin. The credentials for the next level are:<br>";
    print "<pre>Username: natas23\n";
    print "Password: <censored></pre>";
    }
?>
```

Which states that if "revelio" is found in the GET request, then the flag is revealed. I used the following one liner to receive the flag:

```bash
curl -u natas22:d8rwGBl0Xslg3b76uh3fEbSlnOUBlozz "http://natas22.natas.labs.overthewire.org/?revelio"
```

This is a classic example of Execution After Redirect. 

The root cause is that the initial redirect is never followed by exit(), so PHP continues executing.

The line header("Location: /") suggests a redirect to the client but does not halt the php function. 

Browsers follow the redirect and discard the body, so if you were attempting to modify the cookie in developer tools and refresh the page, this would not work. However curl does not follow the redirect by default, so the response is visible in full when the exploit is executed through the terminal.

The fix is a single added line — exit() immediately after the header() call — which terminates execution before the privileged output is reached. 

The broader principle: never delegate access control enforcement to client behavior.

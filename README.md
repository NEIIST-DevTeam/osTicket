# osTicket

This is a fork used by NEIIST, with a few patches applied to workaround
limitations/bugs in the original osTicket.

# Patches

These modifications were made for our use cases, so check if they are appropriate for you.

Patches are marked in the source code with `BUGFIX` comments.

### Domain name for HELO message

The SMTP implementation of the PEAR Mail interface can receive the following parameter:

```php
    /**
     * Hostname or domain that will be sent to the remote SMTP server in the
     * HELO / EHLO message.
     *
     * @var string
     */
    var $localhost = 'localhost';
```

However, osTicket's wrappers never pass this parameter to the factory. Thus, spam filters will be triggered (`FSL_HELO_NON_FQDN_1` and `HELO_LOCALHOST`), since the default 'localhost' value is sent in the HELO message.

As long as the fqdn of the server is defined in /etc/hosts, passing the following parameter to the factory seems to work:

```diff
diff --git include/class.email.php include/class.email.php
index 26cd686..cb9cfd2 100644
--- include/class.email.php
+++ include/class.email.php
@@ -356,6 +356,8 @@ class Email {
                            'auth' => (bool) $vars['smtp_auth'],
                            'username' =>$vars['userid'],
                            'password' =>$passwd,
+                           // BUGFIX[nunes] Set localhost to avoid spam flagging
+                           'localhost' => gethostbyaddr(gethostbyname(gethostname())),
                            'timeout'  =>20,
                            'debug' => false,
                            ));
diff --git include/class.mailer.php include/class.mailer.php
index 316d32e..0b7daec 100644
--- include/class.mailer.php
+++ include/class.mailer.php
@@ -417,6 +417,8 @@ class Mailer {
                     'auth' => $smtp['auth'],
                     'username' => $smtp['username'],
                     'password' => $smtp['password'],
+                    // BUGFIX[nunes] Set localhost to avoid spam flagging
+                    'localhost' => gethostbyaddr(gethostbyname(gethostname())),
                     'timeout'  => 20,
                     'debug' => false,
                     'persist' => true,
```

### Collaborator notifications

We noticed that we were receiving only `New Ticket` alerts, missing `New Message` alerts, among others. No configuration in the GUI seemed to solve this.

Apparently there is a bug in auto reply checks, made in `isBounceOrAutoReply()`, which always returned true. We simply skip checks for auto replies.

```diff
diff --git include/class.ticket.php include/class.ticket.php
index e64c38f..35cd1eb 100644
--- include/class.ticket.php
+++ include/class.ticket.php
@@ -1812,7 +1812,9 @@ class Ticket {
             $alerts = isset($vars['flags'])
                 ? !$vars['flags']['bounce'] && !$vars['flags']['auto-reply']
                 : true;
-        if ($alerts && $message->isBounceOrAutoReply())
+        // BUGFIX[nunes] Skip isAutoReply() check, 
+        // it leads to collaborators not being notified
+        if ($alerts && $message->isBounce())
             $alerts = false;

         $this->onMessage($message, $alerts); //must be called b4 sending alerts to staff.
```

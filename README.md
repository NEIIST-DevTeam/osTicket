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
--- a/include/class.email.php	2016-12-31 02:41:37.584410795 +0000
+++ b/include/class.email.php	2016-12-31 02:15:47.335257531 +0000
@@ -361,6 +361,8 @@
                            'auth' => (bool) $vars['smtp_auth'],
                            'username' =>$vars['userid'],
                            'password' =>$passwd,
+                           // BUGFIX[nunes] Set localhost to avoid spam flagging
+                           'localhost' => gethostbyaddr(gethostbyname(gethostname())),
                            'timeout'  =>20,
                            'debug' => false,
                            ));
--- a/include/class.mailer.php	2016-12-31 02:41:37.589410724 +0000
+++ b/include/class.mailer.php	2016-12-31 02:15:47.402256588 +0000
@@ -534,6 +534,8 @@
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
--- a/include/class.ticket.php	2016-12-31 02:41:37.597410612 +0000
+++ b/include/class.ticket.php	2016-12-31 02:31:49.042704762 +0000
@@ -2335,7 +2335,9 @@
                 ? !$vars['mailflags']['bounce'] && !$vars['mailflags']['auto-reply']
                 : true;
         $reopen = $autorespond; // Do not reopen bounces
-        if ($autorespond && $message->isBounceOrAutoReply())
+        // BUGFIX[nunes] Skip isAutoReply() check, 
+        // it leads to collaborators not being notified
+        if ($autorespond && $message->isBounce())
             $autorespond = $reopen= false;
         elseif ($autorespond && isset($vars['autorespond']))
             $autorespond = $vars['autorespond'];
```

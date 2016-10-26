# Let's Encrypt Master Server
## Notes on deploying a master Let's Encrypt server

### Environment / Software
* FreeBSD (but should work on any *nix OS)
* LE client (https://github.com/lukas2511/dehydrated bash script)
* Apache 2.4+ (optional) ...
* ... or PHP with built-in web server enabled (http://php.net/manual/en/features.commandline.webserver.php) ...
* ... or some other script with HTTP listening ability (Python?)

### Goal
Set up Let's Encrypt (LE) management on a single server for creating and renewing certificates that may be deployed on many servers, including non-HTTP servers.

### First Challenge
A single Common Name (CN) domain certificate can have several Subject Alternative Names (SANs) for services that are hosted on multiple servers. Having to run multiple LE clients for the same domain on multiple servers will quickly cause the domain to be rate-limited by LE.  So it's best to put all the SANs in a single certificate. However, this presents a challenge (no pun intended).  The LE client creates challenge files in the .well-known/acme-challenge ($WELLKNOWN) directory on the local host. Because LE certificate validation must connect to the server addresses for the CN and all the SANs, the validation challenge files must be accessible on all the referenced servers.  You could do this by sharing $WELLKNOWN over NFS, or hook into the LE client to copy the challenge files to remote servers. But that's messy and it also requires that those servers run a web server to handle those challenge requests from the LE validator.

### Second Challenge
Some SANs might even be for non-HTTP servers, like SMTP and IMAP. That means the validation connection would fail without a web service to answer for them.  You might not want to have to deploy Apache on some servers just for LE validation, and you shouldn't have to.

### Solution
There's an easier way to do this with a single master LE management server and a little bit of trickery around your network.

First, set up the LE client on a dedicated web server and configure it to manage all the certificates you need on that server and any others in your organization.

Next, configure Apache on your master server (e.g., www.example.com in these examples) *as well as any other web-enabled servers* to respond to validation requests. If using Apache, this requires mod_alias and mod_rewrite. Add this to your server's global configuration:

```
#
# Alias all challenge requests to the challenge token directory
#

Alias "/.well-known/acme-challenge" "/path/to/your/acme-challenges"

#
# Apply properties and rules on challenge token directory
#

<Directory "/path/to/your/acme-challenges">

        # LE mime type for files from this directory
        Header set Content-Type "application/jose+json"

        RewriteEngine on
        RewriteBase /.well-known/acme-challenge

        # If token file does not exist...
        RewriteCond %{REQUEST_FILENAME} !-f

        # ... and this server is not www.example.com
        RewriteCond %{HTTP_HOST}        !^www\.example\.com$ [NC]

        # Redirect to www.example.com passing the request URI
        RewriteCond %{REQUEST_URI}      ^(.*)$
        RewriteRule ^ http://www.example.com%1 [END]

</Directory>
```

**What this does:** You can use this same configuration on the master and any other Apache servers that might be addressed via the SAN references. When the LE client requests certificate validation, the LE validation server will connect to the web server that would handle the CN or SAN. Apache serves up the corresponding file if it exists (that is, on the master).  If the file doesn't exist (that is, the web server is not the master), this will redirect to your master server to handle it.

**Why?** Since the master server will have the local challenge files in $WELLKNOWN, secondary web servers are going to be called for validation but they won't have access to those files. They instead will redirect the validator back to the master. Fortunately, LE supports redirection so this just works.  Once the validator is sent back to the master, it will be able to serve up the challenge response files.  The circuit is complete and validation is successful.

### What about non-HTTP services?

So you want certificates for non-web services, like SMTP, IMAP, MySQL, etc.  You also don't want to have to install Apache on those servers. Here's where some PHP (or equivalent) magic comes to the rescue in a script called *maintenance.php*:

```
<?php
/**
 * Handle Let's Encrypt ACME challenges
 *
 * If the challenge token directory has a matching token, return it.
 * Otherwise, redirect to www.example.com to process the request.
 */

if (preg_match('%^/\.well-known/acme-challenge/(.*)$%', $_SERVER["REQUEST_URI"], $matches)) {
        $redirect_server = 'www.example.com';
        $token = $matches[1];
        $token_file = '/path/to/your/acme-challenges/' . $token;

        if (file_exists($token_file)) {
                header('Content-type: application/jose+json');
                $result = @file_get_contents($token_file) ?: json_encode(array('error' => 'Request resource ' . $token . ' not readable'));
                echo($result);
        } elseif ($_SERVER['HTTP_HOST'] != $redirect_server) {
                header('Location: http://' . $redirect_server . '/.well-known/acme-challenge/' . $token);
        } else {
                header('Content-type: application/jose+json');
                echo(json_encode(array('error' => 'Request resource ' . $token . ' not found')));
        }
    return true;    // served the resource myself
}
?>
<!DOCTYPE html>
<html>
  <head>
    <meta charset="UTF-8">
    <title>Closed for Maintenance</title>
  </head>
  <body>
    <h1>We are temporarily closed<br />for site maintenance.</h1>
    <p>We'll be back in a little while... Thanks for your patience!</p>
  </body>
</html>
```

Start up this script to run in the background from the command line with:

```
nohup /usr/local/bin/php -S 0.0.0.0:80 maintenance.php 2>&1 >/dev/null &
```

You can expand on this to save the process ID, making it a full-fledged service startup script, etc.

**What this does:** This starts up a PHP script in a lightweight web server mode, binding to any interface addresses on port 80 that aren't already in use. This is super handy to have running all the time on a web server in the event you need to take Apache offline, because this script will jump into action and give your web visitors a nice message.  And that's where the magic comes in. If you don't have Apache running, like, say, on an SMTP server, then this script will also answer on port 80 and do something useful, like LE validation!

It will attempt to resolve the challenge using a local version of the challenge response file, if one exists.  That would be the case if you run this on your master server along with Apache, but the address for a CN or SAN coresponds to one that Apache is not listening on.  (Example: your web server also runs MySQL on its own IP address and you want to get a certificate for mysql.example.com).

If the script is running on a server other than the master, it will then issue a redirect to www.example.com so the master Apache server can validate the challenge.  The circuit is complete.

**This combination of solutions satisfies the primary goal and overcomes the challenges.**

### Alternative Solution (no Apache required)

You can use the PHP web server exclusively to respond to LE challenge requests if you prefer.  But if running a PHP script like this  isn't your bag, any other lightweight web server solution that gets the job done is fine. It doesn't have to be robust or full-featured.  It just has to answer to LE requests for challenge files.

I've since adopted the PHP-based server (see above) as the master server solution. It runs on a server that does not have a full-fledged web server on it. I'm using the Apache redirect only on true web servers to send challenge verifications to the PHP master. The neat thing with this combination is that it's easy to move that role around if ever needed.

### Next Steps...

Once you have your master server creating and renewing all the certificates, you simply need to deploy the updated certificates to your various machines via scp or rsync, etc. That exercise is left up to you.

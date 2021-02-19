*********************
Sanitizing user input
*********************

The most obvious, and frequently ignored, danger of web application security is unsanitized user input. Whether URL parameter strings or form submission data, any input coming from outside of your own programming should *never* be trusted.

Manifesto includes a robust sanitization library called **Cleantext** that performs some default sanitization routines (checking for script tags, null characters, etc), and allows for advanced scrubbing of text based on a variety of data types.

Given a variable holding user input, e.g. `$_POST['first_name']` we pass the variable to the Cleantext library for scrubbing::

    Cleantext::clean($_POST['first_name']);
    or the shortcut function
    cleantext($_POST['first_name']);

And this prevents a malicious user from entering something like `<script>foo</script>` as their first name in an attempt to execute unauthorized javascript.

In fact, Manifesto uses the Cleantext library to catch common XSS or SQL injection attacks automatically. For example, since Manifesto allows users to request data with GET queries, like::

    https://example.org/index.php?category=foo&id=123

We automatically run "foo" and "123" through the Cleantext library, and if the original input does not match the output, we can reasonably assume that someone is trying to inject malicious code::

    if ($_POST['first_name'] != cleantext($_POST['first_name'])) {
        // Send a 404, or even a 400 HTTP code to prevent the request from being fulfilled
    }

Manifesto executes this default behavior for the most common request parameters: module, function, id, and category. So if someone requests

   https://example.org/index.php?&category=<script>foo</script>&id=123'
   
their reqeuest will be immediately rejected as a hacking attempt.

Cleantext options
-----------------

As mentioned above, Cleantext can be invoked with a second parameter, indicating a particular format that is expected, e.g.::

    Cleantext::clean($_POST['id'],'integer'); // Allow only integers
    or
    cleantext($_POST['web_address','url'); // All only valid URLs
    
A comprehensive list of available options is below.

+------------------------+-----------------------------------------------------+
| Option string          | Description                                         |
+========================+=====================================================+
| **text/html**          | Allow full HTML*                                    |
+------------------------+-----------------------------------------------------+
| **text/x-html**        | Allow a subset of HTML**                            |
+------------------------+-----------------------------------------------------+
| **passthrough**        | Only essential scrubbing                            |
+------------------------+-----------------------------------------------------+
| **shortname**          | Conforms to Manifesto shortname format              |
+------------------------+-----------------------------------------------------+
| **int**/**integer**    | Integers only                                       |
+------------------------+-----------------------------------------------------+
| **float**              | Floats only                                         |
+------------------------+-----------------------------------------------------+
| **alpha**              | Only a-zA-z characters                              |
+------------------------+-----------------------------------------------------+
| **alphanumeric**       | Only a-zA-z0-9 characters                           |
+------------------------+-----------------------------------------------------+
| **ltr**                | a-zA-z, but only 1 character                        |
+------------------------+-----------------------------------------------------+
| **datetime**           | YYYY-MM-DD HH:MM:SS format                          |
+------------------------+-----------------------------------------------------+
| **year**               | YYYY format                                         |
+------------------------+-----------------------------------------------------+
| **month**              | M, 0M or MM format                                  |
+------------------------+-----------------------------------------------------+
| **datenum**            | D, 0D, or DD format                                 |
+------------------------+-----------------------------------------------------+
| **date**               | YYYY-MM-DD or DD-MM-YYYY format                     |
+------------------------+-----------------------------------------------------+
| **time**               | HH:MM:SS                                            |
+------------------------+-----------------------------------------------------+
| **yearmonth**          | YYYY-MM format                                      |
+------------------------+-----------------------------------------------------+
| **cc**                 | Credit card format (16 numbers + dashes or spaces)  |
+------------------------+-----------------------------------------------------+
| **phone**              | Phone number with optional extension                |
+------------------------+-----------------------------------------------------+
| **email**              | Cleans email address                                |
+------------------------+-----------------------------------------------------+
| **email-check**        | Returns email address if valid, or false            |
+------------------------+-----------------------------------------------------+
| **url**                | Returns scrubbed URL                                |
+------------------------+-----------------------------------------------------+
| **dateobj**            | Returns Manifesto Date object if valid              |
+------------------------+-----------------------------------------------------+
| **array**              | Checks for match against provided 3rd parameter***  |
+------------------------+-----------------------------------------------------+

**\*** "Allow full HTML" *always* removed any `style` or `head` blocks, and removes HTML comments and `script` tags based on the global "Permissive content" settings.

**\*\*** "X-HTML" allows `address, a, b, strong, blockquote, i, em, span, img, u, ol, ul, li, br, p, h1-6`

**\*\*\*** "array" option takes a third parameter, specifying the array whose elements are checked for a match, e.g.::

    Cleantext::clean($_POST['selection'],'array', array('Blue','Red','White'));
    
This will return an empty string if the user 'selection' variable does not match one of 'Blue','Red', or 'White'.

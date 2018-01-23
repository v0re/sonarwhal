# Require resources to be served compressed (`http-compression`)

`http-compression` warns against not serving resources compressed when
requested as such using the most appropriate encoding.

## Why is this important?

One of the fastest and easiest ways one can improve the web site's/app's
performance is to reduce the amount of data that needs to get delivered
to the client by using HTTP compression. This not only [reduces the data
used by the user][wdmsc], but can also significallty cut down on the
server costs.

However, there are a few things that need to be done right in order for
get the most out of compression:

* Only compress resources for which the result of the compression
  will be smaller than original size.

  In general text-based resources (HTML, CSS, JavaScript, SVGs, etc.)
  compresss very well especially if the file is not very small.
  The same goes for some other file formats (e.g.: ICO files, web fonts
  such as EOT, OTF, and TTF, etc.)

  However, compressing resources that are already compressed (e.g.:
  images, audio files, PDFs, etc.) not only waste CPU resources, but
  usually result in little to no reduction, or in some cases even
  a bigger file size.

  The same goes for resources that are very small because of the
  overhead of compression file formats.

* Use most efficien compression method.

  gzip is the most used encoding nowadays as it strikes a good
  balance between compression ratio (as [high as 70%][gzip ratio]
  especially for larger files) and encoding time, and is supported
  pretty much everywhere.

  Better savings can be achieved using [Zopfli][zopfli] which
  can reduce the size on average [3–8% more than gzip][zopfli
  blog post]. Since Zopfli output (for the gzip option) is valid
  gzip content, Zopfli works eveywere gzip works. The only downsize
  is that encoding takes more time than with gzip, thus,
  making Zopfli more suitable for static content (i.e. encoding
  resources as part of build script, not on the fly).

  But, things can be improved even futher using [Brotli][brotli].
  This encoding allows to get [20–26% higher compression ratios][brotli
  blog post] even over Zopfli. However, this encoding is not compatible
  with gzip, limiting the support to modern browsers and its usage to
  [only over HTTPS (as proxies misinterpreting unknown encodings)][brotli
  over https].

  So, in general, for best performance and interoperability resources
  should be served compress with Zopfli, and Brotli over HTTPS with
  a fallback to Zopfli if not supported HTTPS.

* Avoid using deprecated or not widlly supported compression formats,
  and `Content-Type` values.

  Avoid using deprecated `Content-Type` values such as `x-gzip`. Some
  user agents may alias them to the correct, current equivalent value
  (e.g.: alias `x-gzip` to gzip), but that is not always true.

  Also avoid using encoding that are not widely supported (e.g.:
  `compress`, `bzip2`, [`SDCH`][unship sdch], etc.), and/or may not
  be as efficient, or can create problems (e.g.: [`deflate`][deflate
  issues]). In general these should be avoided, and one should just
  stick to the encoding specified in the previous point.

* Avoid potential caching related issues.

  When resources are served compressed, they should be served with
  the `Vary` header containing the `Accept-Encoding` value (or with
  something such as `Cache-Control: private` that prevents caching
  in proxy caches and such altogether).

  This needs to be done in order to avoid problems such as an
  intermediate proxy caching the compress version of the resource and
  then sending it to all user agents regardless if they support that
  particular encoding or not, or if they even want the compressed
  version or not.

* Resources should be served compressed only when requested as such,
  appropriately encoded, and without relying on user agent sniffing.

  The `Accept-Encoding` request header specified should be respected.
  Sending a content encoded with a different encoding than one of the
  ones accepted can lead to problems.

* Dealing with special cases.

  One such special case are `SVGZ` files that are just `SVG` files
  compressed with gzip. Since they are already compressed, they
  shouldn't be compressed again. However sending them without the
  `Content-Encoding: gzip` header will create problems as user agents
  will not know they need to decompress them before displaying them,
  and thus, try to display them directly.

## What does the rule check?

The rule checks for the use cases previously specified, namely, it
checks that:

* Only resources for which the result of the compression is smaller
  than original size are served compressed.

* The most efficient encodigs are used (by default the rule check if
  Zopfli is used over HTTP and Brotli over `HTTPS`, however that can
  be changed, see: [`Can the rule be configured?`
  section](#can-the-rule-be-configured)).

* Deprecated or not widely supported encodigs, and `Content-Type`
  values are not used.

* Potential caching related issues are avoided.

* Resources are served compressed only when requested as such, are
  appropriately encoded, and no user sniffing is done.

* Special cases (such as `SVGZ`) are handled correctly.

### Examples that **trigger** the rule

Resource that should be compressed is not served compressed.

e.g.: When the request for `https://example.com/example.js` contains

```text
...
Accept-Encoding: gzip, deflate, br
```

response is

```text
HTTP/... 200 OK

...
Content-Type: text/javascript

<file content>
```

Resource that should not be compressed is served compressed.

e.g.: When the request for `https://example.com/example.png` contains

```text
...
Accept-Encoding: gzip, deflate, br
```

response is

```text
HTTP/... 200 OK

...
Content-Encoding: br
Content-Type: image/png
Vary: Accept-Encoding

<file content compressed with Brotli>
```

Resource that compressed results in a bigger or equal size to the
uncompressed size is still served compressed.

e.g.: For `http://example.com/example.js` containing only `const x = 5;`,
using the defaults, the sizes may be as follows.

```text
origina size: 13 bytes

gzip size:    38 bytes
zopfli size:  33 bytes
brotli size:  17 bytes
```

When the request for `http://example.com/example.js` contains

```text
...
Accept-Encoding: gzip, deflate
```

response is

```text
HTTP/... 200 OK

...
Content-Encoding: gzip
Content-Type: text/javascript
Vary: Accept-Encoding

<file content compressed with gzip>
```

Resource that should be compressed is served compressed with deprecated
or disallowed compression method or `Content-Encoding` value.

e.g.: When the request for `http://example.com/example.js` contains

```text
...
Accept-Encoding: gzip, deflate
```

response contains deprecated `x-gzip` value for `Content-Encoding`

```text
HTTP/... 200 OK

...
Content-Encoding: x-gzip
Content-Type: text/javascript
Vary: Accept-Encoding

<file content compressed with gzip>
```

response is compressed with disallowed `compress` compression method

```text
HTTP/... 200 OK

...
Content-Encoding: compress
Content-Type: text/javascript
Vary: Accept-Encoding

<file content compressed with compress>
```

or response tries to use deprecated SDCH

```text
HTTP/... 200 OK

...
Content-Encoding: gzip,
Content-Type: text/javascript
Get-Dictionary: /dictionaries/search_dict, /dictionaries/help_dict
Vary: Accept-Encoding

<file content compressed with gzip>
```

Resource that should be compressed is not served compressed using
Zopfli over HTTP.

e.g.: When the request for `http://example.com/example.js` contains

```text
...
Accept-Encoding: gzip, deflate
```

response is

```text
HTTP/... 200 OK

...
Content-Encoding: gzip
Content-Type: text/javascript
Vary: Accept-Encoding

<content compressed with gzip>
```

Resource that should be compressed is served compressed using Brotli
over HTTP.

e.g.: When the request for `http://example.com/example.js` contains

```text
...
Accept-Encoding: gzip, deflate, br
```

response is

```text
HTTP/... 200 OK

...
Content-Encoding: br
Content-Type: text/javascript
Vary: Accept-Encoding

<content compressed with Brotli>
```

Resource that should be compressed is not served compressed using
Brotli over HTTPS.

e.g.: When the request for `https://example.com/example.js` contains

```text
...
Accept-Encoding: gzip, deflate, br
```

response is

```text
HTTP/... 200 OK

...
Content-Encoding: gzip
Content-Type: text/javascript
Vary: Accept-Encoding

<content compressed with Zopfli>
```

Resource that is served compressed doesn't account for caching
(e.g: is not served with the `Vary` header with the `Accept-Encoding`
value included, or something such as `Cache-Control: private`).

e.g.: When the request for `https://example.com/example.js` contains

```text
...
Accept-Encoding: gzip, deflate, br
```

response is

```text
HTTP/... 200 OK

...
Content-Encoding: br
Content-Type: text/javascript

<content compressed with Brotli>
```

Resource is blindly served compressed using gzip no matter what the
user agent advertises as supporting.

E.g.: When the request for `https://example.com/example.js` contains

```text
...
Accept-Encoding: br
```

response is

```text
HTTP/... 200 OK

...
Content-Encoding: gzip
Content-Type: text/javascript
Vary: Accept-Encoding

<content compressed with gzip>
```

Resource is served compressed only for certain user agents.

E.g.: When the request for `https://example.com/example.js` contains

```text
...
Accept-Encoding: gzip, deflate, br
User-Agent: Mozilla/5.0 Gecko
```

response is

```text
HTTP/... 200 OK

...
Content-Type: text/javascript

<file content>
```

however when requested with

```text
...
Accept-Encoding: gzip, deflate, br
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.13; rv:57.0) Gecko/20100101 Firefox/57.0
```

response is

```text
HTTP/... 200 OK

...
Content-Encoding: br
Content-Type: text/javascript
Vary: Accept-Encoding

<content compressed with Brotli>
```

`SVGZ` resource is not served with `Content-Encoding: gzip` header:

E.g.: When the request for `https://example.com/example.svgz` contains

```text
...
Accept-Encoding: gzip, deflate, br
```

response is

```text
HTTP/... 200 OK

...
Content-Type: image/svg+xml

<file content>
```

### Examples that **pass** the rule

Resource that should be compressed is served compressed using Zopfli
over HTTP and with the `Vary: Accept-Encoding` header.

e.g.: When the request for `http://example.com/example.js` contains

```text
...
Accept-Encoding: gzip, deflate
```

response is

```text
HTTP/... 200 OK

...
Content-Encoding: gzip
Content-Type: text/javascript
Vary: Accept-Encoding

<content compressed with Zopfli>
```

Resource that should be compressed is served compressed using Brotli
over HTTPS and with the `Vary: Accept-Encoding` header.

e.g.: When the request for `https://example.com/example.js` contains

```text
...
Accept-Encoding: gzip, deflate, br
```

response is

```text
HTTP/... 200 OK

...
Content-Encoding: br
Content-Type: text/javascript
Vary: Accept-Encoding

<content compressed with Brotli>
```

Resource that should not be compressed is not served compressed.

e.g.: When the request for `https://example.com/example.png` contains

```text
...
Accept-Encoding: gzip, deflate, br
```

response is

```text
HTTP/... 200 OK

...
Content-Type: image/png

<image content>
```

`SVGZ` resource is served with `Content-Encoding: gzip` header:

e.g.: When the request for `https://example.com/example.svgz` contains

```text
...
Accept-Encoding: gzip, deflate, br
```

response is

```text
HTTP/... 200 OK

...
Content-Encoding: gzip
Content-Type: image/svg+xml

<SVGZ content>
```

## How to configure the server to pass this rule

<!-- markdownlint-disable MD033 -->

<details>
<summary>How to configure Apache</summary>

Apache can be configured to conditionally (based on media type)
compress resources using gzip as well as send the appropriate
`Content-Encoding` and `Vary` headers using [`mod_deflate`][mod_deflate]
and the [`AddOutputFilterByType` directive][addoutputfilterbytype].

For Zopfli, there isn't a core Apache module or directive to do it,
However, since compressing things using Zopfli takes more time, it's
usually indicated to do it as part of your build step. Once that is
done, Apache needs to be configure to server those pre-compressed
files when gzip compression is requested by the user agent.

Starting with Apache `v2.4.26`, [`mod_brotli`][mod_brotli] and the
[`AddOutputFilterByType` directive][addoutputfilterbytype] can be used
to condtitionally compress with Brotli as well as add the
`Content-Encoding` and `Vary` headers. However, since, like Zopfli,
Brotli takes more time, it's indicated to compress resources at build
time, and configure Apache to just serve those pre-compressed resources
whenever Brotli compression is requested over HTTPS by the user agent.

If you don't want to start from scratch, below is a generic starter
snippet that contains the necessary configurations to ensure that
commonly used file types are served compressed and with the appropriate
headers, and thus, make your web site/app pass this rule.

Important notes:

* The following relies on Apache being configure to have the correct
  filename extensions to media types mappings (see Apache section from
  [`content-type` rule](content-type.md#how-to-configure-the-server-to-pass-this-rule)).

* For Zopfli and Brotli this snippet assumes that running the build
  step will result in 3 version for every resource:

  * the original (e.g.: script.js) - you should also have this file
    in case the user agent doesn't requests things compressed
  * the file compressed with Zopfli (e.g.: script.js.gz)
  * the file compressed with Brotli (e.g.: script.js.br)

```apache
<IfModule mod_headers.c>
    <IfModule mod_rewrite.c>

        # Turn on the rewrite engine (this is necessary in order for
        # the `RewriteRule` directives to work).
        #
        # https://httpd.apache.org/docs/current/mod/core.html#options

        RewriteEngine On

        # Enable the `FollowSymLinks` option if it isn't already.
        #
        # https://httpd.apache.org/docs/current/mod/core.html#options

        Options +FollowSymlinks

        # If the web host doesn't allow the `FollowSymlinks` option,
        # it needs to be comment out or removed, and then the following
        # uncomment, but be aware of the performance impact.
        #
        # https://httpd.apache.org/docs/current/misc/perf-tuning.html#symlinks

        # Options +SymLinksIfOwnerMatch

        # Depending on how the server is set up, you may also need to
        # use the `RewriteOptions` directive to enable some options for
        # the rewrite engine.
        #
        # https://httpd.apache.org/docs/current/mod/mod_rewrite.html#rewriteoptions

        # RewriteBase /

        # - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -

        # 1) Brotli

            # If `Accept-Encoding` header contains `gzip` and the
            # request is made over HTTP.

            RewriteCond "%{HTTPS:Accept-encoding}" "br"

            # The Brotli pre-compressed version of the file exists
            # (e.g.: `script.js` is requested and `script.js.gz` exists).

            RewriteCond "%{REQUEST_FILENAME}\.br" "-s"

            # Then serve the Brotli pre-compressed version of the file.

            RewriteRule "^(.*)" "$1\.br" [QSA]

            # Set the media types of the file, as otherwise, because
            # the file has the `.gz` extension, it wil be served with
            # the gzip media type.
            #
            # Also, set the special purpose environment variables so
            # that Apache doesn't recompress these files.

            RewriteRule "\.(ico|cur)\.br$"      "-" [T=image/x-icon,E=no-brotli:1,E=no-gzip:1]
            RewriteRule "\.(md|markdown)\.br$"  "-" [T=text/markdown,E=no-brotli:1,E=no-gzip:1]
            RewriteRule "\.appcache\.br$"       "-" [T=text/cache-manifest,E=no-brotli:1,E=no-gzip:1]
            RewriteRule "\.atom\.br$"           "-" [T=application/atom+xml,E=no-brotli:1,E=no-gzip:1]
            RewriteRule "\.bmp\.br$"            "-" [T=image/bmp,E=no-brotli:1,E=no-gzip:1]
            RewriteRule "\.css\.br$"            "-" [T=text/css,E=no-brotli:1,E=no-gzip:1]
            RewriteRule "\.eot.\.br$"           "-" [T=application/vnd.ms-fontobject,E=no-brotli:1,E=no-gzip:1]
            RewriteRule "\.geojson\.br$"        "-" [T=application/vnd.geo+json,E=no-brotli:1,E=no-gzip:1]
            RewriteRule "\.html?\.br$"          "-" [T=text/html,E=no-brotli:1,E=no-gzip:1]
            RewriteRule "\.ics\.br$"            "-" [T=text/calendar,E=no-brotli:1,E=no-gzip:1]
            RewriteRule "\.json\.br$"           "-" [T=application/json,E=no-brotli:1,E=no-gzip:1]
            RewriteRule "\.jsonld\.br$"         "-" [T=application/ld+json,E=no-brotli:1,E=no-gzip:1]
            RewriteRule "\.m?js\.br$"           "-" [T=text/javascript,E=no-brotli:1,E=no-gzip:1]
            RewriteRule "\.otf\.br$"            "-" [T=font/otf,E=no-brotli:1,E=no-gzip:1]
            RewriteRule "\.rdf\.br$"            "-" [T=application/rdf+xml,E=no-brotli:1,E=no-gzip:1]
            RewriteRule "\.rss\.br$"            "-" [T=application/rss+xml,E=no-brotli:1,E=no-gzip:1]
            RewriteRule "\.svg\.br$"            "-" [T=image/svg+xml,E=no-brotli:1,E=no-gzip:1]
            RewriteRule "\.ttc\.br$"            "-" [T=font/collection,E=no-brotli:1,E=no-gzip:1]
            RewriteRule "\.ttf\.br$"            "-" [T=font/ttf,E=no-brotli:1,E=no-gzip:1]
            RewriteRule "\.txt\.br$"            "-" [T=text/plain,E=no-brotli:1,E=no-gzip:1]
            RewriteRule "\.vc(f|ard)\.br$"      "-" [T=text/vcard,E=no-brotli:1,E=no-gzip:1]
            RewriteRule "\.vtt\.br$"            "-" [T=text/vtt,E=no-brotli:1,E=no-gzip:1]
            RewriteRule "\.webmanifest\.br$"    "-" [T=application/manifest+json,E=no-brotli:1,E=no-gzip:1]
            RewriteRule "\.xhtml\.br$"          "-" [T=application/xhtml+xml,E=no-brotli:1,E=no-gzip:1]
            RewriteRule "\.xml\.br$"            "-" [T=text/xml,E=no-brotli:1,E=no-gzip:1]

            # Set the `Content-Encoding` header.

            <FilesMatch "\.br$">
                Header append Content-Encoding br
            </FilesMatch>

        # - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -

        # 2) Zopfli

            # If `Accept-Encoding` header contains `gzip` and the
            # request is made over HTTP.

            RewriteCond "%{HTTP:Accept-encoding}" "gzip"

            # The Zopfli pre-compressed version of the file exists
            # (e.g.: `script.js` is requested and `script.js.gz` exists).

            RewriteCond "%{REQUEST_FILENAME}\.gz" "-s"

            # Then serve the Zopfli pre-compressed version of the file.

            RewriteRule "^(.*)" "$1\.gz" [QSA]

            # Set the media types of the file, as otherwise, because
            # the file has the `.gz` extension, it wil be served with
            # the gzip media type.
            #
            # Also, set the special purpose environment variables so
            # that Apache doesn't recompress these files.

            RewriteRule "\.(ico|cur)\.gz$"      "-" [T=image/x-icon,E=no-brotli:1,E=no-gzip:1]
            RewriteRule "\.(md|markdown)\.gz$"  "-" [T=text/markdown,E=no-brotli:1,E=no-gzip:1]
            RewriteRule "\.appcache\.gz$"       "-" [T=text/cache-manifest,E=no-brotli:1,E=no-gzip:1]
            RewriteRule "\.atom\.gz$"           "-" [T=application/atom+xml,E=no-brotli:1,E=no-gzip:1]
            RewriteRule "\.bmp\.gz$"            "-" [T=image/bmp,E=no-brotli:1,E=no-gzip:1]
            RewriteRule "\.css\.gz$"            "-" [T=text/css,E=no-brotli:1,E=no-gzip:1]
            RewriteRule "\.eot.\.gz$"           "-" [T=application/vnd.ms-fontobject,E=no-brotli:1,E=no-gzip:1]
            RewriteRule "\.geojson\.gz$"        "-" [T=application/vnd.geo+json,E=no-brotli:1,E=no-gzip:1]
            RewriteRule "\.html?\.gz$"          "-" [T=text/html,E=no-brotli:1,E=no-gzip:1]
            RewriteRule "\.ics\.gz$"            "-" [T=text/calendar,E=no-brotli:1,E=no-gzip:1]
            RewriteRule "\.json\.gz$"           "-" [T=application/json,E=no-brotli:1,E=no-gzip:1]
            RewriteRule "\.jsonld\.gz$"         "-" [T=application/ld+json,E=no-brotli:1,E=no-gzip:1]
            RewriteRule "\.m?js\.gz$"           "-" [T=text/javascript,E=no-brotli:1,E=no-gzip:1]
            RewriteRule "\.otf\.gz$"            "-" [T=font/otf,E=no-brotli:1,E=no-gzip:1]
            RewriteRule "\.rdf\.gz$"            "-" [T=application/rdf+xml,E=no-brotli:1,E=no-gzip:1]
            RewriteRule "\.rss\.gz$"            "-" [T=application/rss+xml,E=no-brotli:1,E=no-gzip:1]
            RewriteRule "\.svg\.gz$"            "-" [T=image/svg+xml,E=no-brotli:1,E=no-gzip:1]
            RewriteRule "\.ttc\.gz$"            "-" [T=font/collection,E=no-brotli:1,E=no-gzip:1]
            RewriteRule "\.ttf\.gz$"            "-" [T=font/ttf,E=no-brotli:1,E=no-gzip:1]
            RewriteRule "\.txt\.gz$"            "-" [T=text/plain,E=no-brotli:1,E=no-gzip:1]
            RewriteRule "\.vc(f|ard)\.gz$"      "-" [T=text/vcard,E=no-brotli:1,E=no-gzip:1]
            RewriteRule "\.vtt\.gz$"            "-" [T=text/vtt,E=no-brotli:1,E=no-gzip:1]
            RewriteRule "\.webmanifest\.gz$"    "-" [T=application/manifest+json,E=no-brotli:1,E=no-gzip:1]
            RewriteRule "\.xhtml\.gz$"          "-" [T=application/xhtml+xml,E=no-brotli:1,E=no-gzip:1]
            RewriteRule "\.xml\.gz$"            "-" [T=text/xml,E=no-brotli:1,E=no-gzip:1]

            # Set the `Content-Encoding` header.

            <FilesMatch "\.gz$">
                Header append Content-Encoding gzip
            </FilesMatch>

        # - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -

        # Set the `Vary` header.

        <FilesMatch "\.(br|gz)$">
            Header append Vary Accept-Encoding
        </FilesMatch>

    </IfModule>
</IfModule>

<IfModule mod_deflate.c>

    # 3) gzip
    #
    # [!] For Apache versions below version 2.3.7 you don't need to
    # enable `mod_filter` and can remove the `<IfModule mod_filter.c>`
    # and `</IfModule>` lines as `AddOutputFilterByType` is still in
    # the core directives.
    #
    # https://httpd.apache.org/docs/current/mod/mod_filter.html#addoutputfilterbytype

    <IfModule mod_filter.c>
        AddOutputFilterByType DEFLATE "application/atom+xml" \
                                      "application/json" \
                                      "application/ld+json" \
                                      "application/manifest+json" \
                                      "application/rdf+xml" \
                                      "application/rss+xml" \
                                      "application/schema+json" \
                                      "application/vnd.geo+json" \
                                      "application/vnd.ms-fontobject" \
                                      "application/xhtml+xml" \
                                      "font/collection" \
                                      "font/opentype" \
                                      "font/otf" \
                                      "font/ttf" \
                                      "image/bmp" \
                                      "image/svg+xml" \
                                      "image/x-icon" \
                                      "text/cache-manifest" \
                                      "text/calendar" \
                                      "text/css" \
                                      "text/html" \
                                      "text/javascript" \
                                      "text/markdown" \
                                      "text/plain" \
                                      "text/vcard" \
                                      "text/vtt" \
                                      "text/xml"
    </IfModule>

    # - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -

    # Special case: SVGZ
    #
    # If these files type would be served without the
    # `Content-Enable: gzip` response header, user agents would
    # not know that they first need to uncompress the response,
    # and thus, wouldn't be able to understand the content.

    <IfModule mod_mime.c>
        AddEncoding gzip              svgz
    </IfModule>

</IfModule>
```

Also note that:

* The above snippet works with Apache `v2.2.0+`, but you need to
  have [`mod_deflate`][mod_deflate], [`mod_mime`][mod_mime],
  [`mod_rewrite`][mod_rewrite], and for Apache versions below
  `v2.3.7` [`mod_filter`][mod_filter] [enabled][how to enable apache
  modules] in order for it to take effect.

* If you have access to the [main Apache configuration file][main
  apache conf file] (usually called `httpd.conf`), you should add
  the logic in, for example, a [`<Directory>`][apache directory]
  section in that file. This is usually the recommended way as
  [using `.htaccess` files slows down][htaccess is slow] Apache!

  If you don't have access to the main configuration file (quite
  common with hosting services), just add the snippets in a `.htaccess`
  file in the root of the web site/app.

</details>

<!-- markdownlint-enable MD033 -->

## Can the rule be configured?

You can override the defaults by specifying what type of compression
you don't want the rule to check for. This can be done for the `target`
(main page) and/or the `resources` the rule determines should be served
compressed, using the following format:

```json
"http-compression": [ "warning", {
    "resource": {
        "<compression_type>": <true|false>,
        ...
    },
    "target": {
        "<compression_type>": <true|false>,
        ...
    }
}
```

Where `<compression_method>` can be one of: `brotli`, `gzip`, or
`zopfli`.

E.g. If you want the rule to check if only the page resources are
served compressed using Brotli, and not the page itself, you can
use the following configuration:

```json
"http-compression": [ "warning", {
    "target": {
        "brotli": false
    }
}]
```

Note: You can also use the [`ignoredUrls`](../index.md#rule-configuration)
property from the `.sonarwhalrc` file to exclude domains you don’t control
(e.g.: CDNs) from these checks.

<!-- Link labels: -->

[brotli blog post]: https://opensource.googleblog.com/2015/09/introducing-brotli-new-compression.html
[brotli over https]: https://medium.com/@yoavweiss/well-the-technical-reason-for-brotli-being-https-only-is-that-otherwise-there-s-a-very-high-508f15f0ad95
[brotli]: https://github.com/google/brotli
[deflate issues]: https://stackoverflow.com/questions/9170338/why-are-major-web-sites-using-GZIP/9186091#9186091
[gzip is not enough]: https://www.youtube.com/watch?v=whGwm0Lky2s
[gzip ratio]: https://www.youtube.com/watch?v=Mjab_aZsdxw&t=24s
[unship sdch]: https://groups.google.com/a/chromium.org/forum/#!topic/blink-dev/nQl0ORHy7sw
[wdmsc]: https://whatdoesmysitecost.com/
[zopfli blog post]: https://developers.googleblog.com/2013/02/compress-data-more-densely-with-zopfli.html
[zopfli]: https://github.com/google/zopfli

<!-- Apache links -->

[addoutputfilterbytype]: https://httpd.apache.org/docs/2.4/mod/mod_filter.html#addoutputfilterbytype
[apache directory]: https://httpd.apache.org/docs/current/mod/core.html#directory
[how to enable apache modules]: https://github.com/h5bp/server-configs-apache/wiki/How-to-enable-Apache-modules
[htaccess is slow]: https://httpd.apache.org/docs/current/howto/htaccess.html#when
[main apache conf file]: https://httpd.apache.org/docs/current/configuring.html#main
[mod_brotli]:  https://httpd.apache.org/docs/trunk/mod/mod_brotli.html
[mod_deflate]: https://httpd.apache.org/docs/current/mod/mod_deflate.html
[mod_filter]: https://httpd.apache.org/docs/current/mod/mod_filter.html
[mod_mime]: https://httpd.apache.org/docs/current/mod/mod_mime.html
[mod_rewrite]: https://httpd.apache.org/docs/current/mod/mod_rewrite.html
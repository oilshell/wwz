wwz
===

A WSGI program that serves files from a zip file.  Deploy as CGI or FastCGI.
See this comment for context:

- <https://lobste.rs/s/xl63ah/fastcgi_forgotten_treasure#c_4jggfz>
- Related answer: Why FastCGI and not HTTP?  Because FastCGI processes are
  stateless/ephemeral.
  - <https://lobste.rs/s/xl63ah/fastcgi_forgotten_treasure#c_kaajpp>
- Why FastCGI and not CGI?  For lower latency.  We "cache" the opening of the
  zip file, and the overhead of starting CPython.

## Live Demos

- <https://www.oilshell.org/release/0.8.3/test/wild.wwz/> (HTML files thousands
  of shell scripts)
- <http://travis-ci.oilshell.org/travis-jobs/> (for raw build logs)

## June 2024 Update

FastCGI no longer works reliably on DreamHost: [Comments on Scripting, CGI, and
FastCGI](https://www.oilshell.org/blog/2024/06/cgi.html).

So for now I've changed the default WSGI to CGI, rather than FastCGI.  This
means that some pages on `oilshell.org` took a 50 millisecond latency hit :-(

I hope to deploy it as a persistent process on other servers.

## Older Notes

The notes below aren't up to date, but they may give you a feeling for how it
works.

### The General Idea

`wwz.py` is a very small WSGI app.  You download the `flup` "middleware" which
turns the WSGI app into a FastCGI server.  (Analogously, you can turn a WSGI
app into a CGI program.)

### Files

    wwz.py         # The WSGI program
    wwz-test.sh    # Shell tests for the FastCGI program
    wwz.htaccess   # A snippet to configure Apache on Dreamhost
    admin.sh       # Some shell functions that may be useful

Not included:

    dispatch.fcgi  # A shell wrapper specific to the hosting environment.

### Local Development

I do this both locally **and** on the server:

    ./admin.sh download-flup  # Yes this is old but it works!
    ./admin.sh build-flup
    ./admin.sh smoke-test

Then test out the app locally:

    ./wwz-test.sh make-testdata  # makes a zip file
    ./wwz-test.sh all

### Deploying

I deploy it on Dreamhost.  Instructions vary depending on the web host.  (TODO:
I want to stand it up on NearlyFreeSpeech too.)

Requirements:

1. A directory for the binary
2. A directory for logs and unhandled exceptions.
3. A `dispatch.fcgi` script that sets PYTHONPATH and execs `wwz.py`.
4. The `.htaccess` file for Apache to read.

If all those elements are in place, Dreamhost's Apache server will send
requests like 

    https://www.oilshell.org/release/0.8.1/test/wild.wwz/

to a `wwz.py` process that the `dispatch.fcgi` shell wrapper started.  (The web
host maintains a FastCGI process manager, so a process isn't started on every
request.)

Example deploy function:

    for-travis() {
      local dest=~/travis-ci.oilshell.org

      mkdir -v -p $dest/wwz-bin ~/wwz-logs

      cp -v wwz.htaccess $dest/.htaccess

      cp -v wwz.py $dest/wwz-bin/
      cp -v travis_dispatch.fcgi $dest/wwz-bin/dispatch.fcgi

      make-testdata
      copy-testdata
    }

Example `dispatch.fcgi`:

    #!/bin/sh
    root=/home/travis_admin/git/dreamhost/wwz-fastcgi

    PYTHONPATH=$root/_tmp/flup-1.0.3.dev-20110405 \
      exec ~/travis-ci.oilshell.org/wwz-bin/wwz.py ~/wwz-logs

You can also set `WWZ_REQUEST_LOG=1` and/or `WWZ_TRACE_LOG=1` to get more
detailed logs.

### Administering

Sometimes I do this on the server:

    ./admin.sh kill-wwz

## Feedback

I'm interested if you deploy this script on shared hosting and use it to serve
content.

Feel free to fork and modify!  It's been working for ~3 years for me without
many changes.  I may or may not accept PRs, but let me know what you're doing
with it!



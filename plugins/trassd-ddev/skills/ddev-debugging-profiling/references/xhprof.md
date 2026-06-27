# XHProf profiling in DDEV

DDEV has built-in XHProf support (the PECL extension requires PHP >= 7.x). Pick
one of two modes globally with `ddev config global --xhprof-mode=...`, then
restart.

## XHGui mode (recommended, DDEV v1.24.4+)

```bash
ddev config global --xhprof-mode=xhgui && ddev restart
ddev xhgui on        # start collecting profiling data
# browse a few pages in the app, then:
ddev xhgui launch    # open the XHGui web interface
ddev xhgui           # also launches the web interface
```

## Traditional `prepend` mode

Fall back to the classic XHProf web UI if XHGui gives you trouble:

```bash
ddev config global --xhprof-mode=prepend
ddev xhprof on       # aliases: `ddev xhprof`, `ddev xhprof enable`
ddev xhprof status   # show status
```

- `ddev xhprof on` prints the URL to view analysis; recent runs are at
  `https://<projectname>.ddev.site/xhprof`. Keep a tab open and refresh.
- To avoid first-time cache-building noise, hit the page under study twice.
- On the output page, drill into the function you care about or open
  "View Full Callgraph". Click column headers to sort by run count and by
  inclusive / exclusive wall time.
- Runs are erased on `ddev restart`.
- **Apache:** if you use a custom `.ddev/apache/apache-site.conf`, it must
  include `Alias "/xhprof" "/var/xhprof/xhprof_html"` (present in DDEV's default
  apache-site.conf).

## Advanced `prepend` customization

Edit the `xhprof_prepend` function in `.ddev/xhprof/xhprof_prepend.php` to change
behavior. When you customize it, remove the `#ddev-generated` line at the top and
force-add it: `git add -f .ddev/xhprof/xhprof_prepend.php`.

Examples:

- Add a link to the profile run at the bottom of the profiled page (the shipped
  file includes a sample function that works with Drupal 7).
- Drop memory profiling for fewer columns: change
  `xhprof_enable(XHPROF_FLAGS_MEMORY);` to `xhprof_enable();`.

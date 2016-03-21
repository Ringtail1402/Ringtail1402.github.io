ringtail1402.github.io
======================

GitHub pages repo.

Note to self: use http://www.photo2blog.ru/ for quick album page generation
(hopefully this service will not go down in foreseeable future).

Preset to be used (800x537 images, Retina-enabled with picturefill.js):

```
<p><a href="%(source_o)s" target="_blank"><img srcset="%(source_ml)s 1x, %(source_xl)s 2x" width="%(width_ml)s" height="%(height_ml)s" alt="%(num)s" title="%(title)s" class="img-thumbnail"></a></p>

<p>%(num)s. %(description)s</p>
```
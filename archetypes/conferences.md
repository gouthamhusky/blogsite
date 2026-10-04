---
title: "{{ replace .Name "-" " " }}"
# The first day of the conference. Future dates show as "upcoming".
date: {{ .Date }}
# Lets the page build while `date` is still in the future. Leave it alone.
publishDate: {{ .Date }}
event_url: ""   # the conference's own page
location: ""
role: "attended"   # attended | spoke | organized | booth staff for …
linkedin: ""    # your post about it
draft: false
---
<!-- Optional: notes and takeaways. For photos, create this as a bundle
     (hugo new conferences/<name>/index.md) and drop .jpg files next to it. -->

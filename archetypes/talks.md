---
title: "{{ replace .Name "-" " " }}"
# The day of the talk. Future dates are fine — they show as "upcoming".
date: {{ .Date }}
# Lets the page build while `date` is still in the future. Leave it alone.
publishDate: {{ .Date }}
event: ""
event_url: ""   # meetup / conference page
location: ""
slides: ""
video: ""
linkedin: ""    # your post about the talk
draft: false
---
<!-- Optional: the abstract. Leave empty and the page is just the details line. -->

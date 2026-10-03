---
title: "{{ replace .Name "-" " " }}"
# The day of the talk. Future dates are fine — they show as "upcoming".
date: {{ .Date }}
# Lets the page build while `date` is still in the future. Leave it alone.
publishDate: {{ .Date }}
event: ""
location: ""
slides: ""
video: ""
draft: false
---
<!-- Optional: the abstract. Leave empty and the page is just the details line. -->

---
title: "{{ replace .Name "-" " " }}"
# The first day of the conference. Future dates show as "upcoming".
date: {{ .Date }}
# Lets the page build while `date` is still in the future. Leave it alone.
publishDate: {{ .Date }}
location: ""
role: "attended"   # attended | spoke | organized
draft: false
---
<!-- Optional: notes and takeaways. -->

---
title: "Grafana for Kubernetes-based platforms: every cluster, every region, one dashboard"
date: 2026-09-22
publishDate: 2026-09-22
event: "Grafana & Friends Austin — Autumn of Observability"
event_url: "https://www.meetup.com/grafana-friends-austin-meetup-group/events/316372961/"
location: "Austin, TX"
slides: "https://github.com/gouthamhusky/talks/blob/main/2026-09-22-grafana-for-kubernetes-platforms/grafana-for-kubernetes-platforms.pdf"
video: ""
linkedin: "https://www.linkedin.com/posts/goutham-kanags_i-just-finished-presenting-my-talk-on-grafana-ugcPost-7508382117324623873-Vtz4/"
---
A tenant cluster is stuck: which one, in which region, and why? On a
platform built from operators, the answer is already written down. Every
reconcile loop writes status conditions back onto the resource. This talk
exports those conditions as Prometheus time series (one gauge, the condition
as a label, `min` to aggregate) and builds one Grafana dashboard for the
whole fleet, across every cluster and every region.

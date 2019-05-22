
## Notes
- Loop's custom envoy seems to be interferring with supergloo's ability to translate faults into istio resources. The effect of the vulnerability experiment was not shown when I had loop set up and running in the same cluster.
  - this may be resolved when the tap feature is fully upstreamed
  - for now, use separate clusters for the two demos

# Konflux Build Identification (step 6b reference)

Identify the FBC builds and the eventrouter image for the release. Component and FBC builds live
in Konflux; dots in the stream become dashes in the application name. URL patterns (replace
`X-Y`, e.g. `6-5`):

- Components: `https://konflux-ui.apps.kflux-ocp-p01.7ayg.p1.openshiftapps.com/ns/art-logging-tenant/applications/logging-<X-Y>/components`
- FBC: `https://konflux-ui.apps.kflux-ocp-p01.7ayg.p1.openshiftapps.com/ns/art-logging-tenant/applications/fbc-logging-<X-Y>/components`
- Releases: `https://konflux-ui.apps.kflux-ocp-p01.7ayg.p1.openshiftapps.com/ns/art-logging-tenant/applications/logging-<X-Y>/releases`

## Steps

1. **Check recent component builds** happened and were successful (PipelineRuns for the release,
   e.g. `logging-6.6`).
2. **Check recent staging releases** were successful and are **newer than the component builds**,
   comparing each component's builds against the staging releases on the application's releases
   page (e.g. `logging-6-5`). This matters mostly for the **eventrouter** image — it is not part
   of the operators and thus the FBCs, so it must be mentioned separately in the release. Filter
   releases by name to find the relevant ones, e.g.
   `.../applications/logging-6-5/releases?name=event` for eventrouter releases.
   The last staging release was probably triggered by a bundle build (CLO or LO), unless the last
   change was to eventrouter.
3. **Pick the eventrouter candidate:** the most recent eventrouter release (found via the
   `?name=event` filter above). Open it and note the NVR for the eventrouter image from its
   **Release Artifacts** page, e.g.
   `ose-logging-eventrouter-container-6.5.2-202608131800.p2.ge2c2ae9.assembly.stream.el9`.
4. **Check the FBC builds** for the release (e.g. `fbc-logging-6.6`). Releases usually target
   4 OCP versions and consist of two operators, so there are **two groups of four runs** per
   release. The operators build independently — the four FBC builds for one operator may be older
   than the other operator's four. Identify the latest successful **group of four runs** for each
   operator.
5. In each PipelineRun, note the **FBC tag** from the "additional tags" list in the "apply-tags"
   step, e.g.
   `v4.19__operator_nvr__ose-cluster-logging-operator-container-6.5.2-202608132150.p2.g75664fd.assembly.stream.el9`.
6. You now have one eventrouter image and **eight FBC tags**. Prefix each FBC tag with the
   repository to form the full image name:
   `quay.io/redhat-user-workloads/ocp-art-tenant/art-fbc:<tag>`

## Gotchas

- Never mix FBC runs from different "groups of four" — each operator's four runs must come from
  the same latest successful group.
- Never use a staging release that is older than the component builds it should contain.
- The Konflux UI is a JavaScript app: plain HTTP fetches return nothing. Use an authenticated
  `oc` session against the kflux cluster, or ask the user to paste the tags/NVR from the UI.

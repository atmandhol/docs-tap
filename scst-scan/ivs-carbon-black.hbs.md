# Configure a ImageVulnerabilityScan for Carbon Black

To configure an ImageVulnerabilityScan for Carbon Black, use the following ImageVulnerabilityScan and secret configuration:

- Configure Carbon Black CLI with CarbonBlack `cbctl-creds` secret and credentials by using the `~/.cbctl/cbctl.yaml` config file. See the [Carbon Black](https://developer.carbonblack.com/reference/carbon-black-cloud/container/latest/image-scanning-cli#configuration) documentation.
- Set the tekton-pipelines feature-flags configmap `enable-api-fields` to `alpha`. This enables the user to use the `stdoutConfig` which is needed to output the scan report as a file.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: cbctl-creds
stringData:
  cbctl: |
    cb_api_id: CB-API-ID
    cb_api_key: CB-API-KEY
    org_key: ORG-KEY
    saas_url: SAAS-URL
---
apiVersion: app-scanning.apps.tanzu.vmware.com/v1alpha1
kind: ImageVulnerabilityScan
metadata:
  name: carbon-black-ivs
spec:
  image: nginx@sha256:... # The image to be scanned. Digest must be specified.
  scanResults:
    location: registry/project/scan-results
  serviceAccountNames:
    publisher: publisher
    scanner: scanner
  workspace:
    bindings:
    - name: cbctl
      secret:
        secretName: cbctl-creds
        items:
          - key: cbctl
            path: .cbctl.yaml
  steps:
  - name: carbon-black
    image: CARBON-BLACK-SCANNER-IMAGE
    imagePullPolicy: IfNotPresent
    command:
    - cbctl
    - image
    - scan
    - --force=true
    - $(params.image)
    - --config
    - /cbctl/.cbctl.yaml
    - -ocyclonedx
    stdoutConfig:
      path: /workspace/scan-results/scan-results.cdx.xml
```

Where:

- `CB-API-ID` is the API ID obtained from VMware Carbon Black Cloud (CBC).
- `CB-API-KEY` is the API Key obtained from CBC.
- `ORG-KEY` is the Org Key for your CBC organization.
- `SAAS-URL` is the CBC Backend URL.
- `CARBON-BLACK-SCANNER-IMAGE` is the Carbon Black scanner image.

**Note** The Carbon Black `cbctl-creds` secret is mounted as a workspace binding and the credentials are inserted into a `cbctl.yaml` config file that the Carbon Black CLI uses.

**Note** `stdoutConfig.path` is specified to take the output stream of the step to a given file where it can be published to the registry. See [tekton docs](https://github.com/tektoncd/community/blob/main/teps/0011-redirecting-step-output-streams.md) for more details.

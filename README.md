# Nexus IQ Server High Availability Helm Chart

## Location

The helm chart is located in the [chart directory](./chart), see the [helm chart readme](./chart/README.md) for more
information.

## Contributing

See the [contributing document](./CONTRIBUTING.md) for details.

For Sonatypers, note that external contributors must sign the CLA and the Dev-Ex team must verify this prior to
accepting any PR.

## Testing

### Running Lint

Helm's lint command will highlight formatting problems in the chart that need to be corrected.

```
helm lint chart
```

### Running Unit Tests

The existing unit tests are intended to be run using the `helm-unittest` plugin, which can be installed as follows

```
helm plugin install https://github.com/quintush/helm-unittest
```

The test suites are located in the [chart/tests directory](./chart/tests). Each test file name must end in
`_test.yaml` in order for the plugin to automatically execute it.

To run all unit tests, execute `helm unittest --helm3 chart`.

To run an individual unit test, execute `helm unittest --helm3 chart --file chart/tests/<name_test.yaml>`.
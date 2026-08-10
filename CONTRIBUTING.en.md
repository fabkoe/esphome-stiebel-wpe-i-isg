[🇩🇪 Deutsch](CONTRIBUTING.md) · **🇬🇧 English**

# Contributing

Contributions are welcome – issues, value confirmations on other WPE-I/WPM
devices, new Elster indices, docs, dashboard.

## License & sign-off (DCO)

The project is licensed under **Apache-2.0** (see [`LICENSE`](LICENSE)). By
contributing, you agree that your contribution is published under the same
license.

We use the **Developer Certificate of Origin (DCO)** instead of a CLA: certify
the origin of your contribution by signing each commit with `-s` (adds a
`Signed-off-by` line with your name/email):

```bash
git commit -s -m "…"
```

The DCO text: https://developercertificate.org/ – in short: you certify that you
are allowed to contribute the work and that it may be placed under the project
license.

## Safety first (important)

Real costs are attached to the device: wrong **write** values can, per the
manual, damage the heat pump or the screed.

- Only mark new **write** formats as "confirmed" after verification on the real
  device (log **and** display comparison).
- Always give new write entities **conservative** min/max limits.
- Never actively set/test **emergency mode** (operating mode 6).
- Read-only/reading contributions are uncritical and the easiest way to start.

Details on the protocol and the proven procedure:
[`docs/reverse-engineering.md`](docs/reverse-engineering.md).

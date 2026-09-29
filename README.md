# Meridian Pay Pipeline

A minimal Flask app used to demonstrate a secure CI/CD pipeline with software
supply chain security controls, built for a class assignment.

## Project structure

```
.
├── app.py                      # Flask app
├── tests/
│   └── test_app.py             # pytest tests
├── requirements.txt
├── Dockerfile
└── .github/workflows/ci-cd.yml # CI/CD pipeline
```

## Running locally

```bash
pip install -r requirements.txt
python app.py
```

Visit http://localhost:5000 and http://localhost:5000/health.

## Running tests

```bash
pip install -r requirements.txt
pytest -v
```

## Building the container image

```bash
docker build -t meridian-pay .
docker run -p 5000:5000 meridian-pay
```

## CI/CD pipeline

The workflow in `.github/workflows/ci-cd.yml` runs on every push and pull
request to `main`, and implements four supply chain security controls:

| Stage | Tool | What it does |
|---|---|---|
| Automated testing | [pytest](https://docs.pytest.org/) | Runs the test suite before anything else is built or shipped. |
| Vulnerability scanning | [Trivy](https://aquasecurity.github.io/trivy/) | Scans the built container image for known OS and dependency vulnerabilities; the job fails on CRITICAL/HIGH findings. |
| SBOM generation | [Syft](https://github.com/anchore/syft) | Generates a CycloneDX Software Bill of Materials listing every package in the image, uploaded as a build artifact. |
| Image signing | [Cosign](https://docs.sigstore.dev/cosign/overview/) | Signs the published image using keyless (OIDC) signing, so consumers can verify the image was built by this GitHub Actions workflow and hasn't been tampered with. |

On pull requests, the image is only built and scanned (not pushed or signed).
On pushes to `main`, the image is built, scanned, pushed to GitHub Container
Registry (`ghcr.io`), and then its SBOM is generated and it is signed.

### Verifying a signed image

Anyone can verify a signed image without needing any of our private keys,
since Cosign's keyless signing ties the signature to this repository's GitHub
Actions identity:

```bash
cosign verify \
  --certificate-identity-regexp "https://github.com/<owner>/<repo>/.github/workflows/ci-cd.yml@.*" \
  --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
  ghcr.io/<owner>/<repo>:<sha>
```

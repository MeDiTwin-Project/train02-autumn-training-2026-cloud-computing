# Cloud Computing | MediTwin Autumn Training 2026

Course material for the **Cloud Computing** track of the MediTwin Autumn Training 2026, a five-day
hands-on training on cloud computing for scientific workloads. Participants start with virtual
machines and storage and finish with an application that they deploy, monitor and secure on
Kubernetes.

- **Dates:** 2–6 November 2026
- **Venue:** National Technical University of Athens (NTUA), Athens, Greece
- **Read the course online:** <https://training.pages.data.meditwin-project.eu/meditwin-autumn-training-2026>

The material is a [Jupyter Book](https://jupyterbook.org/) (MyST Markdown) in the [`docs/`](docs/)
folder.

## Course contents

| Day | Subject | Main practical work |
| --- | --- | --- |
| 1 | [Cloud Computing Foundations](docs/day01/index.md) | Virtualization, IaaS/PaaS/SaaS, OpenStack, block and object storage, networking, cloud APIs and S3 on a training VM |
| 2 | [Containers & Docker](docs/day02/index.md) | Images, volumes, Dockerfiles, registries and Docker Compose; capstone: the MeDiTwin climate extremes pipeline in containers |
| 3 | [Kubernetes and Helm](docs/day03/index.md) | The Day 2 service on Kubernetes: Deployments, Services, Ingress, TLS, scaling, configuration, storage, GPUs and Helm charts |
| 4 | [Git, CI/CD & GitOps](docs/day04/index.md) | Git, GitLab CI, pipeline security, automated releases and GitOps with Rancher Fleet |
| 5 | [Advanced topics: Monitoring, Security](docs/day05/index.md) | Prometheus and Grafana, security fundamentals, and a group project that packages and serves a model |

See the [agenda](docs/agenda.md) for the prerequisites and sections of each day, and the
[participant instructions](docs/instructions.md) for access to the training environment. To do the
exercises without the training VM, see [Working on your own laptop](docs/laptop-setup.md).

## Build the book locally

Jupyter Book 2 needs Python and Node.js 18 or later.

```bash
python -m venv .venv
source .venv/bin/activate
pip install "jupyter-book>=2,<3"

cd docs
jupyter book start              # live preview while you edit
jupyter book build --html       # static site in docs/_build/html
```

For more information, see the [Jupyter Book documentation](https://jupyterbook.org/stable/).

## Acknowledgement

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="docs/_static/EN_FundedbytheEU_RGB_NEG.png">
  <img src="docs/_static/EN_FundedbytheEU_RGB_POS.png" alt="Funded by the European Union" height="80">
</picture>

Material by the European Centre for Medium-Range Weather Forecasts (ECMWF), 2026.

The MeDiTwin Project has received funding from the European Union's Horizon Europe WIDERA
Coordination and Support Actions under Grant Agreement no. 101159723.

Funded by the European Union. Views and opinions expressed are however those of the author(s) only
and do not necessarily reflect those of the European Union or the European Research Executive
Agency (REA). Neither the European Union nor the granting authority can be held responsible for
them.

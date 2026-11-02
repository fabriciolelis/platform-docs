# Formula Insights architecture

This diagram shows the target Release 1 direction. The highlighted local slice
is the current implementation focus; Kubernetes, GitOps, and AWS are later
stages and are not required to run the service locally.

```mermaid
flowchart TB
    source[Jolpica F1 API\nHistorical Formula 1 data]

    subgraph local[Local development — Sprint 1]
        importer[Idempotent importer\nPython / FastAPI service]
        database[(PostgreSQL)]
        api[Formula Insights API\nHealth · races · results · standings]

        importer -->|upsert season data| database
        api -->|read data| database
    end

    source -->|HTTP / rate-limit-aware requests| importer
    consumer[Analytics team / API consumer] -->|HTTPS| api

    subgraph delivery[Later delivery path]
        ci[GitHub Actions\nTest · build · scan]
        registry[Container registry\nImmutable image]
        gitops[platform-gitops\nReviewed desired state]
        argocd[Argo CD]
        cluster[Kubernetes]
        cronjob[Scheduled importer]
        service[API deployment\nService / Ingress]
        observability[Metrics · logs · alerts]

        ci --> registry
        registry -->|approved image version| gitops
        gitops --> argocd
        argocd --> cluster
        cluster --> cronjob
        cluster --> service
        cronjob -->|upsert| clusterdb[(PostgreSQL\nin-cluster, disposable data)]
        service -->|read| clusterdb
        cronjob --> observability
        service --> observability
    end

    subgraph foundations[Later cloud foundations]
        terraform[platform-infrastructure\nTerraform]
        aws[AWS networking\nand identity]
        terraform --> aws
        aws -. hosts .-> cluster
    end

    docs[platform-docs\nADRs · SLOs · runbooks · reviews]
    docs -. guides .-> local
    docs -. guides .-> delivery
    docs -. guides .-> foundations

    classDef active fill:#d7f5e5,stroke:#16803a,color:#102a18;
    classDef future fill:#e8eefc,stroke:#3d5faa,color:#172544;
    classDef external fill:#fff3d6,stroke:#9a6700,color:#3b2b00;
    classDef docs fill:#f4e8ff,stroke:#7e3fb2,color:#2f1644;

    class importer,database,api active;
    class source,consumer external;
    class ci,registry,gitops,argocd,cluster,cronjob,service,observability,terraform,aws,clusterdb future;
    class docs docs;
```

## Repository ownership

| Area | Repository |
| --- | --- |
| API, importer, database model, image build | `formula-insights-api` |
| Kubernetes desired state and approved image versions | `platform-gitops` |
| Terraform-managed foundations | `platform-infrastructure` |
| Decisions, SLOs, diagrams, runbooks, reviews | `platform-docs` |

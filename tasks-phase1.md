IMPORTANT ❗ ❗ ❗ Please remember to destroy all the resources after each work session. You can recreate infrastructure by creating new PR and merging it to master.

![img.png](doc/figures/destroy.png)
                                                                                                                                                                                                                                                                                                                                                                                  
## Phase 1 Exercise Overview

  ```mermaid
  flowchart TD
      A[🔧 Step 0: Fork repository] --> B[🔧 Step 1: Environment variables\nexport TF_VAR_*]
      B --> C[🔧 Step 2: Bootstrap\nterraform init/apply\n→ GCP project + state bucket]
      C --> D[🔧 Step 3: Quota increase\nCPUS_ALL_REGIONS ≥ 24]
      D --> E[🔧 Step 4: CI/CD Bootstrap\nWorkload Identity Federation\n→ keyless auth GH→GCP]
      E --> F[🔧 Step 5: GitHub Secrets\nGCP_WORKLOAD_IDENTITY_*\nINFRACOST_API_KEY]
      F --> G[🔧 Step 6: pre-commit install]
      G --> H[🔧 Step 7: Push + PR + Merge\n→ release workflow\n→ terraform apply]

      H --> I{Infrastructure\nrunning on GCP}

      I --> J[📋 Task 3: Destroy\nGitHub Actions → workflow_dispatch]
      I --> K[📋 Task 4: New branch\nModify tasks-phase1.md\nPR → merge → new release]
      I --> L[📋 Task 5: Analyze Terraform\nterraform plan/graph\nDescribe selected module]
      I --> M[📋 Task 6: YARN UI\ngcloud compute ssh\nIAP tunnel → port 8088]
      I --> N[📋 Task 7: Architecture diagram\nService accounts + buckets]
      I --> O[📋 Task 8: Infracost\nUsage profiles for\nartifact_registry + storage_bucket]
      I --> P[📋 Task 9: Spark job fix\nAirflow UI → DAG → debug\nFix spark-job.py]
      I --> Q[📋 Task 10: BigQuery\nDataset + external table\non ORC files]
      I --> R[📋 Task 11: Spot instances\npreemptible_worker_config\nin Dataproc module]
      I --> S[📋 Task 12: Auto-destroy\nNew GH Actions workflow\nschedule + cleanup tag]

      style A fill:#4a9eff,color:#fff
      style B fill:#4a9eff,color:#fff
      style C fill:#4a9eff,color:#fff
      style D fill:#ff9f43,color:#fff
      style E fill:#4a9eff,color:#fff
      style F fill:#ff9f43,color:#fff
      style G fill:#4a9eff,color:#fff
      style H fill:#4a9eff,color:#fff
      style I fill:#2ed573,color:#fff
      style J fill:#a55eea,color:#fff
      style K fill:#a55eea,color:#fff
      style L fill:#a55eea,color:#fff
      style M fill:#a55eea,color:#fff
      style N fill:#a55eea,color:#fff
      style O fill:#a55eea,color:#fff
      style P fill:#a55eea,color:#fff
      style Q fill:#a55eea,color:#fff
      style R fill:#a55eea,color:#fff
      style S fill:#a55eea,color:#fff
```

  Legend

  - 🔵 Blue — setup steps (one-time configuration)
  - 🟠 Orange — manual steps (GCP Console / GitHub UI)
  - 🟢 Green — infrastructure ready
  - 🟣 Purple — tasks to complete and document in tasks-phase1.md

1. Authors:

   ***12***

   ***https://github.com/DonAlberton/tbd-workshop-1/***

2. Follow all steps in README.md.

3. From available Github Actions select and run destroy on master branch.

4. Create new git branch and:
    1. Modify tasks-phase1.md file.

    2. Create PR from this branch to **YOUR** master and merge it to make new release.

    ![img.png](doc/figures/successfull-ga.png)
    ***place the screenshot from GA after successful application of release***


5. Analyze terraform code. Play with terraform plan, terraform graph to investigate different modules.
    Module Dataproc creates iam & identity roles, storage (dataproc_staging bucket — stores job artifacts (jars, scripts), dataproc_temp bucket — stores intermediate job data) and the cluster. The cluster consists of a master node (compute engine, 100GB disk size), workers (two compute engines of 100GB disk size each), the image of the machines (Dataproc image) and a network with an internal IP address. The default compute engine specs are vCPUs 2 (shared), 4GB RAM. On startup each node runs pip-intall.sh to install pandas, mlflow, google-cloud-storage, jupyterlab, dbt-core + dbt-spark. HTTP port access is enabled so you can access Jupyter/Spark UIs via the Dataproc web interface.
    ![img.png](doc/figures/terraform-graph.png)
    ***describe one selected module and put the output of terraform graph for this module here***

6. Reach YARN UI
    ```
    gcloud compute ssh tbd-cluster-m \
        --project=tbd-2026l-347426 \
        --zone=europe-west1-b \
        --tunnel-through-iap \
        -- -L 8088:localhost:8088 -N
    ```
    ![img.png](doc/figures/yarn-panel.png)
   ***place the command you used for setting up the tunnel, the port and the screenshot of YARN UI here***

   Hint: the Dataproc cluster has `internal_ip_only = true`, so you need to use an IAP tunnel.
   See: `gcloud compute ssh` with `-- -L <local_port>:localhost:<remote_port>` and `--tunnel-through-iap` flag.
   YARN ResourceManager UI runs on port **8088**.

7. Draw an architecture diagram (e.g. in draw.io) that includes:
    1. Description of the components of service accounts
    2. List of buckets for disposal

    ***place your diagram here***

8. Create a new PR and add costs by entering the expected consumption into Infracost
For all the resources of type: `google_artifact_registry_repository`, `google_storage_bucket`
create a sample usage profiles and add it to the Infracost task in CI/CD pipeline. Usage file [example](https://github.com/infracost/infracost/blob/master/infracost-usage-example.yml)

   ***place the expected consumption you entered here***
    ```
    version: 0.1
    
    resource_usage:
    
    google_artifact_registry_repository.registry:
        storage_gb: 150
        monthly_egress_data_transfer_gb:
        europe-west1-b: 150
    
    
    google_storage_bucket.tbd-code-bucket:
        storage_gb: 50
        monthly_class_a_operations: 2000
        monthly_class_b_operations: 5000
        monthly_egress_data_gb: 20
    
    google_storage_bucket.tbd-data-bucket:
        storage_gb: 500
        monthly_class_a_operations: 5000
        monthly_class_b_operations: 20000
        monthly_egress_data_gb: 100
    
    google_storage_bucket.dataproc_staging:
        storage_gb: 100
        monthly_class_a_operations: 1000
        monthly_class_b_operations: 5000
        monthly_egress_data_gb: 50
    
    google_storage_bucket.dataproc_temp:
        storage_gb: 50
        monthly_class_a_operations: 1000
        monthly_class_b_operations: 5000
        monthly_egress_data_gb: 20
    
    ```
   ***place the screenshot from infracost output here***

   ![img.png](doc/figures/infracost.png)

9. Find and correct the error in spark-job.py

    After `terraform apply` completes, connect to the Airflow cluster:
    ```bash
    gcloud container clusters get-credentials airflow-cluster --zone europe-west1-b --project PROJECT_NAME
    ```
    
    Then check the external IP (AIRFLOW_EXTERNAL_IP) of the webserver service:
    kubectl get svc -n airflow airflow-webserver                                                                                                                                                                 
                                              
                                                                                                                                                                                                               
    ▎ Note: If EXTERNAL-IP shows <pending>, wait a moment and retry — LoadBalancer IP allocation may take 1-2 minutes.  

    DAG files are synced automatically from your GitHub repo via git-sync sidecar.
    Airflow variables and the `google_cloud_default` GCP connection are also configured by Terraform.

    a) In the Airflow UI (http://AIRFLOW_EXTERNAL_IP:8080, login: admin/admin), find the `dataproc_job` DAG, unpause it and trigger it manually.

    ***place a screenshot of the DAG in the Airflow UI***
    ![img.png](doc/figures/DAG_UI.png)

    b) The DAG will fail. Examine the task logs in the Airflow UI to find the root cause.



    Airlfow logs:
    ```
    pyspark_job {
    main_python_file_uri: "gs://tbd-2026l-347426-code/spark-job.py"
    properties {
        key: "spark.executor.memory"
        value: "2g"
    }
    properties {
        key: "spark.executor.instances"
        value: "2"
    }
    properties {
        key: "spark.driver.memory"
        value: "2g"
    }
    }
    status {
    state: ERROR
    details: "Google Cloud Dataproc Agent reports job failure. If logs are available, they can be found at:\nhttps://console.cloud.google.com/dataproc/jobs/3f32e1f3-03d4-4f2e-8b61-f06d54d45f90?project=tbd-2026l-347426&region=europe-west1\ngcloud dataproc jobs wait \'3f32e1f3-03d4-4f2e-8b61-f06d54d45f90\' --region \'europe-west1\' --project \'tbd-2026l-347426\'\nhttps://console.cloud.google.com/storage/browser/tbd-2026l-347426-dataproc-staging/google-cloud-dataproc-metainfo/503d51b2-c2d3-424d-8c60-be01b94e1589/jobs/3f32e1f3-03d4-4f2e-8b61-f06d54d45f90/\ngs://tbd-2026l-347426-dataproc-staging/google-cloud-dataproc-metainfo/503d51b2-c2d3-424d-8c60-be01b94e1589/jobs/3f32e1f3-03d4-4f2e-8b61-f06d54d45f90/driveroutput.*"
    state_start_time {
        seconds: 1776013363
        nanos: 798816000
    }
    }

    ```
    GCP Logs:
    ```
    {
    "code": 404,
    "errors": [
        {
        "domain": "global",
        "message": "The specified bucket does not exist.",
        "reason": "notFound"
        }
    ],
    "message": "The specified bucket does not exist."
    }
    ```
    ***paste the relevant error message from the Airflow task log***

    Wynika z tego, ze bucket źródłowy podany w definicji joba sparkowego nie istnieje. Trzeba go zmienić.

    ***describe what the error is and how you found it***

    c) Fix the error in `modules/data-pipeline/resources/spark-job.py` and re-upload the file to GCS:
    ```bash
    gsutil cp modules/data-pipeline/resources/spark-job.py gs://PROJECT_NAME-code/spark-job.py
    ```
    Then trigger the DAG again from the Airflow UI.

    ***paste the link to the fixed file***

    https://storage.googleapis.com/tbd-2026l-347426-code/spark-job.py

    d) Verify the DAG completes successfully and check that ORC files were written to the data bucket:
    ```bash
    gsutil ls gs://PROJECT_NAME-data/data/shakespeare/
    ```
    Są pliki od 00000 do 00116
    ![img.png](doc/figures/DAG_OK.png)

    ***place a screenshot of the successful DAG run in Airflow UI***

11. Create a BigQuery dataset and an external table using SQL

    Using the ORC data produced by the Spark job in task 9, create a BigQuery dataset and an external table.

    Note: the dataset must be created in the same region as the GCS bucket (`europe-west1`), e.g.:
    ```bash
    bq mk --dataset --location=europe-west1 shakespeare
    ```

    ***place the SQL code and query output here***
    ```
    CREATE OR REPLACE EXTERNAL TABLE `shakespeare.shakespeare`
        OPTIONS (
        format = 'ORC',
        uris = ['gs://tbd-2026l-347426-data/data/shakespeare/*.orc']
        );
    ```
    ***why does ORC not require a table schema?***
    BigQuery korzysta z mechanizmu schema auto-detection. Format ORC przechowuje nalezyte informacje (nazwy kolumn, typy danych) w swoich metadanych.

12. Add support for preemptible/spot instances in a Dataproc cluster

    ***place the link to the modified file and inserted terraform code***

    https://github.com/bdg-tbd/tbd-workshop-1/blob/master/modules/dataproc/main.tf

    ```
    secondary_worker_config {
      num_instances    = 2
      machine_type     = var.machine_type
      preemptibility   = "SPOT" 
      disk_config {
        boot_disk_type    = "pd-standard"
        boot_disk_size_gb = 100
      }
    }
    ```

13. Triggered Terraform Destroy on Schedule or After PR Merge. Goal: make sure we never forget to clean up resources and burn money.

Add a new GitHub Actions workflow that:
  1. runs terraform destroy -auto-approve
  2. triggers automatically:

   a) on a fixed schedule (e.g. every day at 20:00 UTC)

   b) when a PR is merged to master containing [CLEANUP] tag in title

Steps:
  1. Create file .github/workflows/auto-destroy.yml
  2. Configure it to authenticate and destroy Terraform resources
  3. Test the trigger (schedule or cleanup-tagged PR)

Hint: use the existing `.github/workflows/destroy.yml` as a starting point.

***paste workflow YAML here***

***paste screenshot/log snippet confirming the auto-destroy ran***

***write one sentence why scheduling cleanup helps in this workshop***

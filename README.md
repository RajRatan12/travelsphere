# travelsphere
# **Terraform Deployment Guide for Tourism Management App**

## **📌 Overview**
This guide provides a **detailed Terraform implementation** to deploy the **Tourism Management App** on **Google Cloud Platform (GCP)**. The setup includes:
- **VPC & Networking**
- **API Gateway & Load Balancer**
- **GKE (Google Kubernetes Engine) for Microservices**
- **Cloud SQL (PostgreSQL) for Data Storage**
- **Firestore (NoSQL) for User Preferences**
- **Pub/Sub for Event-Driven Processing**
- **Cloud Run & Cloud Functions for Serverless Workloads**
- **Security (IAM, Cloud Armor, Secret Manager)**

---

## **🔹 Prerequisites**
1. Install **Terraform**: [Install Guide](https://developer.hashicorp.com/terraform/downloads)
2. Install **Google Cloud SDK**: [GCP Setup](https://cloud.google.com/sdk/docs/install)
3. Authenticate with GCP:
   ```bash
   gcloud auth application-default login
   gcloud auth login
   ```
4. Enable Required APIs:
   ```bash
   gcloud services enable compute.googleapis.com \
       container.googleapis.com \
       sqladmin.googleapis.com \
       servicenetworking.googleapis.com \
       cloudfunctions.googleapis.com \
       cloudrun.googleapis.com \
       pubsub.googleapis.com
   ```

---

## **1️⃣ Setting Up VPC & Subnets**
```hcl
resource "google_compute_network" "tourism_vpc" {
  name                    = "tourism-network"
  auto_create_subnetworks = false
}

resource "google_compute_subnetwork" "public_subnet" {
  name          = "public-subnet"
  network       = google_compute_network.tourism_vpc.self_link
  ip_cidr_range = "10.10.1.0/24"
  region        = "us-central1"
}

resource "google_compute_subnetwork" "private_subnet" {
  name          = "private-subnet"
  network       = google_compute_network.tourism_vpc.self_link
  ip_cidr_range = "10.10.2.0/24"
  region        = "us-central1"
  private_ip_google_access = true
}
```

---

## **2️⃣ Deploying API Gateway**
```hcl
resource "google_api_gateway_api" "tourism_api" {
  provider = google
  api_id   = "tourism-api"
}

resource "google_api_gateway_api_config" "tourism_api_config" {
  provider  = google
  api       = google_api_gateway_api.tourism_api.api_id
  api_config_id = "v1"
  openapi_documents {
    document {
      path     = "openapi-spec.yaml"
      contents = filebase64("./api-gateway/openapi.yaml")
    }
  }
}
```

---

## **3️⃣ Deploying GKE Cluster**
```hcl
resource "google_container_cluster" "tourism_gke" {
  name     = "tourism-gke"
  location = "us-central1"
  network  = google_compute_network.tourism_vpc.self_link
  subnetwork = google_compute_subnetwork.private_subnet.self_link

  remove_default_node_pool = true
  initial_node_count       = 1
}

resource "google_container_node_pool" "primary_nodes" {
  cluster = google_container_cluster.tourism_gke.name
  node_count = 3

  node_config {
    machine_type = "e2-standard-4"
    oauth_scopes = [
      "https://www.googleapis.com/auth/cloud-platform"
    ]
  }
}
```

---

## **4️⃣ Deploying Cloud SQL (PostgreSQL)**
```hcl
resource "google_sql_database_instance" "tourism_db" {
  name             = "tourism-db"
  database_version = "POSTGRES_14"
  region          = "us-central1"
  settings {
    tier = "db-f1-micro"
    ip_configuration {
      private_network = google_compute_network.tourism_vpc.self_link
    }
  }
}

resource "google_sql_database" "app_db" {
  name     = "app_db"
  instance = google_sql_database_instance.tourism_db.name
}
```

---

## **5️⃣ Deploying Firestore**
```hcl
resource "google_firestore_database" "tourism_firestore" {
  name        = "tourism-firestore"
  location_id = "us-central"
  type        = "NATIVE"
}
```

---

## **6️⃣ Deploying Cloud Run Service**
```hcl
resource "google_cloud_run_service" "tourism_api" {
  name     = "tourism-api"
  location = "us-central1"

  template {
    spec {
      containers {
        image = "gcr.io/YOUR_PROJECT/tourism-api:latest"
      }
    }
  }
}
```

---

## **7️⃣ Deploying Pub/Sub for Event Processing**
```hcl
resource "google_pubsub_topic" "booking_topic" {
  name = "booking-events"
}
```

---

## **8️⃣ Deploying Cloud Function for Notifications**
```hcl
resource "google_cloudfunctions_function" "send_email" {
  name        = "send-email"
  region      = "us-central1"
  runtime     = "nodejs16"
  source_archive_bucket = google_storage_bucket.source_bucket.name
  source_archive_object = google_storage_bucket_object.source_code.name
  entry_point = "sendEmail"
  event_trigger {
    event_type = "google.pubsub.topic.publish"
    resource   = google_pubsub_topic.booking_topic.id
  }
}
```

---

## **9️⃣ Deploying Cloud Armor for Security**
```hcl
resource "google_compute_security_policy" "tourism_security" {
  name = "tourism-security-policy"
  rule {
    action   = "deny(403)"
    priority = 1000
    match {
      expr {
        expression = "origin.region_code == 'RU'"
      }
    }
  }
}
```

---

## **🚀 Deployment Instructions**
```bash
tf init
tf plan
tf apply -auto-approve
```

---

## **📢 Next Steps**
- 🔹 **Deploy Microservices in GKE** using Helm charts.
- 🔹 **Monitor Logs with Cloud Logging**.
- 🔹 **Set up CI/CD with Cloud Build**.

This Terraform script **automates infrastructure deployment**, ensuring a scalable and production-ready system. Let me know if you need **additional configurations or diagrams**! 🚀

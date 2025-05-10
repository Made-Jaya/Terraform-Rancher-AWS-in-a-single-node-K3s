# Panduan Instalasi Rancher di AWS

## Ringkasan Eksekutif
Dokumen ini memberikan panduan lengkap untuk menginstal dan mengkonfigurasi Rancher di AWS, mencakup:
- Quick start installation dengan 1 master node dan 1 worker node
- Spesifikasi lengkap infrastructure (compute, network, storage)
- Panduan konfigurasi untuk development dan production
- Best practices untuk keamanan, backup, dan maintenance
- Strategi scaling dan high availability

### Quick Overview
- Default Setup: Single master node (t3.medium) + Single worker node (t3.medium)

### Arsitektur Cluster Rancher

```
                                  ┌─────────────────────┐
                                  │                     │
                                  │   Rancher Server    │
                                  │   (K3s Cluster)     │
                                  │                     │
                                  └──────────┬──────────┘
                                            │
                                            ▼
                              ┌─────────────────────────────┐
                              │    Cluster Management       │
                              └─────────────┬───────────────┘
                                          │
                    ┌─────────────────────┴──────────────────────┐
                    │                                            │
                    ▼                                            ▼
        ┌─────────────────────┐                    ┌─────────────────────┐
        │   Local Cluster     │                    │  Custom Cluster     │
        │   (K3s)             │                    │  (RKE2)            │
        │   - Rancher         │                    │  - Workloads       │
        │   - System Tools    │                    │  - Applications    │
        └─────────────────────┘                    └─────────────────────┘

Alur Kerja:
1. Rancher Server diinstall di cluster K3s lokal
2. Custom cluster (RKE2) diregister ke Rancher
3. Rancher mengelola semua cluster dari UI yang terpusat
4. Workload dijalankan di custom cluster

```

### Arsitektur Infrastructure AWS

```
┌────────────────────────────────────────────────┐
│                   AWS VPC                      │
│                (10.0.0.0/16)                  │
│                                               │
│  ┌─────────────────┐      ┌─────────────────┐ │
│  │  Master Node    │      │   Worker Node   │ │
│  │   (Rancher)    │      │   (Workload)    │ │
│  │   t3.medium    │      │   t3.medium     │ │
│  │   K3s Server   │      │   RKE2 Agent    │ │
│  └─────────────────┘      └─────────────────┘ │
│          │                       │            │
│          └───────────┬───────────┘            │
│                      │                        │
│            ┌──────────────────┐               │
│            │  Load Balancer   │               │
│            └──────────────────┘               │
│                      │                        │
└──────────────────────┼────────────────────────┘
                       │
                    Internet

Untuk Multiple Workers:
┌────────────────────────────────────────────────┐
│                   AWS VPC                      │
│                                               │
│  ┌─────────────┐   ┌──────────────────────┐   │
│  │Master Node  │   │    Worker Nodes      │   │
│  │  Rancher   │   │                      │   │
│  │            │   │  ┌───┐ ┌───┐ ┌───┐   │   │
│  │            │◄──┼──┤W1 │ │W2 │ │W3 │   │   │
│  │            │   │  └───┘ └───┘ └───┘   │   │
│  └─────────────┘   │     Auto Scale      │   │
│                    └──────────────────────┘   │
└────────────────────────────────────────────────┘
```
- Versi Software: Rancher v2.7.9, K3s v1.24.14+k3s1, RKE2 v1.24.14+rke2r1
- Region: Configurable (default ap-southeast-3 Jakarta)
- Estimated Setup Time: 15-20 menit
- Estimated AWS Cost: Varies by region dan usage

### Use Cases
- Development dan Testing
- Proof of Concept
- Small to Medium Production (dengan additional configuration)

## Prasyarat
1. AWS Account dengan akses credentials (Access Key dan Secret Key)
2. Terraform terinstall di komputer lokal
3. Git terinstall di komputer lokal

## Langkah-langkah Instalasi

### 1. Clone Repository
```bash
git clone https://github.com/rancher/quickstart
cd quickstart/rancher/aws
```

### 2. Konfigurasi AWS Credentials
1. Copy file terraform.tfvars.example menjadi terraform.tfvars:
```bash
cp terraform.tfvars.example terraform.tfvars
```

2. Edit file terraform.tfvars dan isi informasi berikut:
```hcl
# AWS Access Key
aws_access_key = "YOUR_AWS_ACCESS_KEY"

# AWS Secret Key
aws_secret_key = "YOUR_AWS_SECRET_KEY"

# Password untuk admin Rancher (minimal 12 karakter)
rancher_server_admin_password = "admin-password-here"

# AWS Region (pilih yang terdekat)
# Contoh: ap-southeast-3 untuk Jakarta
aws_region = "ap-southeast-3"

# Availability Zone
aws_zone = "ap-southeast-3a"

# Instance Type
instance_type = "t3.medium"
```

### 3. Deploy Infrastructure

1. Inisialisasi Terraform:
```bash
terraform init
```

2. Deploy infrastructure:
```bash
terraform apply --auto-approve
```

Proses deployment akan memakan waktu sekitar 10-15 menit. Setelah selesai, Terraform akan menampilkan output berupa:
- rancher_server_url: URL untuk mengakses Rancher dashboard
- rancher_node_ip: IP address dari Rancher server
- workload_node_ip: IP address dari worker node

### 4. Akses Rancher Dashboard

1. Buka rancher_server_url di browser (contoh: https://rancher.xx.xx.xx.xx.sslip.io)
2. Login menggunakan credentials:
   - Username: admin
   - Password: password yang diset di terraform.tfvars (rancher_server_admin_password)

++++++++++++======================BERHASIL==========++++++++++++++++++++++++++++++++++++ 
### 5. Struktur Cluster Rancher

#### Management Server (Control Plane)
- Instance Type: t3.medium
- Running Rancher v2.7.9 di atas K3s
- Mengelola dan mengontrol semua downstream clusters
- URL: https://rancher.xx.xx.xx.xx.sslip.io

#### Downstream Cluster yang Diregister
Setelah instalasi, Anda akan memiliki:
1. **Local Cluster**: Cluster K3s yang menjalankan Rancher server
2. **Default Custom Cluster**: RKE2 cluster yang diregister ke Rancher
   - Nama: quickstart-aws-custom
   - 1 node dengan RKE2 v1.24.14+rke2r1
   - Digunakan untuk workload aplikasi

#### Menambah Custom Cluster Baru
1. Login ke Rancher UI
2. Klik "Create" di bagian Clusters
3. Pilih "Custom"
4. Isi informasi cluster:
   - Nama cluster
   - Kubernetes version
   - Network provider
   - Node options

#### Meregister Existing Cluster
1. Di Rancher UI, klik "Create" -> "Import Existing"
2. Pilih jenis cluster (Generic, EKS, GKE, etc)
3. Ikuti instruksi untuk mengapply manifest ke cluster

#### Infrastructure Details
- Instance Type: t3.medium
  * 2 vCPU
  * 4 GB Memory
  * 40 GB root volume (gp2)
- OS: SLES 15 SP3
- Kubernetes: K3s v1.24.14+k3s1
- Role: Control plane untuk Rancher management
- Jumlah node: 1 (single-node setup)

#### Worker Node
- Instance Type: t3.medium
  * 2 vCPU
  * 4 GB Memory
  * 40 GB root volume (gp2)
- OS: SLES 15 SP3
- Kubernetes: RKE2 v1.24.14+rke2r1
- Role: Worker node untuk workload
- Jumlah node: 1 (default, dapat ditambah)

#### Network Configuration
- VPC baru dengan CIDR: 10.0.0.0/16
- Subnet: 10.0.0.0/24
- Security Group: All traffic allowed (customize untuk production)
- DNS: sslip.io untuk Rancher URL

#### Rancher Installation
- Version: 2.7.9
- Cert-Manager: v1.11.0
- High Availability: Tidak (single-node setup)
- Ingress: Traefik (default dari K3s)
- Default Storage Class: AWS EBS

### 6. Kustomisasi Infrastructure

Untuk mengubah spesifikasi infrastructure, edit file terraform.tfvars:

#### Mengubah Instance Type
```hcl
# Untuk master dan worker nodes
instance_type = "t3.large"  # Upgrade ke instance yang lebih besar
```

#### Menambah Worker Node
```hcl
# Tambahkan worker node Windows (opsional)
add_windows_node = true
windows_instance_type = "t3a.large"
```

#### Mengubah Region/Zone
```hcl
# Ubah region dan availability zone
aws_region = "ap-southeast-3"  # Jakarta
aws_zone = "ap-southeast-3a"
```

#### Mengubah Versi Kubernetes
```hcl
# Di bagian rancher_kubernetes_version
rancher_kubernetes_version = "v1.24.14+k3s1"

# Di bagian workload_kubernetes_version
workload_kubernetes_version = "v1.24.14+rke2r1"
```

### Menambah Worker Nodes

Untuk menambah worker nodes, ikuti langkah-langkah berikut:

#### 1. Melalui Terraform (Sebelum Deploy)
Tambahkan variabel berikut di terraform.tfvars:
```hcl
# Jumlah worker nodes yang diinginkan
worker_node_count = 3  # Default: 1

# Konfigurasi worker node
worker_instance_type = "t3.medium"  # Sesuaikan dengan kebutuhan
worker_volume_size = "50"  # Size dalam GB
```

#### 2. Melalui Rancher UI (Setelah Deploy)
1. Login ke Rancher dashboard
2. Pilih cluster yang ingin ditambah nodenya
3. Klik "Edit Cluster"
4. Di bagian "Machine Pools":
   - Klik "Add Node Pool"
   - Set jumlah nodes yang diinginkan
   - Pilih instance type
   - Konfigurasi labels/taints jika diperlukan
5. Klik "Save" untuk apply perubahan

#### 3. Menggunakan AWS Auto Scaling Group
1. Buat Launch Template:
   ```bash
   aws ec2 create-launch-template \
     --launch-template-name "rancher-worker" \
     --version-description "Worker node template" \
     --launch-template-data '{"InstanceType":"t3.medium"}'
   ```

2. Buat Auto Scaling Group:
   ```bash
   aws autoscaling create-auto-scaling-group \
     --auto-scaling-group-name "rancher-workers" \
     --launch-template "LaunchTemplateName=rancher-worker,Version=1" \
     --min-size 2 \
     --max-size 5 \
     --desired-capacity 3 \
     --vpc-zone-identifier "subnet-xxx,subnet-yyy"
   ```

3. Konfigurasi Scaling Policy:
   ```bash
   aws autoscaling put-scaling-policy \
     --auto-scaling-group-name "rancher-workers" \
     --policy-name "cpu-scaling" \
     --policy-type "TargetTrackingScaling" \
     --target-tracking-configuration '{"TargetValue":70,"PredefinedMetricSpecification":{"PredefinedMetricType":"ASGAverageCPUUtilization"}}'
   ```

#### Best Practices untuk Multiple Workers
1. **High Availability**:
   - Distribusikan nodes ke multiple Availability Zones
   - Minimal 3 worker nodes untuk HA
   - Gunakan node anti-affinity untuk workload critical

2. **Resource Management**:
   - Set resource requests dan limits untuk pods
   - Implementasi node selectors dan taints
   - Monitor resource usage per node

3. **Scaling**:
   - Set minimum dan maximum nodes
   - Gunakan metrics yang sesuai untuk auto-scaling
   - Consider cost vs performance trade-offs

4. **Maintenance**:
   - Lakukan rolling updates
   - Backup data sebelum scaling down
   - Monitor node health secara regular

### 7. Rekomendasi Production Setup

#### Minimal Requirements untuk Production
1. Master Node (Rancher Control Plane):
   - Minimal 2 node untuk High Availability
   - Rekomendasi: 3 node untuk full HA
   - Instance type minimal: t3.large (2 vCPU, 8 GB RAM)
   - Storage: 50 GB SSD per node
   - Dedicated node (tidak shared dengan workload)

2. Worker Node (Workload Cluster):
   - Minimal 3 node untuk workload
   - Instance type sesuai workload (minimal t3.large)
   - Storage: 50 GB SSD per node + additional EBS sesuai kebutuhan
   - Auto Scaling Group untuk elasticity

3. Network:
   - Dedicated VPC
   - Multiple Availability Zones
   - Private subnets untuk nodes
   - Public subnets untuk load balancers
   - NAT Gateway untuk egress traffic
   - AWS Application Load Balancer untuk ingress

4. Backup:
   - S3 bucket untuk etcd backup
   - Regular snapshot untuk EBS volumes
   - Cross-region backup untuk DR

#### Scaling Strategy
1. Vertical Scaling:
   - Upgrade instance type untuk performa lebih baik
   - Tambah CPU/Memory sesuai kebutuhan
   - Gunakan instance type dari r6i family untuk memory-intensive workloads

2. Horizontal Scaling:
   - Tambah worker nodes untuk kapasitas lebih besar
   - Implementasi auto-scaling berdasarkan metrics
   - Distribusi nodes ke multiple AZ

3. Storage Scaling:
   - Gunakan EBS volumes untuk persistent storage
   - Implementasi dynamic provisioning
   - Setup backup dan retention policy

### 6. SSH ke Instance
File SSH key digenerate secara otomatis dan disimpan di direktori project:
- Private key: ./id_rsa
- Public key: ./id_rsa.pub

Untuk SSH ke instance:
```bash
ssh -i id_rsa ec2-user@<rancher_node_ip>
```

### 7. Menghapus Infrastructure
Untuk menghapus semua resource yang dibuat:
```bash
terraform destroy --auto-approve
```

## Troubleshooting

### Instance Type Tidak Tersedia
Jika mendapat error instance type tidak tersedia di region tertentu, ganti instance_type di terraform.tfvars ke tipe instance yang tersedia di region tersebut. Contoh:
- t3.medium (tersedia di sebagian besar region)
- t3a.medium (tidak tersedia di semua region)

### SSL Certificate Warning
Browser mungkin menampilkan warning SSL certificate karena menggunakan self-signed certificate. Ini normal untuk environment development/testing.

### Timeout saat Destroy
Jika terjadi timeout saat menjalankan terraform destroy, tunggu beberapa menit dan coba jalankan destroy command lagi.

## Keamanan

### Firewall Rules
Deployment ini membuat security group yang mengizinkan akses dari semua IP (0.0.0.0/0). Untuk production environment, sebaiknya:
1. Batasi IP range yang bisa mengakses Rancher dashboard
2. Gunakan VPN untuk akses management
3. Aktifkan AWS GuardDuty untuk monitoring keamanan

### Password
- Gunakan password yang kuat untuk Rancher admin (minimal 12 karakter)
- Aktifkan Multi-Factor Authentication (MFA) setelah instalasi
- Ganti password secara berkala
- Jangan share credential melalui plain text

### Backup
Sangat disarankan untuk:
1. Setup regular backup untuk Rancher
2. Backup etcd dari Kubernetes cluster
3. Simpan terraform state file dengan aman
4. Dokumentasikan semua konfigurasi custom

## Biaya
Perhatikan bahwa deployment ini akan menimbulkan biaya di AWS untuk:
1. EC2 instances (2 instance t3.medium)
2. EBS volumes
3. Data transfer
4. Elastic IP (jika digunakan)

Pastikan untuk menghapus semua resource dengan `terraform destroy` jika sudah tidak digunakan untuk menghindari biaya yang tidak perlu.

## Maintenance dan Monitoring

### Update dan Patch
1. Update Rancher secara berkala untuk mendapatkan fitur terbaru dan security patches
2. Monitor security advisory dari Rancher dan K3s
3. Update sistem operasi instance secara regular
4. Pastikan semua node memiliki resource yang cukup

### Monitoring
1. Setup monitoring menggunakan Rancher monitoring stack:
   - Prometheus untuk metrics collection
   - Grafana untuk visualisasi
   - AlertManager untuk alerting

2. Monitor metrics penting:
   - CPU & Memory usage
   - Disk usage
   - Network traffic
   - Pod status
   - Node health

### Log Management
1. Aktifkan logging untuk:
   - Rancher server logs
   - Kubernetes system logs
   - Application logs
   - AWS CloudWatch logs

2. Setup log retention policy yang sesuai dengan kebutuhan

### Health Check
Lakukan pengecekan rutin untuk:
1. Kubernetes cluster health
2. Node status
3. Network connectivity
4. Backup status
5. Security compliance

## Otomatisasi Cloudflare dengan Terraform

### Persiapan
1. Token API Cloudflare
2. Provider Cloudflare untuk Terraform
3. Domain yang sudah terdaftar di Cloudflare

### Konfigurasi Terraform untuk Cloudflare

#### 1. Tambahkan Provider Cloudflare
Tambahkan di file `provider.tf`:
```hcl
terraform {
  required_providers {
    cloudflare = {
      source = "cloudflare/cloudflare"
      version = "~> 4.0"
    }
  }
}

provider "cloudflare" {
  api_token = var.cloudflare_api_token
}
```

#### 2. Tambahkan Variabel
Di file `variables.tf`:
```hcl
variable "cloudflare_api_token" {
  type = string
  description = "Cloudflare API token"
}

variable "cloudflare_zone_id" {
  type = string
  description = "Cloudflare Zone ID for your domain"
}

variable "domain_name" {
  type = string
  description = "Your domain name (e.g., example.com)"
}
```

#### 3. Konfigurasi di terraform.tfvars
```hcl
# Cloudflare configuration
cloudflare_api_token = "your-api-token"
cloudflare_zone_id = "your-zone-id"
domain_name = "example.com"
hostname = "rancher.example.com"
```

#### 4. Tambahkan Resource Cloudflare
Buat file baru `cloudflare.tf`:
```hcl
# DNS Record untuk Rancher
resource "cloudflare_record" "rancher" {
  zone_id = var.cloudflare_zone_id
  name    = "rancher"
  value   = aws_instance.rancher_server.public_ip
  type    = "A"
  proxied = true
}

# SSL/TLS Settings
resource "cloudflare_zone_settings_override" "rancher_settings" {
  zone_id = var.cloudflare_zone_id
  settings {
    ssl = "full_strict"
    always_use_https = "on"
    min_tls_version = "1.2"
    tls_1_3 = "on"
    automatic_https_rewrites = "on"
    opportunistic_encryption = "on"
    universal_ssl = "on"
  }
}

# Firewall Rules
resource "cloudflare_filter" "rancher" {
  zone_id = var.cloudflare_zone_id
  description = "Filter for Rancher access"
  expression = "(http.request.uri.path contains \"/v3/\" or http.request.uri.path contains \"/dashboard\")"
}

resource "cloudflare_firewall_rule" "rancher" {
  zone_id = var.cloudflare_zone_id
  description = "Rancher Security Rule"
  filter_id = cloudflare_filter.rancher.id
  action = "allow"
  priority = 1
}

# Page Rules
resource "cloudflare_page_rule" "rancher_ssl" {
  zone_id = var.cloudflare_zone_id
  target = "rancher.${var.domain_name}/*"
  priority = 1

  actions {
    ssl = "full_strict"
    always_use_https = true
  }
}
```

#### 5. Update Output
Tambahkan di `output.tf`:
```hcl
output "rancher_url" {
  value = "https://${cloudflare_record.rancher.hostname}"
}
```

### Langkah-langkah Implementasi

1. Generate Cloudflare API Token:
   - Login ke dashboard Cloudflare
   - Pergi ke User Profile > API Tokens
   - Create Token dengan permissions:
     * Zone.DNS (Edit)
     * Zone.Settings (Edit)
     * Zone.Page Rules (Edit)

2. Dapatkan Zone ID:
   - Di dashboard Cloudflare
   - Pilih domain Anda
   - Zone ID ada di Overview page

3. Apply Konfigurasi:
```bash
terraform init  # Install provider Cloudflare
terraform apply # Deploy semua konfigurasi
```

### Verifikasi
1. Check DNS record terbuat:
```bash
dig rancher.yourdomain.com
```

2. Verifikasi SSL/TLS:
```bash
curl -I https://rancher.yourdomain.com
```

### Maintenance
- Simpan API token dengan aman
- Update provider Cloudflare secara berkala
- Monitor perubahan konfigurasi melalui Terraform state
- Backup terraform state file

## Konfigurasi Custom Domain dan HTTPS dengan Cloudflare Manual

### Persiapan
1. Domain terdaftar di Cloudflare
2. Akses ke Cloudflare dashboard
3. Rancher server sudah terinstall

### Langkah-langkah Konfigurasi

#### 1. Setup di Terraform (Sebelum Deploy)
1. Edit file terraform.tfvars:
```hcl
# Tambahkan konfigurasi hostname
hostname = "rancher.yourdomain.com"  # Ganti dengan domain Anda
```

2. Deploy atau re-deploy dengan terraform apply

#### 2. Setup DNS di Cloudflare
1. Login ke dashboard Cloudflare
2. Pilih domain Anda
3. Pergi ke menu DNS
4. Tambahkan A record:
   - Name: rancher (atau subdomain yang diinginkan)
   - IPv4 address: [IP Rancher Server]
   - Proxy status: DNS only (grey cloud) untuk sementara

#### 3. Konfigurasi SSL/TLS di Cloudflare
1. Pergi ke menu SSL/TLS
2. Set SSL/TLS encryption mode ke "Full (strict)"
3. Di bagian Edge Certificates:
   - Aktifkan "Always Use HTTPS"
   - Aktifkan "Automatic HTTPS Rewrites"
   - Aktifkan "TLS 1.3"

#### 4. Update Ingress (Jika Deploy Sudah Ada)
1. SSH ke Rancher server:
```bash
ssh -i id_rsa ec2-user@<rancher_node_ip>
```

2. Update ingress dengan domain baru:
```bash
kubectl -n cattle-system patch ingress rancher --patch '{
    "spec": {
        "rules": [{
            "host": "rancher.yourdomain.com",
            "http": {
                "paths": [{
                    "pathType": "ImplementationSpecific",
                    "backend": {
                        "service": {
                            "name": "rancher",
                            "port": {"number": 80}
                        }
                    }
                }]
            }
        }]
    }
}'
```

#### 5. Verifikasi Setup
1. Tunggu DNS propagation (bisa memakan waktu hingga 24 jam)
2. Test akses HTTPS:
```bash
curl -I https://rancher.yourdomain.com
```
3. Setelah berhasil, ubah Proxy status di Cloudflare menjadi "Proxied" (orange cloud)

### Troubleshooting Domain dan SSL
1. **DNS Issues**:
   - Verifikasi A record di Cloudflare
   - Check domain propagation: `dig rancher.yourdomain.com`
   - Pastikan IP Rancher server benar

2. **SSL Errors**:
   - Pastikan mode SSL/TLS di Cloudflare sesuai
   - Verifikasi certificate Rancher valid
   - Check ingress controller logs

3. **Access Issues**:
   - Verifikasi security group mengizinkan port 80/443
   - Check Cloudflare firewall rules
   - Pastikan domain routing benar

### Best Practices untuk Custom Domain
1. Gunakan dedicated subdomain untuk Rancher
2. Aktifkan semua security features Cloudflare:
   - WAF (Web Application Firewall)
   - Rate limiting
   - Bot protection
3. Setup monitoring untuk SSL certificate
4. Backup konfigurasi DNS dan SSL
5. Dokumentasikan semua custom settings
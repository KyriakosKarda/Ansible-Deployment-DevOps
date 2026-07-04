# ⚙️ Ansible Deployment & DevOps Automation

Αυτό το repository περιέχει τα **Ansible playbooks** που χρησιμοποιούνται για το αυτοματοποιημένο **deployment, provisioning και configuration management** των remote VMs στις οποίες τρέχει η εφαρμογή **[Street Food Go](https://github.com/KyriakosKarda/Project-DS-Street_Food_Go)**.

Καλείται αυτόματα ως τελευταίο στάδιο (`Deploy App To VM`) του Jenkins pipeline της κύριας εφαρμογής, αλλά μπορεί να εκτελεστεί και χειροκίνητα από οποιοδήποτε control machine με SSH πρόσβαση στα target VMs.

---

## 📋 Πίνακας Περιεχομένων

- [Τι κάνει](#-τι-κάνει)
- [Δομή Project](#-δομή-project)
- [Inventory & Μεταβλητές](#-inventory--μεταβλητές)
- [Playbooks](#-playbooks)
- [Προαπαιτούμενα](#-προαπαιτούμενα)
- [Εκτέλεση](#-εκτέλεση)
- [Ενσωμάτωση με το CI/CD της εφαρμογής](#-ενσωμάτωση-με-το-cicd-της-εφαρμογής)

---

## 🎯 Τι κάνει

Οι playbooks του repo καλύπτουν 3 βασικά στάδια provisioning ενός VM ώστε να μπορεί να φιλοξενήσει την εφαρμογή:

1. **Εγκατάσταση Docker** (`docker.yml`) — Docker Engine, `docker-compose` plugin, docker group setup.
2. **Deployment της εφαρμογής** (`spring.yml`) — αντιγραφή του `docker-compose.yml` στο VM και εκτέλεση `docker compose up`.
3. **Reverse proxy με Nginx** (`nginx.yml`) — εγκατάσταση/ρύθμιση Nginx ώστε να προωθεί το traffic από την port 80 στο container της εφαρμογής.

Τα VMs υποστηρίζονται ως ένα **inventory group** (`app-server`), οπότε το ίδιο σύνολο playbooks μπορεί να τρέξει παράλληλα σε πολλαπλά περιβάλλοντα (π.χ. VM σε AWS και VM σε Azure ταυτόχρονα).

---

## 📁 Δομή Project

```
Ansible-Deployment-DevOps/
├── ansible.cfg               # Βασικές ρυθμίσεις Ansible (inventory path, host key checking)
├── hosts.yml                  # Inventory: ορισμός των target VMs (ομάδα "app-server")
├── group_vars/
│   └── all.yml                 # Μεταβλητές κοινές για όλα τα hosts (π.χ. app_port)
├── host_vars/
│   ├── application-vm.yml       # Μεταβλητές ειδικά για το πρώτο VM (π.χ. AWS)
│   └── application-vm2.yml       # Μεταβλητές ειδικά για το δεύτερο VM (π.χ. Azure)
├── playbooks/
│   ├── docker.yml                # Εγκατάσταση Docker Engine + docker-compose
│   ├── spring.yml                 # Deploy του docker-compose.yml της εφαρμογής & εκκίνηση
│   └── nginx.yml                   # Εγκατάσταση & ρύθμιση Nginx ως reverse proxy
└── templates/
    └── nginx.http.j2               # Jinja2 template για το Nginx server block
```

---

## 🗂 Inventory & Μεταβλητές

Το `hosts.yml` ορίζει την ομάδα `app-server` με δύο hosts:

```yaml
app-server:
  hosts:
    application-vm:
      ansible_host: aws-vm
    application-vm2:
      ansible_host: azure-vm
```

> Τα `aws-vm` / `azure-vm` αναμένεται να είναι aliases ορισμένα στο `~/.ssh/config` του control machine (host, IP, user, key path).

**Μεταβλητές:**

| Αρχείο | Scope | Περιεχόμενο |
|---|---|---|
| `group_vars/all.yml` | Όλα τα hosts | `app_port: 8080` — η port στην οποία ακούει το Spring Boot container, χρησιμοποιείται από το Nginx template. |
| `host_vars/application-vm.yml` | Μόνο `application-vm` | Repo URL, path προορισμού και branch της εφαρμογής (`spring_app_repo`, `spring_app_dest`, `spring_app_version`). |
| `host_vars/application-vm2.yml` | Μόνο `application-vm2` | Διαφορετικό path προορισμού (`/home/azureuser/spring_app`), λόγω διαφορετικού OS user. |

---

## 📜 Playbooks

### `playbooks/docker.yml`
Τρέχει σε όλο το group `app-server`. Εγκαθιστά prerequisites, προσθέτει το official Docker APT repository, εγκαθιστά Docker Engine + `docker-compose-plugin`, δημιουργεί compatibility wrapper `docker-compose` → `docker compose`, και προσθέτει τον χρήστη στο `docker` group ώστε να τρέχει Docker χωρίς `sudo`.

### `playbooks/spring.yml`
Τρέχει σε όλο το group `app-server`. Δημιουργεί το directory `/opt/spring_app`, αντιγράφει εκεί το `docker-compose.yml` της κύριας εφαρμογής και εκτελεί `docker compose up` μέσω του module `community.docker.docker_compose_v2`.

### `playbooks/nginx.yml`
Τρέχει μόνο στο `application-vm`. Εγκαθιστά Nginx, παράγει configuration από το template `nginx.http.j2` (proxy προς `localhost:{{ app_port }}`), το ενεργοποιεί ως `sites-enabled`, απενεργοποιεί το default site, επαληθεύει το config με `nginx -t` και κάνει restart το service.

---

## 🧰 Προαπαιτούμενα

Στο **control machine** (π.χ. το laptop σου ή τον Jenkins agent):

```bash
sudo apt update
sudo apt install ansible -y
```

Επιπλέον χρειάζεται:
- Ρυθμισμένο **SSH access** από το control machine προς τα target VMs (keys, `~/.ssh/config` entries).
- Το collection **`community.docker`** εγκατεστημένο (απαιτείται από το `spring.yml`):
  ```bash
  ansible-galaxy collection install community.docker
  ```

---

## 🚀 Εκτέλεση

1. Προσάρμοσε το `hosts.yml` και τα `group_vars/` / `host_vars/` στα δικά σου VMs.
2. Εκτέλεσε το playbook που χρειάζεσαι:

```bash
# Εγκατάσταση Docker στα VMs
ansible-playbook -i hosts.yml playbooks/docker.yml

# Deploy της εφαρμογής (docker compose up)
ansible-playbook -i hosts.yml playbooks/spring.yml

# Ρύθμιση Nginx reverse proxy
ansible-playbook -i hosts.yml playbooks/nginx.yml
```

---

## 🔗 Ενσωμάτωση με το CI/CD της εφαρμογής

Αυτό το repository δεν τρέχει αυτόνομα ένα δικό του CI/CD — καλείται από το **Jenkinsfile** του κύριου project **Street Food Go**, στο στάδιο `Deploy App To VM`:

```groovy
stage('Deploy App To VM') {
    steps {
        dir('deploy_to_vm') {
            git branch: 'main',
                credentialsId: 'git-private-key',
                url: 'git@github.com:KyriakosKarda/Ansible-Deployment-DevOps.git'

            withCredentials([...]) {
                sh 'ansible-playbook -i hosts.yml playbooks/spring.yml'
            }
        }
    }
}
```

Δηλαδή: μόλις το Jenkins pipeline κάνει build & push το Docker image στο GHCR, κλωνοποιεί αυτό το repo και τρέχει το `spring.yml` playbook ώστε το νέο `docker-compose.yml` να «κατέβει» στο VM και να ξεκινήσει.

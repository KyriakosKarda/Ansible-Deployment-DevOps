# Ansible Deployment & DevOps Automation

Αυτό το repository περιέχει **Ansible playbooks** για αυτοματοποιημένο deployment και διαχείριση υποδομών.

---

## 🧰 Prerequisites

### Για Ubuntu / Debian

```bash
sudo apt update
sudo apt install ansible -y

## 📦 Περιεχόμενα του Repo

Το repository περιλαμβάνει:

- `ansible.cfg` – Ρύθμισεις για την Ansible (Δεν θα χρειαστεί ως το παρόν να το αλλάξετε).
- `hosts.yml` – Inventory αρχείο ορισμού hosts (Σημαντικό να το αλλάξετε ώστε να τρέχει για τα Δικά σας VM)
- `group_vars/` – Μεταβλητές που εφαρμόζονται σε ομάδες hosts. (Θα βρείτε γενικές ρυθμίσεις για την εφαρμογή)
- `host_vars/` – Μεταβλητές ανά host. (Θα βρείτε το στοιχεία για git repo)
- `playbooks/` – Φάκελος με playbooks (Δεν θα χρειαστεί ως το παρόν να το αλλάξετε).
- `templates/` – Jinja2  για δυναμική δημιουργία αρχείων.

## 🚀 Απαιτήσεις

Για να εκτελέσεις αυτό το project:

1. Ρύθμιστε SSH connection από το control machine(το laptop σας) στους target hosts.  
2. Τροποποίηστε τα hosts.yml και group/host variables σύμφωνα με τις ανάγκες σας.

## 🗂️ Παράδειγμα Inventory (hosts.yml)

Παρακάτω φαίνεται ένα παράδειγμα ορισμού inventory σε YAML μορφή, με ομάδες και hosts:

```yaml
app-server:               # <-- group
  hosts:                  # <-- list of hosts in group
   application-vm:          # <-- host 1
      ansible_host: x.x.x.x
      ansible_port: 22  
      ansible_ssh_user: ubuntu
    app01:                # <-- host 2
      ansible_host: app01   # Αυτός Ο host χρησιμοποιεί ρωτάει το ~/.ssh/config file για το όνομα 'app01'

    Host app01
    HostName 35.189.109.16
    User rg
    Port 22
    IdentityFile ~/.ssh/id_rsa  #Εδώ βάζουμε το path για το private rsa key που έχουμε κάνει generate ώστε να γίνει το ssh connection

🛠️ Εκτέλεση Playbooks

Για να τρέξεις ένα playbook:

   
    ansible-playbook -i hosts.yml playbooks/<your-playbook>.yml

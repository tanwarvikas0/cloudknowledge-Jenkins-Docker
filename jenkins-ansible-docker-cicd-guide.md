# Jenkins + Ansible + Docker CI/CD Project — Complete Setup Guide

Yeh guide AWS par server create karne se lekar GitHub account setup, Jenkins job, Ansible playbook, aur Docker deployment tak — sab steps cover karti hai.

---

## Part 0: GitHub Account & Repository Setup

### Step 1: GitHub Account Banana
1. https://github.com par jao
2. **Sign up** par click karo
3. Username, email, password daalo → account verify karo

### Step 2: New Repository Create Karna
1. Login ke baad top-right corner mein **+** icon → **New repository**
2. Repository name: `cloudknowledge` (ya jo naam chaho)
3. **Public** select karo (private bhi chalega, bas Jenkins access dena padega)
4. **Create repository** click karo

### Step 3: Dockerfile Repository Mein Add Karna
1. Repo ke andar **Add file → Create new file**
2. File name: `Dockerfile`
3. Content paste karo:

```dockerfile
FROM centos:latest
MAINTAINER sanjay.dahiya332@gmail.com
RUN yum install -y httpd \
    zip \
    unzip
ADD https://www.free-css.com/assets/files/free-css-templates/download/page247/kindle.zip /var/www/html/
WORKDIR /var/www/html
RUN unzip kindle.zip
RUN cp -rvf markups-kindle/* .
RUN rm -rf __MACOSX markups-kindle kindle.zip
CMD ["/usr/sbin/httpd", "-D", "FOREGROUND"]
EXPOSE 80
```

4. **Commit new file** click karo

### Step 4: Personal Access Token Generate Karna (Webhook ke liye baad mein chahiye)
1. Profile icon → **Settings**
2. Left sidebar mein sabse niche **Developer settings**
3. **Personal access tokens → Tokens (classic)**
4. **Generate new token (classic)**
5. Scope mein `repo` aur `admin:repo_hook` check karo
6. Token generate karke **copy karke safe jagah save karo** (yeh dubara nahi dikhega)

---

## Part 1: AWS Par 3 Servers Create Karna

### Step 5: EC2 Instances Launch Karna
AWS Console → EC2 → **Launch Instance**, teen baar (ya ek saath 3 instances) yeh details ke saath:

| Server | Name | AMI | Instance Type |
|---|---|---|---|
| 1 | jenkins-server | CentOS/Amazon Linux | t2.medium (min 2GB RAM) |
| 2 | ansible-server | CentOS/Amazon Linux | t2.medium |
| 3 | docker-host | CentOS/Amazon Linux | t2.micro |

**Security Group mein yeh ports open karo (sabke liye):**
- SSH (22) – Anywhere
- HTTP (80) – Anywhere
- Custom TCP (8080) – Jenkins UI ke liye
- Custom TCP (9000) – Docker Host application ke liye

Key pair download karke rakho (PuTTY se connect karne ke liye).

### Step 6: Teeno Server Ko PuTTY Se Connect Karna
Har server ki public IP nikalo aur PuTTY se `.pem`/`.ppk` key se connect karo.

---

## Part 2: Jenkins Server Configure Karna

### Step 7: Java Install Karna
```bash
yum install java-11-openjdk -y
```

### Step 8: Jenkins Install Karna
```bash
wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
yum install jenkins -y
```

### Step 9: Jenkins Start Karna
```bash
systemctl start jenkins
systemctl enable jenkins
```

### Step 10: Git Package Install Karna
```bash
yum install git -y
```
(Baad mein GitHub URL integrate karte waqt issue na aaye isliye pehle hi install kar lo)

### Step 11: Jenkins Browser Mein Kholna
1. `http://<Jenkins-Public-IP>:8080` browser mein kholo
2. Initial admin password nikalo:
```bash
cat /var/lib/jenkins/secrets/initialAdminPassword
```
3. Password paste karo → Suggested plugins install karo → Admin user create karo

---

## Part 3: Ansible Server Configure Karna

### Step 12: Ansible Install Karna
```bash
yum install epel-release -y
yum install ansible -y
```

### Step 13: Docker Host Ko Ansible Inventory Mein Add Karna
```bash
vi /etc/ansible/hosts
```
Neeche add karo:
```ini
[docker-host]
<Docker-Host-Private-IP>
```

### Step 14: Docker Package Install Karna (Ansible Server Par)
```bash
yum install docker -y
systemctl start docker
systemctl enable docker
```
(Yahi par Dockerfile build hogi, isliye Docker chahiye)

### Step 15: Ansible → Docker Host Passwordless SSH Setup
```bash
passwd root                          # root password set karo
vi /etc/ssh/sshd_config              # PermitRootLogin yes karo
systemctl restart sshd

ssh-keygen                           # key generate karo (agar pehle se nahi hai)
ssh-copy-id -i root@<Docker-Host-Public-IP>
```
Note: Docker Host par bhi pehle root password set karke `PermitRootLogin yes` karna padega, tabhi yeh command chalega.

---

## Part 4: Docker Host Configure Karna

### Step 16: Docker Install Karna
```bash
yum install docker -y
systemctl start docker
systemctl enable docker
```

### Step 17: Root Login Enable Karna (Ansible se connect hone ke liye)
```bash
passwd root
vi /etc/ssh/sshd_config      # PermitRootLogin yes
systemctl restart sshd
```

---

## Part 5: Jenkins ↔ Ansible Passwordless Connection

### Step 18: Jenkins Server Par Root Setup
```bash
passwd root
vi /etc/ssh/sshd_config      # PermitRootLogin yes
systemctl restart sshd
```

### Step 19: Jenkins → Ansible SSH Key Copy Karna
```bash
ssh-keygen
ssh-copy-id -i root@<Ansible-Private-IP>
```

---

## Part 6: Jenkins Plugin & SSH Server Configuration

### Step 20: "Publish Over SSH" Plugin Install Karna
1. Jenkins Dashboard → **Manage Jenkins → Manage Plugins**
2. **Available** tab mein search karo: `Publish Over SSH`
3. Install karo, Jenkins restart karo

### Step 21: Jenkins Aur Ansible Dono Ko SSH Server Ke Roop Mein Add Karna
1. **Manage Jenkins → Configure System**
2. Sabse niche **SSH Servers** section mein jao
3. **Jenkins entry add karo:**
   - Name: `jenkins-server`
   - Hostname: Jenkins ka Private IP
   - Username: `root`
   - Advanced → Use password authentication → password daalo
4. **Ansible entry add karo:**
   - Name: `ansible-server`
   - Hostname: Ansible ka Private IP
   - Username: `root`
   - Advanced → password daalo
5. Har entry ko **Test Configuration** se check karo — "Success" aana chahiye
6. **Save**

---

## Part 7: GitHub Webhook Setup

### Step 22: Jenkins Mein GitHub Token Add Karna
Yeh Step 4 mein banaya gaya token yahan use hoga.

### Step 23: GitHub Repo Mein Webhook Add Karna
1. Repo → **Settings → Webhooks → Add webhook**
2. **Payload URL**: `http://<Jenkins-Public-IP>:8080/github-webhook/`
3. **Content type**: `application/json`
4. Secret (optional) mein token paste kar sakte ho
5. Pehle Jenkins configuration save karo, phir **Add webhook** click karo

---

## Part 8: Jenkins Job Banana

### Step 24: New Freestyle Job Create Karna
1. Jenkins Dashboard → **New Item**
2. Name: `cloudknowledge-job`
3. **Freestyle project** select karo → OK

### Step 25: Git Source Configure Karna
**Source Code Management → Git** select karo:
- Repository URL: `https://github.com/<your-username>/cloudknowledge.git`

### Step 26: Build Trigger Set Karna
**Build Triggers** section mein check karo: **GitHub hook trigger for GITScm polling**

### Step 27: Build Step 1 — Dockerfile Ko Ansible Par Bhejna
**Build → Add build step → Send files or execute commands over SSH**
- SSH Server: `jenkins-server`
- Exec command:
```bash
cd $WORKSPACE
rsync -avz Dockerfile root@<Ansible-Private-IP>:/opt/
```

### Step 28: Build Step 2 — Ansible Par Build, Tag, Push Commands
Ek aur **Send files or execute commands over SSH** add karo:
- SSH Server: `ansible-server`
- Exec command:
```bash
cd /opt

docker image build -t $JOB_NAME:v1.$BUILD_ID .

docker image tag $JOB_NAME:v1.$BUILD_ID sd171991/$JOB_NAME:v1.$BUILD_ID
docker image tag $JOB_NAME:v1.$BUILD_ID sd171991/$JOB_NAME:latest

docker image push sd171991/$JOB_NAME:v1.$BUILD_ID
docker image push sd171991/$JOB_NAME:latest

docker image rmi $JOB_NAME:v1.$BUILD_ID sd171991/$JOB_NAME:v1.$BUILD_ID sd171991/$JOB_NAME:latest
```
> `sd171991` ki jagah apna Docker Hub username daalo.

### Step 29: Ansible Server Par Docker Hub Login (Manual, Ek Baar)
Ansible server par PuTTY se jaake:
```bash
docker login
# Username/password daalo
```
⚠️ Yeh miss mat karna — bina login ke push fail ho jayega.

### Step 30: Ansible Server Par Playbook Banana
```bash
mkdir /source-code
vi /source-code/docker.yml
```
Content:
```yaml
- hosts: docker-host
  tasks:
    - name: Stop Container
      shell: docker container stop cloudknowledge-container
      ignore_errors: yes

    - name: Remove Container
      shell: docker container rm cloudknowledge-container
      ignore_errors: yes

    - name: Remove Docker Image
      shell: docker image rm sd171991/$JOB_NAME
      ignore_errors: yes

    - name: Create New Container
      shell: docker container run -itd --name cloudknowledge-container -p 9000:80 sd171991/$JOB_NAME
```
> `ignore_errors: yes` isliye taaki pehli baar jab container/image exist hi na ho tab bhi playbook fail na ho.

### Step 31: Post-Build Action — Playbook Run Karwana
**Post-build Actions → Send files or execute commands over SSH**
- SSH Server: `ansible-server`
- Exec command:
```bash
cd /source-code
ansible-playbook docker.yml
```

### Step 32: Job Save Karna
**Apply → Save**

---

## Part 9: Testing

### Step 33: Manual Build Run Karke Test Karna
1. Job page par **Build Now** click karo
2. Console Output check karo — build, tag, push, playbook sab steps pass hone chahiye
3. Docker Host par check karo:
```bash
docker container ls
docker image ls
```
4. Browser mein: `http://<Docker-Host-Public-IP>:9000` — website chalni chahiye

### Step 34: Auto-Trigger Test Karna (End-to-End)
1. GitHub repo mein Dockerfile edit karo (jaise ADD URL change karo)
2. Commit karo
3. Webhook automatically Jenkins job trigger karega
4. Job complete hone ke baad Docker Host par naya container automatically ban jayega
5. Browser refresh karke naya content check karo

---

## Quick Troubleshooting Table

| Problem | Reason | Fix |
|---|---|---|
| Docker push par auth error | Ansible server par Docker Hub login nahi hai | `docker login` manually karo (Step 29) |
| Playbook fail — "container already exists" | Purana container already chal raha hai | Playbook mein stop/remove tasks add karo (Step 30) |
| SSH Test Configuration fail | Root password/PermitRootLogin set nahi hai | Step 15-19 dobara check karo |
| Job trigger nahi ho raha commit ke baad | Webhook ya Build Trigger set nahi | Step 23 & 26 verify karo |
| rsync command fail | Ansible server ki IP galat ya SSH key missing | Step 19 dobara karo |

---

## Variables Cheat Sheet

| Variable | Meaning |
|---|---|
| `$JOB_NAME` | Jenkins job ka naam (e.g. `cloudknowledge-job`) |
| `$BUILD_ID` | Har build ke saath automatically badhne wala number (1, 2, 3...) |
| `$WORKSPACE` | Jenkins server par woh folder jahan job ka code aata hai |

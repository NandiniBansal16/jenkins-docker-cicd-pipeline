## Step 1: Set up Jenkins on EC2

- Launch a Ubuntu Server EC2 instance (`t2.micro` is fine)
- Security group: allow inbound **22** (SSH) and **8080** (Jenkins UI)
- Install Java, then Jenkins from the official apt repository
- Start the Jenkins service and unlock it via the web UI

![AWS](https://imgur.com/Hk28ffE.png)

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y fontconfig openjdk-21-jre-headless

sudo install -m 0755 -d /etc/apt/keyrings
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/" | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt update && sudo apt install -y jenkins
sudo systemctl start jenkins
sudo systemctl enable jenkins
```

Get the unlock password and finish the setup wizard in the browser at `http://<jenkins-public-ip>:8080`:

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

![Jenkins dashboard after setup](docs/images/03-jenkins-dashboard.png)

## Step 2: Configure Git and Maven on Jenkins

Install Git at the OS level, and Maven by downloading the binary directly (the apt repository version is usually outdated):

```bash
sudo apt install -y git

cd /opt
sudo wget https://dlcdn.apache.org/maven/maven-3/3.9.16/binaries/apache-maven-3.9.16-bin.tar.gz
sudo tar -xvzf apache-maven-3.9.16-bin.tar.gz
```

> `dlcdn.apache.org` only hosts the current release — if a version 404s, pull it from `archive.apache.org` instead.

Set environment variables in `~/.bash_profile`:

```bash
JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64
M2_HOME=/opt/apache-maven-3.9.16
M2=$M2_HOME/bin
PATH=$PATH:$JAVA_HOME/bin:$M2
export JAVA_HOME M2_HOME M2 PATH
```

In the Jenkins UI, install the **GitHub Integration** and **Maven Integration** plugins, then configure JDK and Maven paths under **Manage Jenkins → Tools**.

![Jenkins Global Tool Configuration](docs/images/04-global-tool-config.png)

## Step 3: Set up the Docker host

- Launch a second EC2 instance, same VPC/subnet as Jenkins
- Security group: allow inbound **22** and a custom TCP range (e.g. **8081–9000**) for app containers
- Install Docker from the official repository

```bash
sudo apt update
sudo apt install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker
```

Create a dedicated `dockeradmin` user that Jenkins will SSH in as, and add it to the `docker` group:

```bash
sudo adduser dockeradmin
sudo usermod -aG docker dockeradmin
```

> Group membership only takes effect on a **fresh login** — log `dockeradmin` out and back in (or open a brand-new session) before relying on passwordless `docker` access.

Enable password-based SSH for this internal connection:

```bash
sudo vi /etc/ssh/sshd_config
# set: PasswordAuthentication yes
sudo systemctl restart ssh
```

> Ubuntu cloud images ship a drop-in override (commonly `/etc/ssh/sshd_config.d/60-cloudimg-settings.conf`) that can silently force password auth back off. Always confirm with `sudo sshd -T | grep -i passwordauthentication` rather than trusting the file alone.

![Docker host EC2 instance](docs/images/05-docker-host-ec2.png)

![docker run hello-world output](docs/images/06-docker-hello-world.png)

## Step 4: Integrate Docker with Jenkins

Install the **Publish Over SSH** plugin in Jenkins, then register the Docker host under **Manage Jenkins → System → Publish over SSH**:

- Name: `dockerhost`
- Hostname: docker host's private IP
- Username: `dockeradmin`
- Password authentication, with the `dockeradmin` password
- Remote directory: `/opt/docker`

![Publish over SSH configuration](docs/images/07-publish-over-ssh-config.png)

## Step 5: Create the Jenkins job

Freestyle project, configured with:

**Source Code Management** — Git, pointing at the GitHub repo, `*/main` branch.

**Build Triggers** — Poll SCM, schedule `* * * * *` (checks every minute for new commits).

**Build step — Invoke top-level Maven targets:**
- Maven Version: `Maven3`
- Goals: `clean package`
- Advanced → POM: `hello-world/pom.xml` *(only needed if your project lives in a subfolder rather than the repo root)*

**Post-build action — Send build artifacts over SSH:**
- SSH Server: `dockerhost`
- Source files: `hello-world/webapp/target/*.war, hello-world/Dockerfile`
- Remove prefix: `hello-world`
- Remote directory: *(leave blank — set once at the server level above; setting it again here will double up the path)*
- Exec command:
```bash
  cd /opt/docker
  cp webapp/target/*.war . 2>/dev/null || true
  docker stop registerapp || true
  docker rm registerapp || true
  docker build -t regapp:v1 .
  docker run -d --name registerapp -p 8087:8080 regapp:v1
```

![Jenkins job build steps](docs/images/08-jenkins-build-steps.png)

![Jenkins post-build SSH config](docs/images/09-jenkins-postbuild-ssh.png)

## Step 6: Test the full pipeline

Push a change to GitHub:

```bash
git add .
git commit -m "Update registration form"
git push origin main
```

Within a minute, Jenkins polls, detects the change, and triggers automatically:

![Jenkins build console output, success]("https://github.com/user-attachments/assets/a0a786a7-e97c-4cf4-b639-34da0aef1fdf")

Once the build finishes, the new container is live:

![App running in browser]("https://github.com/user-attachments/assets/3f68e56e-c39a-42ff-96f2-73223e53a5ed")

## Project structure

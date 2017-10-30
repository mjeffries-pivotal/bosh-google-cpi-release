# Deploying Concourse on Google Compute Engine

This guide describes how to deploy [Concourse](http://concourse.ci/) on [Google Compute Engine](https://cloud.google.com/) using BOSH. You will deploy a BOSH director as part of these instructions.

## Prerequisites
* You must have the `terraform` CLI installed on your workstation. See [Download Terraform](https://www.terraform.io/downloads.html) for more details.  This lab was written using Terraform v0.10.7.
Once downloaded, install it by unzipping the file and installing the executable:

```
cd ~/Downloads
unzip ./terraform_0.10.7_darwin_amd64.zip
sudo cp ./terraform /usr/local/bin
terraform -v
```

* You must have the `gcloud` CLI installed on your workstation. See [cloud.google.com/sdk](https://cloud.google.com/sdk/).

* You must have the `git` CLI installed on your workstation. See [git Downloads](https://git-scm.com/downloads).

### Setup your workstation
1. Signup

    * [Sign up](https://cloud.google.com/compute/docs/signup) for Google Cloud Platform
    * Create a [new project](https://console.cloud.google.com/iam-admin/projects)
    * Enable the [GCE API](https://console.developers.google.com/apis/api/compute_component/overview) for your project
    * Enable the [IAM API](https://console.cloud.google.com/apis/api/iam.googleapis.com/overview) for your project
    * Enable the [Cloud Resource Manager API](https://console.cloud.google.com/apis/api/cloudresourcemanager.googleapis.com/overview)

2. Clone this repo:

  ```
  cd
  git clone https://github.com/mjeffries-pivotal/bosh-google-cpi-release
  ```

3. Set your project ID:

  ```
  export projectid=REPLACE_WITH_YOUR_PROJECT_ID
  ```

4. Export your preferred compute region and zone:

  ```
  export region=us-east1
  export zone=us-east1-c
  export zone2=us-east1-d
  ```

5. Configure `gcloud` with a user who is an owner of the project:

  ```
  gcloud auth login
  gcloud config set project ${projectid}
  gcloud config set compute/zone ${zone}
  gcloud config set compute/region ${region}
  ```

6. Create a service account and key:

  ```
  gcloud iam service-accounts create terraform-bosh
  gcloud iam service-accounts keys create /tmp/terraform-bosh.key.json \
      --iam-account terraform-bosh@${projectid}.iam.gserviceaccount.com
  ```

7. Grant the new service account editor access to your project:

  ```
  gcloud projects add-iam-policy-binding ${projectid} \
      --member serviceAccount:terraform-bosh@${projectid}.iam.gserviceaccount.com \
      --role roles/editor
  ```

8. Make your service account's key available in an environment variable to be used by `terraform`:

  ```
  export GOOGLE_CREDENTIALS=$(cat /tmp/terraform-bosh.key.json)
  ```

### Create required infrastructure with Terraform

1. From your terminal, change to the directory containing [main.tf](main.tf) and [concourse.tf](concourse.tf) from this repository.  If you've just installed terraform, you'll need to run the `terraform init` command.

  ```
  cd ~/bosh-google-cpi-release/docs/concourse/
  terraform init
  ```

2. Review the main.tf file.  You may need to update it to specify a different network IP range depending on what's already running in your GCP account.  The main.tf will use the following unless you change it:
* bosh subnet - ip_cidr_range = "10.0.10.0/24"

3. Review the concourse.tf file.  You may need to update it to specify a different network IP range depending on what's already running in your GCP account.  The concourse.tf will use the following unless you change it:
* concourse subnet 1 - ip_cidr_range = "10.0.20.0/22"
* concourse subnet 2 - ip_cidr_range = "10.0.40.0/22"

4. From the same directory, view the Terraform execution plan to see the resources that will be created:

  ```
  terraform plan -var projectid=${projectid} -var region=${region} -var zone-1=${zone} -var zone-2=${zone2}
  ```

5. Create the resources:

  ```
  terraform apply -var projectid=${projectid} -var region=${region} -var zone-1=${zone} -var zone-2=${zone2}
  ```

### Deploy a BOSH Director

#### Setup your bosh bastion VM

1. In your new project, open Cloud Shell (the small >_ prompt icon in the web console menu bar).

2. Using Cloud Shell, SSH to the bastion VM you created in the previous step. **All commands after this should be run from the VM**:

  ```
  gcloud compute ssh bosh-bastion-concourse
  ```

3. Configure `gcloud` to use the correct zone, region, and project:

  ```
  zone=$(curl -s -H "Metadata-Flavor: Google" http://metadata.google.internal/computeMetadata/v1/instance/zone)
  export zone=${zone##*/}
  export region=${zone%-*}
  gcloud config set compute/zone ${zone}
  gcloud config set compute/region ${region}
  export project_id=`curl -s -H "Metadata-Flavor: Google" http://metadata.google.internal/computeMetadata/v1/project/project-id`
  ```

4. Explicitly set your secondary zone:

  ```
  export zone2=us-east1-d
  ```

5. Create a **password-less** SSH key:

  ```
  ssh-keygen -t rsa -f ~/.ssh/bosh -C bosh
  ```

6. Run this `export` command to set the full path of the SSH private key you created earlier:

  ```
  export ssh_key_path=$HOME/.ssh/bosh
  ```

7. Navigate to your [project's web console](https://console.cloud.google.com/compute/metadata/sshKeys) and add the new SSH public key by pasting the contents of ~/.ssh/bosh.pub:

  ![](../img/add-ssh.png)

  > **Important:** The username field should auto-populate the value `bosh` after you paste the public key. If it does not, be sure there are no newlines or carriage returns being pasted; the value you paste should be a single line.

8. Install bosh2, and make sure it installed ok.

  ```
  sudo curl -o /usr/bin/bosh2 https://s3.amazonaws.com/bosh-cli-artifacts/bosh-cli-2.0.28-linux-amd64
  sudo chmod +x /usr/bin/bosh2
  bosh2 -v
  ```

9. Create and `cd` to a directory:

  ```
  mkdir google-bosh-director
  cd google-bosh-director
  ```

10. Use `vim` or `nano` to create a BOSH Director deployment manifest named `manifest.yml.erb` with the content below.  You may need to update it to specify a different network IP range depending on what's already running in your GCP account.  **The network values must match the ones you specified in the main.tf template earlier.** The manifest.yml.erb will use the following unless you change it:
* bosh vip subnet
  * range: 10.0.10.0/29
  * gateway: 10.0.10.1
  * static: [10.0.10.3-10.0.10.7]
* bosh private network static_ips: [10.0.10.6]
* bosh dns address: 10.0.10.6
* bosh blobstore address: 10.0.10.6
* bosh agent mbus: nats://nats:nats-password@10.0.10.6:4222
* bosh ssh_tunnel host: 10.0.10.6

  ```
  ---
  <%
  ['region', 'project_id', 'zone', 'ssh_key_path'].each do |val|
    if ENV[val].nil? || ENV[val].empty?
      raise "Missing environment variable: #{val}"
    end
  end

  region = ENV['region']
  project_id = ENV['project_id']
  zone = ENV['zone']
  ssh_key_path = ENV['ssh_key_path']
  %>
  name: bosh

  releases:
    - name: bosh
      url: https://bosh.io/d/github.com/cloudfoundry/bosh?v=260.1
      sha1: 7fb8e99e28b67df6604e97ef061c5425460518d3
    - name: bosh-google-cpi
      url: https://bosh.io/d/github.com/cloudfoundry-incubator/bosh-google-cpi-release?v=25.6.2
      sha1: b4865397d867655fdcc112bc5a7f9a5025cdf311

  resource_pools:
    - name: vms
      network: private
      stemcell:
        url: https://bosh.io/d/stemcells/bosh-google-kvm-ubuntu-trusty-go_agent?v=3312.12
        sha1: 3a2c407be6c1b3d04bb292ceb5007159100c85d7
      cloud_properties:
        zone: <%=zone %>
        machine_type: n1-standard-4
        root_disk_size_gb: 40
        root_disk_type: pd-standard
        service_scopes:
          - compute
          - devstorage.full_control

  disk_pools:
    - name: disks
      disk_size: 32_768
      cloud_properties:
        type: pd-standard

  networks:
    - name: vip
      type: vip
    - name: private
      type: manual
      subnets:
      - range: 10.0.10.0/29
        gateway: 10.0.10.1
        static: [10.0.10.3-10.0.10.7]
        cloud_properties:
          network_name: concourse
          subnetwork_name: bosh-concourse-<%=region %>
          ephemeral_external_ip: true
          tags:
            - bosh-internal

  jobs:
    - name: bosh
      instances: 1

      templates:
        - name: nats
          release: bosh
        - name: postgres
          release: bosh
        - name: powerdns
          release: bosh
        - name: blobstore
          release: bosh
        - name: director
          release: bosh
        - name: health_monitor
          release: bosh
        - name: google_cpi
          release: bosh-google-cpi

      resource_pool: vms
      persistent_disk_pool: disks

      networks:
        - name: private
          static_ips: [10.0.10.6]
          default:
            - dns
            - gateway

      properties:
        nats:
          address: 127.0.0.1
          user: nats
          password: nats-password

        postgres: &db
          listen_address: 127.0.0.1
          host: 127.0.0.1
          user: postgres
          password: postgres-password
          database: bosh
          adapter: postgres

        dns:
          address: 10.0.10.6
          domain_name: microbosh
          db: *db
          recursor: 169.254.169.254

        blobstore:
          address: 10.0.10.6
          port: 25250
          provider: dav
          director:
            user: director
            password: director-password
          agent:
            user: agent
            password: agent-password

        director:
          address: 127.0.0.1
          name: micro-google
          db: *db
          cpi_job: google_cpi
          user_management:
            provider: local
            local:
              users:
                - name: admin
                  password: admin
                - name: hm
                  password: hm-password
        hm:
          director_account:
            user: hm
            password: hm-password
          resurrector_enabled: true

        google: &google_properties
          project: <%=project_id %>

        agent:
          mbus: nats://nats:nats-password@10.0.10.6:4222
          ntp: *ntp
          blobstore:
             options:
               endpoint: http://10.0.0.6:25250
               user: agent
               password: agent-password

        ntp: &ntp
          - 169.254.169.254

  cloud_provider:
    template:
      name: google_cpi
      release: bosh-google-cpi

    ssh_tunnel:
      host: 10.0.10.6
      port: 22
      user: bosh
      private_key: <%=ssh_key_path %>

    mbus: https://mbus:mbus-password@10.0.0.6:6868

    properties:
      google: *google_properties
      agent: {mbus: "https://mbus:mbus-password@0.0.0.0:6868"}
      blobstore: {provider: local, path: /var/vcap/micro_bosh/data/cache}
      ntp: *ntp
  ```

11. Fill in the template values of the manifest with your environment variables:

  ```
  erb manifest.yml.erb > manifest.yml
  ```

12. Deploy the new manifest to create a BOSH Director. Note that this can take 15-20 minutes to complete. You may want to consider starting this command in a terminal multiplexer such as tmux or screen.

  ```
  bosh2 create-env manifest.yml
  ```

13. Fetch ca_cert.pem from the manifest:

  ```
  bosh2 interpolate manifest.yml --path /misc/ca_cert > ca_cert.pem
  ```

14. Target your BOSH environment and login:

  ```
  bosh2 alias-env micro-google --environment 10.0.0.6 --ca-cert ca_cert.pem
  bosh2 login -e micro-google
  ```

Your username is `admin` and password is `admin`.

### Deploy Concourse
Complete the following steps from your bastion VM.

1. Upload the required [Google BOSH Stemcell](http://bosh.io/docs/stemcell.html):

  ```
  bosh2 -e micro-google upload-stemcell https://bosh.io/d/stemcells/bosh-google-kvm-ubuntu-trusty-go_agent?v=3468.1
  ```

2. Upload the required [BOSH Releases](http://bosh.io/docs/release.html):

  ```
  bosh2 -e micro-google upload release https://bosh.io/d/github.com/concourse/concourse?v=3.6.0
  ```

3. Create a new directory, install git, and clone the repo.

```
cd
mkdir cpi
cd cpi
sudo apt-get install git
git clone https://github.com/mjeffries-pivotal/bosh-google-cpi-release
cd bosh-google-cpi-release/docs/concourse/
```

4. Review the [cloud-config.yml](cloud-config.yml) manifest file. You may need to update it to specify a different network IP range depending on what's already running in your GCP account.  **The values must match the ones you specified in the concourse.tf temlate earlier.** The cloud-config.yml will use the following unless you change it:
* subnets: az: z1
  * range: 10.0.20.0/24
  * gateway: 10.0.20.1
* subnets: az: z2
  * range: 10.0.40.0/24
  * gateway: 10.0.40.1

5. Review the [concourse.yml](concourse.yml) manifest file and set a few environment variables:

  ```
  export external_ip=`gcloud compute addresses describe concourse | grep ^address: | cut -f2 -d' '`
  export director_uuid=`bosh status --uuid 2>/dev/null`
  ```

5. Choose unique passwords for internal services and ATC and export them

   ```
   export common_password=
   export atc_password=
   ```

6. Note the value for "external_ip" - that's where concourse will be running

7. Remember the value for "atc_password" - that's the password you'll use to login to concourse.

8. (Optional) Enable https support for concourse atc (you'll need an SSL cert)

  In `concourse.yml` under the atc properties block fill in the following fields:
  ```
  tls_bind_port: 443
  tls_cert: << SSL Cert for HTTPS >>
  tls_key: << SSL Private Key >>
  ```

9. Upload the cloud config:

  ```
  bosh2 -e micro-google update cloud-config cloud-config.yml
  ```

10. Target the deployment file and deploy:

  ```
  bosh2 -e micro-google deployment concourse.yml
  bosh2 -e micro-google deploy
  ```

11.  After the job completes, you'll have your concourse environment available at http://EXTERNAL_IP.  Go ahead and browse to this URL from your workstation.

### Setup Concourse fly CLI on your workstation

Now you're done with the bosh bastion VM and google shell.  Go back to your workstation for the rest of the lab.

1. Download the Fly CLI.  Look for the download link in the lower right corner of the concourse app.  Then install fly as follows:

```
chmod +x ~/Downloads/fly
sudo cp ~/Downloads/fly /usr/local/bin
fly -v
```

2. Login to concourse using the Fly CLI and the concourse external IP from above.  The userid is "concourse", and the password is the value of "atc-password" from above.

```
fly -t gcp login -c http://EXTERNAL_IP
```

3. Create a simple pipeline to test out concourse.  Create a file called "hello.yml" with this content:

```
jobs:
- name: hello-world
  plan:
  - task: say-hello
    config:
      platform: linux
      image_resource:
        type: docker-image
        source: {repository: ubuntu}
      run:
        path: echo
        args: ["Hello, world!"]
```

4. Now send the pipeline to concourse.

```
fly -t gcp set-pipeline -p hello-world -c hello.yml
```

5. To see your pipeline in concourse, copy URL from the output of the command above to your browser.  Login using same userid/password you did for the fly CLI.

6. Now unpause the pipeline.

```
fly -t gcp unpause-pipeline -p hello-world
```

7. Click on hello-world, then kick off a build by clicking on the plus sign at the upper right corner of the page.  Look at the output of the build to see the greeting echoed out.

8. optionally - create DNS A record pointing to EXTERNAL_IP, then use the DNS name instead of the IP.

9. Concourse creates a single team called "main", with "concourse" user, when you first install concourse.  You can add additional teams and users as follows, supplying values for USERID and PASSWORD:

```
fly -t gcp set-team -n team1 --basic-auth-username USERID --basic-auth-password PASSWORD
fly -t gcp login -n team1
```

10. When you're done, you can destroy the pipeline:

```
fly -t gcp destroy-pipeline -p hello-world
```

11. Explore the [fly CLI](http://concourse.ci/fly-cli.html).  You will learn more about using Concourse tomorrow.

# using Terraform with gitlab to deploy multiple containers using a template

## TL;DR
You want to create a template to deploy multiple containers into EKS from a gitlab container repos using terraform and don't want to have access to the gitlab side automated without handling the tokens. then the terraform code you are looking for is here [tf_gitlab_k8](https://github.com/Adampeterjones/tf_gitlab_k8)

## background
after building an eks cluster and deploying all the components that go along with it (DNS, alb-ingress controllers etc) using terraform. I thought it would be good to be able to create a template to handle deployments in kubernetes for deployments with similar requirements.
Now I know there is already whole host of other tools in the market which could do this such as:
- kustomize - create a template with overlays to handle specifics
- argocd or flux - attach a project to the system and allow manifests to be handled in a gitops way

no problem with these tools but this seemed like an additional layer of complexity requiring managing,  we already use gitlab to build our infrastructure (using terraform) and our code and do the deployments so it seemed a sensible conclusion to follow the same path and embed the deployment with the existing process, plus we could use terraform to handle a number of components in one project (i.e. should we need an event queue that could be created as part of the same deployment).

One decision we made early in the deployment process was this:
as we were storing the code in gitlab and we were creating containers it made sense to store the containers in the gitlab project.  

Container repos at a project level can be enabled by going to:

 Settings -> General -> Visibility, project features, permissions

![settings](/blog/docs/assets/container_settings.PNG)


Then switch the toggle to enable the containers

![toggle](..%5Cassets%5CContainer_switch.PNG)

we could now store our images alongside our repos (how to build these in gitlab is a story for another day) and we could then also deploy them across our kubernetes environments. 

## Requirements
Ok so inevitably we had some things already in place as part of our wider implementation which you will need to replicate to use this.  

### Gitlab token
we already control our gitlab projects using terraform and as part of this we have already implemented a Deploy token,  the detail from gitlab how to configure this can be found [here](https://docs.gitlab.com/ee/security/token_overview.html).  my suggestion would be to create a project token (at the top of your project tree to make life easier) and then store the value on the project as a masked variable in the ci/cd settings (only maintainers should be able to see this token).

### AWS role and external id
To increase security we chose to do our deployments via a kubernetes cluster in our control,  the gitlab runner is hosted here and the deployment happens here.  This means we are able to assume a role in the account using an external id,  you don't have to follow this pattern;  feel free to use the AWS keys and secret values (Remember to keep them secure and don't expose them in the repos),  the options for your AWS connectivity via terraform are to be found [here](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)

### state file locations (S3)
remote state in this case was managed in S3 buckets with the dynamoDB and encryption enabled,  you will need to have an equivalent setup or alternatively you be using Terraform cloud to secure this access.

### gitlab-ci.yaml
We heavily template our ci pipelines (which is why i haven't included it here) and this would be well outside of the scope of this post.  There is nothing out of the ordinary for this pipeline though you will just need to create some tasks which do the following:
- reads the variables from settings -> CI/CD -> Variables (see the section on AWS role and external id above)
- read the environment variable and run the following:
  - terraform init -backend-config="../../config/$CI_ENVIRONMENT_NAME/backend.hcl"
  - terraform plan 
  - terraform apply (based on your trigger / gate for acceptance)

## The Terraform folder
so lets start with the basic bits:

### state.tf 
a basic file as the detail is environment specific (see backend.hcl later on),  this file only tells us that we should be using S3 to store our state files:

```
terraform {
  backend "s3" {
  }
}
```

### versions.tf
looks for an up to date version of terraform, the aws and the gitlab providers

```
terraform {
  required_version = ">= 1.0.0"
  required_providers {
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = ">= 2.9.0"
    }
    aws = {
      source  = "hashicorp/aws"
      version = ">= 3.30"
    }
    gitlab = {
      source  = "gitlabhq/gitlab"
      version = ">=3.14.0"
    }
  }
}
```

### provider.tf
a pretty straightforward file which contains 2 providers (AWS and Kubernetes).  the Gitlab one isn't specified, as the only thing it requires is the GITLAB_TOKEN environment variable to be set, the gitlab provider details can be found here https://registry.terraform.io/providers/gitlabhq/gitlab/latest/docs.
the AWS provider has been parameterised to accept a region (I'm using eu-west-2 London), and assume a role (see the section on AWS role and external id above).  Additionally, We like to use a set of default tags as well to identify that the items were created and managed by Terraform and create a link back to the project.

this looks like:

```
provider "aws" {
  region              = var.aws_region
  allowed_account_ids = [var.aws_account_id]
  assume_role {
    role_arn    = "arn:aws:iam::${var.aws_account_id}:role/${var.resource_management_iam_role}"
    external_id = var.external_id
  }
  default_tags {
    tags = {
      Project   = var.project_url
      Region    = var.aws_region
      ManagedBy = "Terraform"
    }
  }
}
```

the next main facet is the kubernetes provider.  this block relies on the eks-data.tf settings which dynamically source the kube-config (they in turn rely on the first aws provider block).

```
provider "kubernetes" {
  host                   = data.aws_eks_cluster.eks.endpoint
  cluster_ca_certificate = base64decode(data.aws_eks_cluster.eks.certificate_authority.0.data)
  token                  = data.aws_eks_cluster_auth.eks.token
  insecure               = false

}
```

### eks-data.tf
this file uses the aws provider to retrieve the kubeconfig settings by looking up the cluster.  I have chosen to name the cluster using the environment as a prefix to the name.  I then use the environment variables to make this a dynamic lookup; which returns the cluster details (host and cluster cert) and the authorisation details (token).

```
# THIS DATA SOURCE WILL RETURN, AMONG OTHER THINGS, YOUR ENDPOINT AND AND CA CERTIFICATE
data "aws_eks_cluster" "eks" {
  name = "${var.environment}-${var.eks_cluster_name}"
}

# THIS DATA SOURCE WILL RETURN, AMONG OTHER THINGS, YOUR API TOKEN
data "aws_eks_cluster_auth" "eks" {
  name = data.aws_eks_cluster.eks.name #--EKS Cluster Name
}

```

### Variables.tf
this declares contains all the variables (unsurprisingly),  i wont list the full contents here as it is all available in the repos
but the one that is worth noting is this section at the end of the file:

```
variable "manifest" {
  description = "manifest for deployment of containers"
  type        = map(any)
  default     = {}
}
```

this creates an empty map for the manifest later,  this means that if you don't create a manifest in the environment terraform.tfvars file then no container deployments will be created.

### gitlab-access.tf
finally an interesting terraform file I hear you say! yes this is where things actually start to happen.  this file does two things:
1. create a deployment token - this is put against a gitlab project,  you can identify the project by the id or the name.  the important thing here is that the deployment token is hierarchical,  so if you have a top level project underneath which all the containers will be built then this is a good place to put the token.  The image below represents this and also shows that the gitlab_token environment variable is allowing access at the company root:
![token creation](..%5Cassets%5Cproject%20structure.png)) 

I'm creating a username and generating a token that is limited to reading from the registry only.
```
resource "gitlab_deploy_token" "gitlab_container_read" {
  project  = "containers-project" 
  name     = "Gitlab container read token"
  username = "container-reader"

  scopes = ["read_registry"]
}
```
the details and spec of this are here: https://registry.terraform.io/providers/gitlabhq/gitlab/latest/docs/resources/deploy_token.

2. now we have a token we need to apply it to our eks cluster,  to do this we need to use a kubernetes_secret_v1 with a type of "kubernetes.io/dockerconfigjson".  the specifics of this secret type is documented here:
https://registry.terraform.io/providers/hashicorp/kubernetes/latest/docs/resources/secret_v1

the string needs to be jsonencoded to work properly, one more point of reference is of course that kubernetes secrets are only base64 encoded so control of who can see this secret should be tightly limited.

the code here is:
```
resource "kubernetes_secret_v1" "gitlab-token" {
  metadata {
    name      = "gitlab-cfg"
    namespace = "workflow"
  }

  type = "kubernetes.io/dockerconfigjson"

  data = {
    ".dockerconfigjson" = jsonencode({
      auths = {
        "registry.gitlab.com" = {
          "username" = gitlab_deploy_token.gitlab_container_read.username
          "password" = gitlab_deploy_token.gitlab_container_read.token
          "auth"     = base64encode("${gitlab_deploy_token.gitlab_container_read.username}:${gitlab_deploy_token.gitlab_container_read.token}, auth: base64encode(${gitlab_deploy_token.gitlab_container_read.username}:${gitlab_deploy_token.gitlab_container_read.token}")
        }
      }
    })
  }
}

```

### std-workers-deployment.tf
this is the highpoint of the folder,  whe have done all the hard work now we have a large file which is a template.  in the first couple of lines we create the deployment resource and loop based on the contents of the manifest file (remember we declared this in the variables.tf earlier).
```
resource "kubernetes_deployment" "standard_worker_deployment" {
  for_each = var.manifest
  metadata {
    name      = each.value.name
    namespace = "workflow"
    labels = {
      test = each.value.name
    }
  }

```
  the only things we are going to be setting in this template are the name and the image.  all the other settings are constant in this template, but they don't need to be feel free to use more variables to reach your goal.  
  A really important point here is that in the template we are specifying the secret to use with the image pull (i have chosen to put it into the spec but you could also embed in to the namespaces service account)
  ```
        spec {
        image_pull_secrets {
          name = kubernetes_secret_v1.gitlab-token.metadata[0].name #"gitlab-cfg"
        }
```
  
a couple of things worth noting here though are:
1. service account settings
  ```
          # service_account_name = "eks-lambda"
  ```

now I have left this commented out in this example but it is possible to create a service account which can have IAM roles assigned to it, in this comment I would be using a service account which allows my pods to execute lambda functions.  I may cover this in a separate post in the future.

2. container environment variables
the below code allows you to set some environment variables within the pod.  again this could be turned into a dynamic block if required or you can remove it you don't require it.

``` 
        container {
          env {
            name  = "TEST_SERVER"
            value = "testserver.test.svc.cluster.local:7233"

          }
```

## CONFIG
This folder is for the Environment specifics.  I have created two example sub-folders, one for dev and one for test. the contents are essentially the same with two files in each:

### backend.hcl
this file specifies the location of the state file and the role which is used to access it (it's a pretty straightforward file):
```
region="eu-west-2"
bucket="dev-tfstate"
key="dev-eks-kubernetes-gitlab.tfstate"
dynamodb_table="dev-tfstate"
encrypt="true"
role_arn="arn:aws:iam::123456789012:role/tfstate-mgnt-role-dev"
```

### terraform.tfvars
There are four distinct objects in this file:
1. aws_account_id   - this is as it implies (required in our case as we separate environments reside in their own accounts).
2. environment      - the environment name (this could be overridden by using the environment variable from the gitlab-ci.yaml)
3. eks_cluster_name - the name of the cluster you are deploying into 
4. manifest         - this is a map of the deployments to be carried out.  the reason for using a map (and not a count loop) is that each one will have a name in the map and therefore if the ordering is changed it will have no impact on the deployments.  within the map are the name and the image used in the template earlier.  

By having a manifest per environment we can handle the deployment to each version individually,  tags can control the management of the deployments as well,  in this case i have just used the :latest tags which would be fine in a dev environment but better tag control should be used in a higher environment.  
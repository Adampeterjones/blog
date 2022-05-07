# using Terraform with gitlab to deploy multiple containers using a template
after building an eks cluster and dploying all the components that go along with it (DNS, alb-ingress controllers etc) using terraform I thought it would be good to be able to create a template to handle deployments in kubernetes for deployments with similar requirements.
Now I know there is already whole host of other tools in the market which could do this such as:
- kustomize - create a template with overlays to handle specifics
- argocd or flux - attach a project to the system and allow manifests to be handled in a gitops way

no problem with these tools but this seemed like an additional layer of complexity requiring managing,  we already use gitlab to build our infrastructure (using terraform) and our code and do the deployments so it seemed a sensible conclusion to follow the same path and embed the deployment with the existing process, plus we could use terraform to handle a number of components in one project (i.e. should we need an eventqueue that could be created as part of the same deployment).

One decision we made early in the deployment process was this:
as we were storing the code in gitlab and we were creating containers it made sense to store the containers in the gitlab project.  

Container repos at a project level can be enabled by going to:

 Settings -> General -> Visibility, project features, permissions

![settings](images\container_settings.PNG)


Then switch the toggle to enable the containers

![toggle](images\container_switch.PNG)

we could now store our images alongside our repos (how to build these in gitlab is a story for another day) and we could then also deploy them across our kubernetes environments. 

## Requirements
Ok so inevitably we had some things already in place as part of our wider implementation which you will need to replicate to use this.  

### Gitlab token
we already control our gitlab projects using terraform and as part of this we have already implemented a Deploy token,  the detail from gitlab how to configure this can be found [here](https://docs.gitlab.com/ee/security/token_overview.html).  my suggestion would be to create a project token (at the top of your project tree to make life easier) and then store the value on the project as a masked variable in the cicd settings (only maintainers should be able to see this token).

### AWS role and external id
To increase security we chose to do our deployments via a kubernetes cluster in our control,  the gitlab runner is hosted here and the deployment happens here.  This means we are able to assume a role in the account using an external id,  you dont have to follow this pattern;  feel free to use the AWS keys and secret values (Remember to keep them secure and dont expose them in the repos),  the options for your AWS connectivity via terraform are to be found [here](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)

### state file locations (S3)

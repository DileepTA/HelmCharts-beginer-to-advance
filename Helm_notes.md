Helm Tutorial Notes — Concepts We Practiced
These notes cover the Helm concepts and troubleshooting we worked on with your Kubernetes project tech_advisor in namespace kcw. You can save them for interview preparation and revision.
1. What is Helm?
Helm is a package manager for Kubernetes. It helps you deploy, configure, upgrade, and roll back Kubernetes applications using reusable templates called charts.

Without Helm, you may need to manage multiple YAML files manually, such as Deployment, Service, and ServiceAccount manifests. Helm packages these configurations together.
Helm Chart

Templates + values.yaml + Chart.yaml


Helm Release

A deployed instance of the chart


Kubernetes Resources

Deployment, Pods, Services, etc.


2. Important Helm terminology
Concept	Meaning
Chart	Package containing Kubernetes templates and configuration
Release	An installed instance of a Helm chart
Revision	A numbered version of a release's recorded history
values.yaml	Default configuration values for the chart
Chart.yaml	Chart metadata, including name and version
Templates	YAML files containing reusable Helm expressions
Repository	A location from which Helm charts can be downloaded
Rollback	Restoring a previous release revision

Example from your project:
- Chart directory: tech_advisor/
- Release name: techrelease
- Namespace: kcw
- Chart name: tech-advisor
The chart name and release name are different things.
3. Understanding the Helm chart structure
Your chart can have the following structure:
tech_advisor/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    └── serviceaccount.yaml


- Chart.yaml — defines chart metadata.
- values.yaml — stores default settings, such as the container image and replica count.
- templates/deployment.yaml — generates a Kubernetes Deployment.
- templates/service.yaml — generates a Kubernetes Service.
- templates/serviceaccount.yaml — generates a ServiceAccount.
For example, a value in values.yaml:
replicaCount: 2


Can be referenced in a template:
spec:
  replicas: {{ .Values.replicaCount }}


This makes your chart reusable across development, testing, and production environments.
4. Installing a Helm chart
The command you used was:
helm install techrelease tech_advisor/ --namespace kcw


Explanation:
- helm install — installs a new release.
- techrelease — the release name.
- tech_advisor/ — the local chart directory.
- --namespace kcw — deploys the release into the kcw namespace.
To verify:
helm list -n kcw
kubectl get pods -n kcw
kubectl get svc -n kcw


Important: The namespace must exist unless you use --create-namespace.
5. Fixing the invalid resource name error
You initially received an error similar to:
Service "techrelease-tech_advisor" is invalid


Why did it happen?
Kubernetes resource names generally cannot contain underscores. The chart was generating names such as:
techrelease-tech_advisor


You corrected the chart naming to use hyphens:
techrelease-tech-advisor


For example, your Chart.yaml should use:
apiVersion: v2
name: tech-advisor
version: 0.1.0


Likewise, use a valid container name:
containers:
  - name: tech-advisor


Rule: Use lowercase letters, numbers, and hyphens in Kubernetes resource names. Container names have the stricter DNS label naming rules, so don't use dots in them.
6. Understanding the “cannot reuse a name” error
You received:
cannot reuse a name that is still in use


This means Helm already had a release registered under that name in the namespace.

You checked it using:
helm list -n kcw -a


Your environment showed two releases:
Release	Status
tech-adv-release	failed
techrelease	deployed

How to handle this error
Upgrade the existing release if you want to apply changes to it:
helm upgrade techrelease .\tech_advisor\ -n kcw


Uninstall and reinstall if you intentionally want to remove the existing release:
helm uninstall techrelease -n kcw
helm install techrelease .\tech_advisor\ -n kcw


Use a different release name if you want a separate installation:
helm install techrelease2 .\tech_advisor\ -n kcw


Be careful with uninstalling: it removes the release's managed resources, which may cause downtime or data loss depending on the application.
7. Understanding Helm revisions and history
Every successful installation or upgrade creates a new release revision.

Example:
helm install techrelease .\tech_advisor\ -n kcw
helm upgrade techrelease .\tech_advisor\ -n kcw
helm upgrade techrelease .\tech_advisor\ -n kcw


The history may look like this:
Revision	Status	Explanation
1	superseded	Older release revision replaced by a later one
2	superseded	Replaced by revision 3
3	deployed	Current successful revision

These are illustrative statuses; your actual history depends on which operations succeeded or failed.
Commands to remember
List releases:
helm list -n kcw


List releases in all statuses:
helm list -n kcw -a


View the revision history of a release:
helm history techrelease -n kcw


Inspect the current release:
helm status techrelease -n kcw


Why doesn't helm list show every revision?

Because helm list shows releases, not the full history of each release. Use helm history for all recorded revisions.
8. What does superseded mean?
Superseded means that a previous release revision has been replaced by a newer revision.

For example:
Revision 1 → superseded
Revision 2 → superseded
Revision 3 → deployed


It does not mean that revision 1 or 2 necessarily failed. It means that a newer revision replaced it.

Other important statuses include:
- deployed — the current successful release revision.
- failed — an installation or upgrade failed.
- pending-upgrade — an upgrade is in progress or may be stuck.
- uninstalled — the release has been uninstalled, if its history is retained.
9. Rolling back to an earlier revision
Suppose revision 3 has a problem and revision 2 was working correctly.

Run:
helm rollback techrelease 2 -n kcw


Verify the result:
helm history techrelease -n kcw
helm status techrelease -n kcw
kubectl get pods -n kcw


Important: A rollback creates another revision. It does not simply erase the newer revision from history.

For example, rolling back from revision 3 to revision 2 can create revision 4 containing the restored configuration.
10. Debugging your application using Kubernetes events
Your pod events included:
Liveness probe failed:
Get "http://10.244.2.2:80/":
connect: connection refused


This indicates that Kubernetes attempted to reach the application on port 80, but no process accepted the connection at that address and port.

Your Tech Stack Advisor application uses Gradio and is expected to listen on port 7860, provided the application configuration hasn't changed.
Why probes matter
- Liveness probe: Checks whether the container should be restarted.
- Readiness probe: Checks whether the container is ready to receive traffic.
- Startup probe: Gives a slow-starting application time to initialize before liveness checks begin.
If the probes target the wrong port, Kubernetes may repeatedly restart an otherwise correctly configured application.

Check the logs:
kubectl logs deployment/techrelease-tech-advisor -n kcw
kubectl logs deployment/techrelease-tech-advisor -n kcw --previous


Inspect the Deployment:
kubectl describe deployment techrelease-tech-advisor -n kcw


If Gradio listens on port 7860, configure the probes to use port 7860 and ensure the application binds to 0.0.0.0:7860.

Also verify the Service's targetPort matches the application's listening port.
11. Accessing the application with port forwarding
Port forwarding lets you access an application on your local machine without requiring an external IP address.

If the application listens on port 7860, run:
kubectl port-forward deployment/techrelease-tech-advisor 7860:7860 -n kcw


Then open:

http://localhost:7860

Keep the terminal running while you use the application.

If your application is still restarting due to failed probes, fix that issue first. Port forwarding does not repair application errors.

You can also forward the Service port:
kubectl port-forward service/techrelease-tech-advisor 8080:80 -n kcw


Then open http://localhost:8080. This works only if the Service routes to the correct application port.
12. Helm troubleshooting commands — quick reference
Task	Command
Install a chart	helm install techrelease .\tech_advisor\ -n kcw
Upgrade a release	helm upgrade techrelease .\tech_advisor\ -n kcw
List releases	helm list -n kcw
Show all release statuses	helm list -n kcw -a
View history	helm history techrelease -n kcw
Check release status	helm status techrelease -n kcw
Roll back	helm rollback techrelease 2 -n kcw
Uninstall	helm uninstall techrelease -n kcw
Validate chart	helm lint .\tech_advisor\
Preview generated YAML	helm template techrelease .\tech_advisor\ -n kcw
Check pods	kubectl get pods -n kcw
Check application logs	kubectl logs deployment/techrelease-tech-advisor -n kcw
Forward local port	kubectl port-forward deployment/techrelease-tech-advisor 7860:7860 -n kcw


13. Interview questions to revise
Q1. What is Helm in Kubernetes?

Helm is a Kubernetes package manager that uses charts to deploy and manage applications.

Q2. What is the difference between a chart and a release?

A chart is the application package; a release is an installed instance of that chart.

Q3. What is a Helm revision?

A revision is a numbered record of a release's installation, upgrade, or rollback history.

Q4. What is the difference between helm list and helm history?

helm list shows releases, while helm history shows the recorded revisions of a particular release.

Q5. What does superseded mean?

It means an older revision was replaced by a newer revision.

Q6. What is the difference between helm upgrade and helm rollback?

Upgrade applies updated chart configuration; rollback restores a previous revision's configuration.

Q7. Why did your pod keep restarting?

The liveness probe was checking port 80, but the application was expected to listen on port 7860. The probe and application port configuration needed to be aligned.

Q8. What is helm lint used for?

It checks a Helm chart for common structural and configuration problems before deployment.


Recommended workflow for your project
When changing your chart, use this sequence:
helm lint .\tech_advisor\
helm template techrelease .\tech_advisor\ -n kcw
helm upgrade --install techrelease .\tech_advisor\ -n kcw
kubectl rollout status deployment/techrelease-tech-advisor -n kcw
kubectl get pods,svc -n kcw
helm history techrelease -n kcw


This workflow validates the chart, previews its manifests, applies the release, checks deployment progress, and lets you inspect the revision history.

Main takeaway: Helm manages the lifecycle of your Kubernetes application; Kubernetes runs the resources created by Helm. Helm revisions help you track changes and roll back when necessary.

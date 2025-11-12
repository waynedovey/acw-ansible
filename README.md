To run the playbook:

ansible-playbook playbooks/ocp4_workload_app_connectivity_workshop.yml -e ACTION=create -e ocp4_workload_app_connectivity_workshop_acme_email="wdovey@gmail.com"


Example runs for Region options

Osaka / ap-northeast-3 (JP, default = true)

ansible-playbook playbooks/ocp4_workload_app_connectivity_workshop.yml \
  -e ACTION=create \
  -e ocp4_workload_app_connectivity_workshop_acme_email=wdovey@gmail.com \
  -e ocp4_workload_app_connectivity_workshop_dnspolicy_geo_code=JP \
  -e ocp4_workload_app_connectivity_workshop_dnspolicy_default=true \
  -e ocp4_workload_app_connectivity_workshop_dnspolicy_weight=120


Seoul / ap-northeast-2 (KR, default = false)

ansible-playbook playbooks/ocp4_workload_app_connectivity_workshop.yml \
  -e ACTION=create \
  -e ocp4_workload_app_connectivity_workshop_acme_email=wdovey@gmail.com \
  -e ocp4_workload_app_connectivity_workshop_dnspolicy_geo_code=KR \
  -e ocp4_workload_app_connectivity_workshop_dnspolicy_default=false \
  -e ocp4_workload_app_connectivity_workshop_dnspolicy_weight=120

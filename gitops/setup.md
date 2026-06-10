# Cluster Setup

## Install Web Termninal

## Add Group

User Management > Groups
cluster-admins
admin

## Add Service Account 


User Management > RoleBindings
Create
Name:  openshift-gitops-argocd-application-controller-cluster-admin-rb

Role Name: cluster-admin
Select Service Account

Project: openshift-gitops
Subject Name:  openshift-gitops-argocd-application-controller

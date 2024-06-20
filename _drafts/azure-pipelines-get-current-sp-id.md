---
title: "Azure Pipelines: How to get the current service principal id"
description: How to get the id of the service principal running the current task in Azure Pipelines
tags: [azure-devops, azure-pipelines]
---

This a quick post for sharing a solution to a problem I have faced recently: in an Azure pipeline running Azure CLI tasks, how can I get the current service principal id ?  
Well, it's a little less simple than it sounds, let's find out !

## An example
Let's imagine the pipeline creates a Key Vault and puts a secret in it: you need to grant the Secret Officer role to the service principal associated to your service connection.

## The solution

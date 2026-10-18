# Implement cloud infrastructure as code

**Optional task** · [Assignment](../../README.md)

## Goal

Turn your required cloud deployment design into understandable infrastructure configuration.

## Requirements

If you choose this task:
- Add Terraform under infra/ for a useful portion of your design. State what it provisions and which services are external inputs.
- Configure environment-specific names, network references and credentials safely; include example inputs without secrets.
- Include cost controls appropriate to the design, such as lifecycle policies or bounded autoscaling.
- Document validation, intended deployment and cleanup. A real cloud apply is not required.

## Suggested starting point

Start with storage or a small compute environment using the provider you know. A complete managed cluster, database estate and network is unnecessary.

Keep application connections configurable and explain how this provider-specific infrastructure preserves the portable interfaces from the cloud design. Show formatting and validation output; disclose checks blocked by provider access.

## How to demonstrate it

Provide a resource diagram/table, sample inputs, validation output and a brief description of the resources that would incur charges. Describe data retention and what cleanup would delete.

## Optional improvements

Implement AKS or another Kubernetes service, separate stable and Spot pools, scale batch capacity to zero, or add workload identity and private networking. Lifecycle tiering and Spot choices should agree with the required cost analysis.

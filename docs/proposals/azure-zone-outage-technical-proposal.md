# Add Azure Support to ZoneOutageScenarioPlugin

## Context

`ZoneOutageScenarioPlugin` currently supports:
- AWS via network isolation (`network_based_zone`)
- GCP via node lifecycle disruption (`node_based_zone`)

Azure currently falls through unsupported-cloud handling in plugin dispatch and exits non-zero.

This proposal adds Azure support using subnet-level NSG isolation, with rollback semantics similar to existing failure handling patterns.

---

## 1. Problem Statement

In `krkn/scenario_plugins/zone_outage/zone_outage_scenario_plugin.py`, plugin `run()` dispatch does not include Azure and returns:
`ZoneOutageScenarioPlugin Cloud type %s is not currently supported for zone outage scenarios`

Result:
- Azure zone outage scenario cannot execute
- `run()` returns `1`
- no fault injection occurs

---

## 2. Design Goals

1. Add Azure support without changing AWS and GCP behavior
2. Reuse existing Azure networking primitives in repo
3. Keep blast radius explicit and config-driven
4. Ensure deterministic restore of original network state
5. Add rollback callback for crash-safe recovery
6. Add tests for success and failure branches

---

## 3. Implementation Design

## 3.1 Dispatch Extension in Plugin

### File
`krkn/scenario_plugins/zone_outage/zone_outage_scenario_plugin.py`

### Change
Add Azure branch in `run()`:

- `elif cloud_type.lower() in ["azure", "az"]`
  - instantiate Azure cloud object
  - call new method `network_based_zone_azure(scenario_config)`
  - propagate non-zero return codes

Pseudo dispatch:

```python
if cloud_type == "aws":
    ...
elif cloud_type == "gcp":
    ...
elif cloud_type in ["azure", "az"]:
    self.cloud_object = Azure()
    result = self.network_based_zone_azure(scenario_config)
    if result != 0:
        return result
else:
    error unsupported
```

---

## 3.2 Azure Zone Outage Mechanism

Approach is network partition at subnet scope:

1. Resolve subnet and current NSG association
2. Create temporary deny-all NSG
3. Associate deny-all NSG to target subnet
4. Hold for `duration`
5. Restore original NSG association
6. Delete temporary NSG
7. Return status

This maps conceptually to AWS network ACL swap behavior.

### Why subnet-level NSG
- closest Azure equivalent to zone-wide network outage primitive
- independent from VMSS scaling/instance mapping bugs
- already aligns with existing Azure node block implementation pattern

---

## 3.3 Method Contract

### New method
`network_based_zone_azure(self, scenario_config: dict[str, Any]) -> int`

### Required config keys
- `resource_group`
- `vnet_name`
- `subnet_name`
- `duration`

### Optional config keys
- `location` (if NSG create requires explicit location)
- `timeout`
- `kube_check`
- `nsg_name_prefix` (for deterministic naming)

### Return semantics
- `0` success
- non-zero on any unrecoverable failure

---

## 3.4 Rollback Strategy

Before applying deny NSG:
- capture `original_nsg_id` (nullable)
- capture target identifiers (`resource_group`, `vnet_name`, `subnet_name`)
- register rollback callable with encoded payload

Rollback callable behavior:
1. if original NSG exists, reattach it to subnet
2. if no original NSG, clear association if supported by API
3. delete temporary chaos NSG if present
4. log outcomes and exceptions

This ensures crash/interruption recovery is possible even if process exits during injected state.

---

## 3.5 Idempotency and Error Handling

Error handling checkpoints:

1. subnet lookup failure
   - fail fast, return non-zero

2. NSG create failure
   - nothing applied yet, return non-zero

3. subnet update to deny NSG failure
   - attempt delete temporary NSG
   - return non-zero

4. restore failure
   - return non-zero with explicit restore failure log
   - keep rollback metadata for manual retry

5. temporary NSG delete failure
   - log warning
   - return non-zero or warning-only depending on maintainer preference
   - recommendation: non-zero for strict cleanup guarantees

---

## 3.6 Telemetry and Logging

Add structured log fields:
- `cloud_type`
- `resource_group`
- `vnet_name`
- `subnet_name`
- `original_nsg_id`
- `chaos_nsg_id`
- `duration`

Add event logs:
- injection start timestamp
- injection applied
- restore start/end
- cleanup result
- rollback invocation result

This improves operability and root-cause triage.

---

## 4. Code Reuse Plan

## 4.1 Existing Azure methods to reuse
From `az_node_scenarios.py`:
- create security group
- update subnet NSG association
- delete security group
- subnet/network client interactions

If existing methods are coupled to node-specific flow, add thin helper wrappers in Azure class rather than duplicating API logic in plugin.

## 4.2 Proposed helper additions (if needed)
- `get_subnet_nsg(resource_group, vnet_name, subnet_name) -> Optional[str]`
- `set_subnet_nsg(resource_group, vnet_name, subnet_name, nsg_id_or_none) -> None`

This keeps plugin logic concise and testable.

---

## 5. Test Plan

### File
`tests/test_zone_outage_scenario_plugin.py`

### Unit tests to add

1. `test_run_dispatches_azure_branch`
   - given `cloud_type: azure`
   - assert `network_based_zone_azure` invoked

2. `test_network_based_zone_azure_success`
   - mock subnet get, nsg create, subnet update, sleep, restore, delete
   - assert call order and return `0`

3. `test_network_based_zone_azure_create_nsg_failure`
   - assert return non-zero and no subnet update

4. `test_network_based_zone_azure_apply_failure_restores_or_cleans`
   - assert cleanup attempted and non-zero

5. `test_network_based_zone_azure_restore_failure`
   - assert non-zero and explicit error log

6. `test_network_based_zone_azure_registers_rollback`
   - assert rollback payload includes original_nsg_id and target subnet IDs

7. `test_rollback_azure_zone_outage_success`
   - assert subnet restored and temp NSG deleted

8. `test_rollback_azure_zone_outage_partial_failure`
   - assert best-effort continuation and logs

### Optional integration test
- gated AKS test fixture in CI matrix, if maintainers allow cloud integration tests

---

## 6. Backward Compatibility

- No behavior change for AWS/GCP paths
- Azure branch is additive
- config schema extension is backward compatible

---

## 7. Security and Permissions

Required Azure permissions for service principal or managed identity:
- read subnet and NSG resources
- create/update/delete NSG
- update subnet NSG association

Document minimum IAM roles and scope in docs to avoid runtime permission ambiguity.

---

## 8. Documentation Updates

### File
`docs/zone_outage_scenarios.md`

Add:
1. Azure section with required keys
2. warning about subnet blast radius
3. rollback/manual recovery steps
4. sample AKS scenario YAML
5. expected logs and success criteria

---

## 9. Acceptance Criteria

1. `cloud_type: azure` executes in zone outage plugin without unsupported-cloud error
2. subnet isolation is injected and removed for configured duration
3. original subnet NSG association is restored
4. rollback function can restore state after interruption
5. tests cover success and failure paths
6. docs include Azure config and recovery instructions
